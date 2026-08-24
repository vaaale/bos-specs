# Feature Specification: Repo-Context Conflict Escalation

**Feature Branch**: `035-spec-promote-conflict`

**Created**: 2026-08-24

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "The automatic/deterministic conflict-resolution steps in the promote/pull pipeline (merge, rebase, etc.) often fail, and when they do, no agent is triggered — the operation dead-ends with a static 'resolve manually' error. When deterministic resolution fails, an agent must be triggered to resolve the conflict. The escalation mechanism must be designed so that any dependency (e.g. worktree access) is CONFIGURED AND PROVIDED UPON EXECUTION of the DevOps agent: if the conflict is in the main BOS source tree, the environment is configured for that; if it is in user-apps, the environment is configured for that; if it is in user-specs, the environment is configured for that; etc. Scope: solve ALL outstanding conflict-escalation gaps in this space — no follow-ups, no deferrals."

## Background

The Supervisor's promote pipeline and the git sync/pull/push surfaces operate on **multiple git repos**, and they currently handle conflicts in two inconsistent ways:

**Path A — the escalation mechanism that works (BOS source repo only).**
`src/lib/gitops/reconcile.ts` contains a shared 5-step reconciliation pipeline: rollback tag → remote sync (optional) → merge strategy → scripted rebase fallback → **escalate to the DevOps Agent**. The escalation creates a persisted, resumable Assistant conversation scoped to the DevOps Agent, kicks off a run, waits (default 25 min), and surfaces the conversation id to the caller. `VersionControls.tsx` already renders the `state: "escalated"` + `devopsConversationId` state. **But the agent's working context is hard-coded to the BOS source repo's feature-branch worktree** (via the `featureBranchForDelegate` field → `dev_delegate`). It does not generalize to other repos.

**Path B — dead-ends (everywhere else).** These conflict paths re-implement a simplified inline rebase and, on conflict, stop with a static error and **no agent**:

| Call site | Repo | Current behavior on conflict |
|---|---|---|
| `src/lib/specs/promote.ts` → `promoteFeature` | user-specs store (`data/specs/`) | `git merge --no-edit <base>`; on conflict → `merge --abort`, returns `{ kind: "conflict", files }`. **The reported bug.** |
| `src/app/api/git-remotes/route.ts` `case "fetch"` (the "Pull" action) | any managed repo (source or VFS mount) | inline `rebaseOntoRemote`; on conflict → `{ rebaseConflict: true, message: "…resolve manually, or force-push…" }` |
| `src/app/api/git-remotes/route.ts` `case "push"` (recovery on non-FF rejection) | any managed repo | inline `rebaseOntoRemote`; on conflict → same static dead-end |
| User-apps repo (app promote / user-apps git ops; partly in the Supervisor process `tools/supervisor/supervisor.mjs`) | user-apps repo | [architect to confirm exact call site(s)] — same dead-end class |

**Repro (the reported bug):** branch `bos/034-event-notification-system` fails to promote with:
```
promote blocked — user-specs: branch bos/034-event-notification-system conflicts with master:
CONFLICT (add/add): Merge conflict in core-platform/034-event-notification-system/test-results.md
```
No agent is triggered; the user must resolve it manually via CLI.

**The core design principle (directed):** the escalation must be **repo-context-parameterized**. The working context — repo identity, worktree path, and the means by which the agent reads/writes within that repo — is a *parameter of the escalation*, configured at execution time to match whichever repo the conflict was detected in. There is no per-repo special-casing of the *access mechanism*; the context object is what varies. The BOS source path (Path A) is the first existing instance of this general pattern; this feature generalizes it.

## User Scenarios & Testing

### User Story 1 — The general escalation mechanism works, proven by the reported bug (Priority: P1)

The reconciliation pipeline's escalation step configures the DevOps agent's working environment for whichever repo the conflict was detected in, and the agent can then read/write within that repo's working tree and resolve the conflict. This is proven end-to-end by the reported case: a user-specs promote conflict (the `034-event-notification-system` repro) no longer dead-ends — it escalates, the agent's environment is configured for the user-specs worktree, and the agent either resolves the conflict (promote then completes: main fast-forwarded, worktree pruned) or times out/fails with the rollback tag for manual recovery.

**Why this priority**: This is the architectural core and the reported bug. It is the MVP that delivers the most value: it directly fixes the repro *and* provides the substrate every other call site hangs off. Nothing else is useful without it.

**Independent Test**: Recreate the repro — two feature branches that both add a same-path file (e.g. `test-results.md`) with different content to the user-specs store; promote one while the other's content is on main. Verify: (a) the escalate state appears in the UI within the same request cycle, (b) a DevOps Agent conversation is created, visible in the Assistant app, and titled after the conflict, (c) the agent's working context targets the user-specs worktree (not the source repo), (d) the agent resolves the conflict → promote completes, **or** times out/fails → promote reports failure with the rollback tag and main is left untouched.

**Acceptance Scenarios**:

1. **Given** a user-specs feature branch whose changes conflict with main, **When** the user clicks Promote, **Then** the Supervisor's promote response includes `devopsConversationId` and the UI transitions to the escalated state within the same request cycle (no separate poll needed to discover the escalation).

2. **Given** the DevOps Agent is resolving the user-specs conflict, **When** its run starts, **Then** its working environment is configured for the user-specs worktree (at `data/specs/.worktrees/<encoded-branch>/`) — it can read the conflicting files and write the merge resolution in that repo, not in the BOS source tree.

3. **Given** the agent resolves the conflict successfully, **When** the run completes, **Then** the promote completes: main is fast-forwarded to the reconciled branch, the worktree is pruned, and the response reports success.

4. **Given** the agent times out (default 25 min) or errors, **When** the run reaches a terminal state, **Then** the promote reports failure with the rollback tag and the conflicting files, and user-specs `main` is never left in a conflicted state.

---

### User Story 2 — Every other dead-end conflict path routes through the mechanism (Priority: P1)

Every deterministic-conflict-resolution path that currently dead-ends is converted to route through the generalized pipeline, with the working context configured for the repo the operation is acting on:

- **Pull from git** (`/api/git-remotes` `case "fetch"`): a diverged/conflicted pull no longer returns a static "resolve manually" error — it routes through the pipeline (strategy + rebase fallback + escalation) with the repo's working context.
- **Push recovery** (`/api/git-remotes` `case "push"`): a non-fast-forward rejection that recovers into a conflicted rebase no longer dead-ends — it routes through the pipeline with the repo's working context.
- **User-apps repo** operations: a conflict in the user-apps repo routes through the pipeline with the user-apps working context.
- **Any other managed git repo** (e.g. VFS-mounted repos) that the architect confirms has a dead-end conflict path: same treatment.

**Why this priority**: The user directed that ALL outstanding conflict-escalation gaps in this space be solved now — no follow-ups, no deferrals. These are the remaining gaps beyond the reported repro.

**Independent Test**: For each path, trigger a resolvable conflict (e.g. a same-path, different-content add/add) and verify it escalates (a DevOps conversation is created, the working context targets the correct repo) instead of returning a static "resolve manually" / `rebaseConflict` error.

**Acceptance Scenarios**:

1. **Given** a "Pull" on a managed repo where local and remote have diverged and the rebase conflicts, **When** the pull is processed, **Then** it escalates to the DevOps Agent with that repo's working context (not a static `rebaseConflict` error).

2. **Given** a "Push" that is rejected as non-fast-forward and whose recovery rebase conflicts, **When** the push is processed, **Then** it escalates to the DevOps Agent with that repo's working context.

3. **Given** a conflict in the user-apps repo, **When** the user-apps operation is processed, **Then** it escalates to the DevOps Agent with the user-apps working context.

4. **Given** any git conflict path in the system, **When** deterministic resolution fails, **Then** it routes through the pipeline and escalates — **no** git conflict path in the system returns a static "resolve manually" error with no agent.

---

### User Story 3 — The UI surfaces the escalated state uniformly (Priority: P2)

For a conflict escalated from **any** repo (not just the source repo), the relevant UI surface shows the live escalation: the "handed to the DevOps Agent" indicator, a link that opens the live conversation in the Assistant app, the conflicting file list, and the rollback tag. The surfaces in scope: `VersionControls.tsx` (topbar promote), `VersionsTab.tsx` (Settings → Versions), `ConflictResolutionDialog.tsx`, and `GitRemotesTab.tsx` (the Pull/Push surfaces). The user does not need to know to go look in the Assistant app — the surface they were already looking at links to the conversation.

**Why this priority**: The mechanism (US1/US2) is functionally complete without this, but the user can't see or watch the agent, and the existing source-repo path already sets the bar (VersionControls already handles `devopsConversationId`) — every other surface should match it.

**Independent Test**: For each surface, trigger an escalated conflict for its repo and verify the surface shows: (a) the amber "handed to the DevOps Agent" indicator, (b) a clickable link opening the conversation, (c) the conflicting file list, (d) the rollback tag.

**Acceptance Scenarios**:

1. **Given** a promote is escalated (agent working), **When** `VersionControls` / `VersionsTab` / `ConflictResolutionDialog` is rendered, **Then** it shows the "handed to the DevOps Agent" indicator, a link to the live conversation, the file list, and the rollback tag.

2. **Given** a Pull/Push on a managed repo is escalated, **When** `GitRemotesTab` is rendered, **Then** it shows the same escalated state (agent link + file list + rollback tag) instead of the static "resolve manually, or force-push" message.

3. **Given** the agent's run has completed (success or failure), **When** the user refreshes the surface, **Then** it shows the terminal state (resolved / timed out / error) and the rollback tag for manual recovery if needed.

---

### User Story 4 — Regression: the source-repo escalate path is unchanged (Priority: P3)

The existing BOS source-repo promote escalation must behave identically. This feature generalizes the mechanism that the source path already uses; the source path becomes the first *instance* of the general pattern but must not regress. The `featureBranchForDelegate` field remains the working-context value for the source case.

**Why this priority**: Regression protection for the one path that already works.

**Independent Test**: Trigger a source-repo promote conflict as before this feature. Verify the escalation is unchanged: same conversation creation, same `devopsConversationId` in the response, same UI state, same agent working context (source feature-branch worktree).

**Acceptance Scenarios**:

1. **Given** a source-repo promote conflict, **When** the promote is escalated, **Then** the behavior is identical to before this feature (conversation created, `devopsConversationId` in response, UI escalated state, agent targets the source feature-branch worktree).

2. **Given** `reconcile()` called with the source-repo working context, **When** it escalates, **Then** the escalation task text still tells the agent the worktree IS the Supervisor-tracked feature branch and `dev_delegate` targets it directly (existing behavior, unchanged).

---

### Edge Cases

- **Concurrent operations on the same repo**: two escalations racing against the same repo. The existing `inFlightEscalations` map in `reconcile.ts` (keyed by `repoPath`) prevents a second escalation while one is in flight — this MUST hold for user-specs, user-apps, and VFS-mount repo paths too, not just the source repo.
- **Agent's resolution doesn't merge cleanly / is wrong**: the agent's resolution is a valid commit; if it's wrong, the user uses the rollback tag to restore pre-reconciliation state and retry.
- **Binary-file conflict**: the agent may not resolve it intelligently. Fallback is the same as today: rollback tag + manual intervention.
- **No actual divergence (already up to date)**: the merge is a no-op; `reconcile()` already handles this (`hasStagedChanges` check).
- **Worktree not materialized**: `promoteFeature` already calls `ensureWorktree`; the working context must reference the materialized worktree path.
- **Agent has no suitable access for a given repo type** (e.g. a repo whose worktree the context can't grant write access to): the escalation must fail loudly with a clear error and the rollback tag, not silently pretend to have resolved it.
- **User-apps path lives partly in the Supervisor process** (`tools/supervisor/supervisor.mjs`, a separate Node process): the escalation there must reach the same generalized mechanism (the `/api/gitfs/reconcile` job endpoint or an equivalent) so the working context is configured consistently.

## Requirements

### Functional Requirements

**The general mechanism**

- **FR-001**: The reconciliation pipeline's escalation step MUST configure the DevOps agent's working environment for the specific repo where the conflict was detected, **at the time the agent run is launched**. The working context — repo identity, worktree path, and the means by which the agent reads/writes within that repo — MUST be a parameter of the escalation, not a hard-coded assumption about the BOS source tree.

- **FR-002**: The mechanism MUST work uniformly for every git repo the system manages — the BOS source repo, the user-specs store, the user-apps repo, and VFS-mounted repos. The per-repo variation MUST be carried entirely in the working context; there MUST be no per-repo special-casing of the *access mechanism*.

- **FR-003**: Given the working context, the DevOps agent MUST be able to read and write files within the conflicting repo's working tree and complete the merge (resolve the conflict and commit) without any CLI intervention by the user.

**Call-site conversions (all in scope — no deferrals)**

- **FR-004**: `promoteFeature` (`src/lib/specs/promote.ts`) MUST route its base-into-branch merge conflict through the pipeline with the **user-specs** working context, instead of `merge --abort` + a static `{ kind: "conflict" }` result. (The reported bug.)

- **FR-005**: The "Pull" action (`src/app/api/git-remotes/route.ts` `case "fetch"`) MUST route a diverged/conflicted pull through the pipeline (strategy + rebase fallback + escalation) with the **target repo's** working context, instead of the inline `rebaseOntoRemote` + static `rebaseConflict` error.

- **FR-006**: The push-recovery path (`src/app/api/git-remotes/route.ts` `case "push"`, on non-fast-forward rejection) MUST route a conflicted recovery rebase through the pipeline with the **target repo's** working context, instead of the inline `rebaseOntoRemote` + static error.

- **FR-007**: Conflicts in the **user-apps** repo (app promote / any user-apps git operation, including those originating in the Supervisor process) MUST route through the pipeline with the **user-apps** working context. [architect to confirm the exact call site(s) — the Supervisor-process side in particular.]

- **FR-008** (completeness invariant): Every deterministic-conflict-resolution path in the system MUST end by routing through the pipeline (which escalates when deterministic steps fail). There MUST be **no** git conflict path that dead-ends with a static "resolve manually" / `rebaseConflict` error and no agent. The `design` step MUST sweep the source to enumerate the complete set of such paths; all of them are in scope for this feature.

**Invariants, response shape, and UI**

- **FR-009**: For any repo with a linear-main invariant (user-specs `main` fast-forward-only; the source repo's base branch), that invariant MUST be preserved: the conflict surfaces on the feature/working branch, `main` is never left in a conflicted state, and a rollback tag is always created before any merge/rebase attempt.

- **FR-010**: The response returned to the caller MUST include `devopsConversationId` (and the state needed to render escalation) when a conflict is escalated — the Supervisor promote response, the git-remotes fetch response, and the git-remotes push response — matching the shape the source-repo path already uses.

- **FR-011**: The UI surfaces (`VersionControls.tsx`, `VersionsTab.tsx`, `ConflictResolutionDialog.tsx`, `GitRemotesTab.tsx`) MUST display the escalated state — "handed to the DevOps Agent" indicator, a link to the live conversation, the conflicting file list, and the rollback tag — for a conflict escalated from **any** repo, not only the source repo.

- **FR-012**: The operation response MUST distinguish "escalated, agent still working" (non-terminal, in progress) from "escalated, agent failed/timed out" (terminal, rollback tag available) so the UI shows the correct state.

- **FR-013** (regression): The existing BOS source-repo promote escalation MUST behave identically after this change. The source path is the first instance of the general mechanism and MUST NOT regress.

### Key Entities

- **Working context**: The per-repo bundle the escalation provides to the DevOps agent at execution time — repo identity (source / user-specs / user-apps / VFS mount / generic), the absolute worktree path, the base/branch being reconciled, and the means by which the agent reads/writes within that repo. The single parameter that makes the mechanism repo-agnostic.
- **User-specs / user-apps / managed-repo conflict**: A merge conflict in a managed git store, detected during promote/pull/push. Key attributes: conflicting file paths, the feature/working branch, the base ref, the rollback tag, and the repo it belongs to.
- **DevOps Agent conversation**: A persisted, resumable Assistant conversation scoped to the DevOps Agent, created by the pipeline's escalation step. Key attributes: conversation id, title (titled after the conflict), the working context it was launched with, and (source case only) `activeFeatureBranch` for `dev_delegate`.

## Success Criteria

### Measurable Outcomes

- **SC-001**: The reported repro (user-specs promote conflict on `034-event-notification-system`) escalates to the DevOps Agent within the same request cycle, the agent's working context targets the user-specs worktree, and the conflict is resolved (or the promote times out/fails cleanly with the rollback tag) — with **no** manual git CLI.

- **SC-002**: Zero managed-repo conflict paths dead-end with a static "resolve manually" / `rebaseConflict` error. Every known dead-end (user-specs promote, Pull, push-recovery, user-apps) and any additional one found by the design source-sweep escalates instead.

- **SC-003**: For a text-file conflict in any managed repo, the user can resolve it end-to-end (operation → escalate → agent resolves → operation completes) without opening a terminal.

- **SC-004**: The source-repo promote escalation is unchanged (no regression) — same conversation, same response shape, same UI state, same agent working context as before this feature.

- **SC-005**: For a conflict escalated from any repo, the relevant UI surface shows the escalated state (agent link, conflicting file list, rollback tag) and the user can open the live conversation from that surface.

## Assumptions

- The **DevOps Agent** is the correct agent for conflicts in every repo type (source, user-specs, user-apps, VFS mounts) — not a separate per-repo agent.
- The 5-step pipeline structure (rollback tag → sync → strategy → rebase fallback → escalate) is sufficient; this feature **generalizes the escalation's working context**, it does not add new pipeline steps.
- The **exact tooling** that realizes the working-context file access (e.g. a repo-path-scoped agent tool, an extension to `run_command`'s sandbox, or a VFS-backed tool) is a **design decision** the `architect` makes against real source — but it MUST satisfy the generality and no-special-casing constraints in FR-002. This is the one genuinely-open design question; its *shape* (per-repo, parameterized, configured at execution) is fixed by FR-001/FR-002.
- **All in scope, no follow-ups**: the completeness invariant (FR-008) means any dead-end conflict path the architect discovers in source during `design` is covered by this feature, not deferred.
- The user-apps promote path partly lives in the Supervisor process (`tools/supervisor/supervisor.mjs`); the architect MUST confirm its exact conflict call site(s) and how the escalation there reaches the generalized mechanism (likely the `/api/gitfs/reconcile` job endpoint or an equivalent).
- The `inFlightEscalations` guard, the rollback-tag step, and the linear-main invariant already exist in `reconcile.ts`; this feature inherits them rather than reimplementing them.
