# Feature Specification: Git Conflict Resolution System

**Feature Branch**: `035-spec-promote-conflict`

**Created**: 2026-08-24

**Status**: Draft

**App Target**: bos-core  *(the feature is a cross-cutting system capability — core plumbing in `src/lib/` + API routes + a new interactive resolution surface. The surface's exact form factor (dedicated window vs desktop panel vs embedded Assistant view) is a design decision, captured in Open Questions; the spec target remains bos-core.)*

**Input**: User description: "This is the full spec for a git conflict resolution system — a core feature of BOS. The automatic/deterministic conflict-resolution steps in the promote/pull/push pipeline (merge, rebase, etc.) often fail, and when they do the operation dead-ends with a static 'resolve manually' error and no agent. When deterministic resolution fails, an agent must be triggered to resolve the conflict. The escalation must be repo-context-parameterized: any dependency (worktree access, etc.) is configured and provided upon execution of the agent — if the conflict is in the main BOS source tree the environment is configured for that; if in user-apps, configured for that; if in user-specs, configured for that; etc. A conflict-resolution UI must launch automatically when a conflict is detected, because resolving a conflict requires decisions (accept theirs / ours, or manual fragment-by-fragment merge), and the agent must be able to communicate with the user when such decisions must be taken. Scope: all of it, in this one spec — no follow-ups, no deferrals."

## Background

The BOS system operates on **multiple git repos** — the BOS source repo, the user-specs store (`data/specs/`), the user-apps repo, and VFS-mounted repos. Today these handle conflicts in two inconsistent ways:

**Path A — an escalation mechanism that works, but is source-repo-only and watch-only.**
`src/lib/gitops/reconcile.ts` contains a shared 5-step reconciliation pipeline: rollback tag → remote sync (optional) → merge strategy → scripted rebase fallback → **escalate to the DevOps Agent**. The escalation creates a persisted, resumable Assistant conversation scoped to the DevOps Agent, launches a run, waits (default 25 min), and surfaces the conversation id. `VersionControls.tsx` renders the `state: "escalated"` + `devopsConversationId` state. Two limitations: (1) the agent's working context is **hard-coded to the BOS source repo's feature-branch worktree** (via `featureBranchForDelegate` → `dev_delegate`), so it does not generalize to other repos; (2) it is **autonomous and watch-only** — the agent either resolves or fails, and the user can only open the Assistant app to peek. There is no surface for the agent to request a decision from the user.

**Path B — dead-ends (everywhere else).** These conflict paths re-implement a simplified inline rebase and, on conflict, stop with a static error and **no agent**:

| Call site | Repo | Current behavior on conflict |
|---|---|---|
| `src/lib/specs/promote.ts` → `promoteFeature` | user-specs store | `git merge --no-edit <base>`; on conflict → `merge --abort`, returns `{ kind: "conflict", files }`. **The reported bug.** |
| `src/app/api/git-remotes/route.ts` `case "fetch"` (the "Pull" action) | any managed repo | inline `rebaseOntoRemote`; on conflict → `{ rebaseConflict: true, message: "…resolve manually, or force-push…" }` |
| `src/app/api/git-remotes/route.ts` `case "push"` (recovery on non-FF rejection) | any managed repo | inline `rebaseOntoRemote`; on conflict → same static dead-end |
| User-apps repo (app promote / user-apps git ops; partly in the Supervisor process `tools/supervisor/supervisor.mjs`) | user-apps repo | [architect to pin exact call site(s)] — same dead-end class |

**Repro (the reported bug):** branch `bos/034-event-notification-system` fails to promote with:
```
promote blocked — user-specs: branch bos/034-event-notification-system conflicts with master:
CONFLICT (add/add): Merge conflict in core-platform/034-event-notification-system/test-results.md
```
No agent is triggered; the user must resolve it manually via CLI.

## Design principles

- **PR-1 — Repo-context parameterization (directed).** The escalation's working context — repo identity, worktree path, and the means by which the agent reads/writes within that repo — is a **parameter of the escalation, configured at execution time** to match whichever repo the conflict was detected in. There is **no per-repo special-casing of the access mechanism**; the context object is what varies. The BOS source path is the first existing instance of this general pattern; this feature generalizes it to all managed repos.
- **PR-2 — Agent decides when it can, asks when it must.** The agent attempts autonomous resolution first (clear/trivial conflicts, unambiguous add/add, etc.). Only when it genuinely cannot decide does it issue a **typed decision request** to the user and wait for an answer. The user is never forced to hand-resolve a conflict the agent could resolve, and never blocked on a conflict the agent can decide autonomously.
- **PR-3 — No dead-ends, no deferrals.** Every deterministic-conflict path in the system ends by routing through the pipeline, which escalates when deterministic steps fail. There is **no** git conflict path that dead-ends with a static "resolve manually" error and no agent. All outstanding gaps in this space are in scope for this one feature.
- **PR-4 — The user always has the wheel.** Every resolution is recoverable: a rollback tag is always created, the user can always abandon and restore the pre-reconciliation state, and base/main is never left in a conflicted state.

## System overview (the six layers)

1. **Detection** — every conflict-capable operation (promote, pull, push-recovery, user-apps ops, any VFS mount) reports a conflict through a single canonical entry point. That entry point creates a **resolution session** (layer 2) and is the one trigger for both the agent run (layer 3) and the auto-launching UI (layer 4).
2. **Resolution session** — a first-class, **persisted, resumable** object that is the source of truth the UI renders and the system resumes: the working context (layer 3), a **conflict snapshot** (conflicting files, and per file/hunk the ours/theirs/base content), the rollback tag, the linked DevOps Agent conversation id, a **status state machine** (`working → awaiting-user → resolved | failed | timed-out`), and a **decision timeline** (each agent decision-request and user answer). Survives a process restart mid-resolution.
3. **Agent** — the **conflict-resolution agent** (user-configurable in Settings → Build Studio; default the DevOps Agent, FR-025), launched with the session's working context. It reads the three-way versions of each conflicting file, attempts autonomous resolution, and **writes hunk-level merges** back. On ambiguity it issues a typed decision request (the session → `awaiting-user`), receives the user's answer through the UI, and continues.
4. **Interaction / UI** — a conflict-resolution surface that **launches automatically when a conflict is detected** (not merely a passive state shown in an existing dialog). It renders the session, provides the 3-way diff / content comparison, the **agent↔user message channel**, per-file/per-hunk **decision controls**, manual editing, live status, and rollback/abandon. Resolving a conflict here completes the underlying operation.
5. **Safety** — base/main never left conflicted; rollback tag always created; one in-flight escalation per repo (the existing `inFlightEscalations` guard, extended to all repo paths); loud failure if the agent has no working context it can actually write to; no silent success on an unresolvable (e.g. binary) conflict.
6. **Cross-repo scope** — source, user-specs, user-apps, and VFS-mounted repos, all through the same session/agent/UI machinery, with the working context as the only per-repo variable.
7. **Configuration** — the conflict-resolution agent is a user-configurable setting in Settings → Build Studio, alongside the existing BS chat agent dropdown.

**Settings → Build Studio (addition to existing tab)**

The existing tab (`src/components/apps/settings/BuildStudioTab.tsx`) has one field: **Agent** (which sub-agent powers the Build Studio chat, stored as `build-studio.agent`). This feature adds a second field, **Conflict resolution agent** (stored as `build-studio.conflictAgent`, default `"devops"`), using the same `<select>` + Save button pattern. The pipeline reads `conflictAgent` at escalation time (FR-025) — the same "read on each operation" model as the existing `agent` field, so no reload is needed to pick up a change. The dropdown is populated from `GET /api/subagents` (same source as the existing field). No new UI mockup is required for this — it is a direct, in-tab extension of an existing, already-shipped pattern.

## User Scenarios & Testing

### User Story 1 — Core engine: a conflict becomes a resolution session and the agent resolves it (Priority: P1)

A conflict is detected anywhere (proven by the reported case: a user-specs promote conflict). It is routed through the pipeline; a resolution session is created; the agent is launched with the **user-specs** working context (not the source tree). The agent reads the three-way versions and resolves conflicts it can decide autonomously. When it resolves everything, the session goes `resolved` and the underlying operation (promote) completes: main fast-forwarded, worktree pruned. When deterministic steps + autonomous resolution can't fully resolve it, the agent issues decision requests (US2). When it fails or times out, the session goes `failed`/`timed-out` with the rollback tag, and main is never left conflicted.

**Why this priority**: This is the architectural core and the reported bug. It is the MVP: it directly fixes the repro *and* provides the substrate (session + working context) every other layer and call site hangs off.

**Independent Test**: Recreate the repro — two feature branches both add a same-path file (e.g. `test-results.md`) with different content to the user-specs store; promote one while the other's content is on main. Verify: (a) a resolution session is created and its state is queryable, (b) the agent's working context targets the user-specs worktree (not the source repo), (c) for an unambiguous conflict the agent resolves it autonomously → promote completes, **or** for an ambiguous one the session transitions to `awaiting-user`, **or** on failure/timeout the session reports the rollback tag and main is untouched.

**Acceptance Scenarios**:

1. **Given** a user-specs feature branch whose changes conflict with main, **When** the user clicks Promote, **Then** a resolution session is created and the promote response carries the session id (and the agent conversation id), so the escalated state is discoverable within the same request cycle.

2. **Given** a resolution session for a user-specs conflict, **When** the agent run starts, **Then** its working environment is configured for the user-specs worktree (`data/specs/.worktrees/<encoded-branch>/`) — it reads the conflicting files and writes the merge resolution in *that* repo, not the BOS source tree.

3. **Given** a conflict the agent can decide autonomously, **When** the agent resolves it, **Then** the session reaches `resolved` and the underlying promote completes (main fast-forwarded, worktree pruned).

4. **Given** the agent times out (default 25 min) or errors, **When** the run reaches a terminal state, **Then** the session is `failed`/`timed-out`, the promote reports failure with the rollback tag and the conflicting files, and user-specs `main` is never in a conflicted state.

---

### User Story 2 — The interactive UI: launch automatically, collaborate with the agent (Priority: P1)

When a conflict is detected, the conflict-resolution UI **launches automatically** (the user does not have to go look for it). It shows the session: the conflicting files, a 3-way view (ours / theirs / base) per file/hunk for code and a content comparison for text, and the agent's live status. The agent works; when it reaches a decision it cannot make autonomously, it **asks the user through the UI** (e.g. "add/add conflict on `test-results.md` — keep the feature branch version, the main version, or merge manually?") and the session goes `awaiting-user`. The user answers using the decision controls (accept theirs / accept ours / keep both / edit the hunk manually / accept the agent's suggestion), and the agent continues. The user can also proactively give direction, manually edit any hunk, and either let the agent finish or resolve the remaining hunks themselves. When everything is resolved, the UI reflects `resolved` and the underlying operation completes. The user can at any point abandon and roll back to the pre-reconciliation state.

**Why this priority**: This is the directed interactive requirement — the agent must make decisions and communicate with the user. Without it, hard/ambiguous conflicts still can't be resolved (the agent would just fail). It is inseparable from the system's value.

**Independent Test**: Trigger a genuinely ambiguous user-specs conflict (two branches adding the same file with incompatible content). Verify: (a) the UI auto-launches (or is unmissably surfaced) within seconds of detection, (b) it shows the 3-way content for the conflict, (c) the agent's status shows working, then `awaiting-user` with the agent's specific question, (d) the user picks "accept theirs" (or edits a hunk), (e) the agent continues and the session reaches `resolved`, (f) the underlying promote completes.

**Acceptance Scenarios**:

1. **Given** a conflict is detected, **When** the resolution session is created, **Then** the conflict-resolution UI launches automatically (or is unmissably surfaced) without the user taking any action to find it.

2. **Given** a code conflict, **When** the user opens the affected file in the UI, **Then** they see a 3-way view (ours / theirs / base) at hunk granularity, with the conflict markers.

3. **Given** the agent cannot decide autonomously, **When** it issues a decision request, **Then** the session shows `awaiting-user`, the agent's question is visible, and the decision controls are enabled.

4. **Given** the session is `awaiting-user`, **When** the user selects a decision (theirs / ours / keep both / edit hunk / accept suggestion), **Then** the answer is recorded in the decision timeline, the session returns to `working`, and the agent continues.

5. **Given** the session is `resolved`, **When** the user confirms (or the agent completes), **Then** the underlying operation completes and the UI shows success (or the rollback tag if the user chose to abandon instead).

6. **Given** the user clicks Abandon/Roll back at any point, **When** they confirm, **Then** the working tree is restored to the pre-reconciliation state via the rollback tag and the session is closed as `failed`(abandoned).

---

### User Story 3 — Completeness: every dead-end path is converted (Priority: P1)

Every deterministic-conflict-resolution path that currently dead-ends is converted to route through the pipeline (which creates a session and escalates), with the working context configured for the repo the operation is acting on:

- **Pull from git** (`/api/git-remotes` `case "fetch"`): a diverged/conflicted pull no longer returns a static `rebaseConflict` error.
- **Push recovery** (`/api/git-remotes` `case "push"`): a non-FF rejection whose recovery rebase conflicts no longer dead-ends.
- **User-apps repo** operations: a conflict routes through the pipeline with the user-apps working context.
- **Any other managed git repo** (e.g. VFS-mounted repos) the architect confirms has a dead-end path: same treatment.

**Why this priority**: The user directed that ALL outstanding conflict-escalation gaps be solved now — no follow-ups, no deferrals. PR-3 is a hard requirement, not an aspiration.

**Independent Test**: For each path, trigger a resolvable conflict (e.g. same-path, different-content add/add) and verify it creates a session and escalates (agent + UI) instead of returning a static error.

**Acceptance Scenarios**:

1. **Given** a "Pull" on a managed repo where local and remote diverge and the rebase conflicts, **When** the pull is processed, **Then** a session is created and it escalates with that repo's working context (not a static `rebaseConflict` error).

2. **Given** a "Push" rejected as non-FF whose recovery rebase conflicts, **When** the push is processed, **Then** a session is created and it escalates with that repo's working context.

3. **Given** a conflict in the user-apps repo, **When** the operation is processed, **Then** a session is created and it escalates with the user-apps working context.

4. **Given** any git conflict path in the system, **When** deterministic resolution fails, **Then** a session is created and the pipeline escalates — **no** path returns a static "resolve manually" / `rebaseConflict` error with no agent.

---

### User Story 4 — The state is surfaced uniformly and the source path doesn't regress (Priority: P2)

For a conflict escalated from **any** repo, the relevant UI surface shows the live session: the "handed to the DevOps Agent" indicator, a link into the resolution UI / live conversation, the conflicting file list, and the rollback tag. The surfaces in scope: `VersionControls.tsx` (topbar promote), `VersionsTab.tsx` (Settings → Versions), `ConflictResolutionDialog.tsx`, and `GitRemotesTab.tsx` (Pull/Push). The existing source-repo promote escalation must behave identically (it is the first instance of the general mechanism and must not regress).

**Why this priority**: The engine (US1–3) is functionally complete without this, but the user must be able to see/watch the session from wherever they triggered the operation, and the one path that already works must not break.

**Independent Test**: For each surface, trigger an escalated conflict for its repo and verify it shows the session state (agent link, file list, rollback tag). Separately, trigger a source-repo promote conflict and verify behavior is unchanged.

**Acceptance Scenarios**:

1. **Given** a promote is escalated (agent working / awaiting-user), **When** `VersionControls` / `VersionsTab` / `ConflictResolutionDialog` renders, **Then** it shows the session state (agent indicator, link into the resolution UI, file list, rollback tag).

2. **Given** a Pull/Push on a managed repo is escalated, **When** `GitRemotesTab` renders, **Then** it shows the session state instead of the static "resolve manually, or force-push" message.

3. **Given** a source-repo promote conflict, **When** escalated, **Then** the behavior is identical to before this feature (same conversation, same response shape, same UI state, same agent working context).

---

### User Story 5 — Safety and resumability (Priority: P2)

The resolution is always safe and always resumable. A rollback tag is created before any merge/rebase; base/main is never left conflicted; only one escalation runs per repo at a time; if the agent's working context cannot actually be written to, the operation fails loudly (with the rollback tag), never silently pretends success; and an unresolvable conflict (e.g. binary) is surfaced, not silently "resolved." A resolution session that is interrupted (e.g. BOS restarts) can be **resumed** — reopening it restores the working context, conflict snapshot, status, and decision timeline, and the agent/user can continue where they left off.

**Why this priority**: Safety is non-negotiable for a feature that writes to the repos that constitute the running system, and resumability is what makes the interactive (multi-turn, possibly minutes-long) loop robust.

**Independent Test**: (a) Trigger a conflict, note the rollback tag, abandon → verify `git log` on main shows it was never conflicted and the worktree matches the tag. (b) Start a resolution, kill/restart the process mid-session, reopen → verify the session's state and the agent's in-flight work are restored. (c) Trigger a binary-file conflict → verify it's surfaced as unresolvable-by-agent, not silently committed.

**Acceptance Scenarios**:

1. **Given** a resolution is in progress, **When** the user abandons it, **Then** the working tree is restored via the rollback tag, main is never conflicted, and the session is closed.

2. **Given** two operations target the same repo concurrently, **When** both would escalate, **Then** only one escalation runs (the existing per-repo in-flight guard), and the second is re-pointed to the existing session.

3. **Given** the agent's working context cannot be written to (no access for that repo type), **When** the agent is launched, **Then** the operation fails loudly with a clear error + rollback tag — it does not silently claim success.

4. **Given** a binary-file conflict, **When** the agent reaches it, **Then** it is surfaced as requiring manual handling (rollback tag provided), not silently resolved.

5. **Given** a resolution session is interrupted by a process restart, **When** the user reopens it, **Then** the working context, conflict snapshot, status, and decision timeline are restored and the resolution can continue.

---

### Edge Cases

- **Concurrent operations on the same repo**: the existing `inFlightEscalations` map (keyed by `repoPath`) must prevent a second escalation while one is in flight — for user-specs, user-apps, and VFS-mount paths too, not just the source repo.
- **Agent's resolution is wrong**: it's a valid commit; the user uses the rollback tag to restore pre-reconciliation state and retry.
- **Binary-file conflict**: surfaced as not agent-resolvable; rollback tag + manual intervention.
- **No actual divergence (already up to date)**: the merge is a no-op; `reconcile()` already handles this (`hasStagedChanges`).
- **Worktree not materialized**: `promoteFeature` already calls `ensureWorktree`; the working context references the materialized worktree path.
- **No suitable working context for a repo type**: loud failure + rollback tag, never silent success (US5).
- **User-apps path partly lives in the Supervisor process** (`tools/supervisor/supervisor.mjs`): its escalation must reach the same generalized mechanism (the `/api/gitfs/reconcile` job endpoint or equivalent) so the working context is configured consistently.
- **User is idle / away when `awaiting-user`**: the session stays `awaiting-user` **indefinitely** (D3) — not timed out — until the user returns or explicitly abandons. The 25-min timeout applies to the agent's *working* phases only, never to a parked `awaiting-user` state.
- **BOS restart / browser refresh mid-resolution**: a `working` session whose agent run died is re-launched on boot (FR-024); an `awaiting-user` session is restored and waits for the user. The conflict pane re-queries the session store on load, so a browser refresh restores it without re-emitting the event.

## Requirements

### Functional Requirements

**Detection & session (layers 1–2)**

- **FR-001**: Every conflict-capable operation (promote, pull, push-recovery, user-apps ops, VFS-mounted repos) MUST report a detected conflict through a single canonical entry point that creates a resolution session — so the agent run and the auto-launching UI are triggered consistently regardless of which surface detected the conflict.

- **FR-002**: A **resolution session** MUST be a first-class, persisted, resumable object containing: the working context (FR-003), a conflict snapshot (conflicting files, and per file/hunk the ours/theirs/base content), the rollback tag, the linked DevOps Agent conversation id, a status state machine (`working → awaiting-user → resolved | failed | timed-out`), and a decision timeline. The session MUST survive a process restart and be resumable.

**Working context & agent (layer 3)**

- **FR-003**: The pipeline's escalation MUST configure the DevOps agent's working environment — repo identity, worktree path, and the means to read/write within that repo — **as a parameter of the escalation, at execution time**, matching the repo the conflict was detected in. No per-repo special-casing of the access mechanism.

- **FR-004**: The mechanism MUST work uniformly for every managed repo (source, user-specs, user-apps, VFS mounts); per-repo variation is carried entirely in the working context.

- **FR-005**: Given its working context, the agent MUST be able to **read the three-way versions** (ours/theirs/base) of each conflicting file and **write hunk-level merges** back to that repo, completing the merge without CLI intervention.

- **FR-006**: The agent MUST attempt **autonomous resolution first** and only issue a typed decision request (accept-theirs / accept-ours / keep-both / manual-per-hunk) when it cannot decide; on a decision request the session MUST transition to `awaiting-user`, and on the user's answer it MUST continue.

**Interaction / UI (layer 4)**

- **FR-007**: A conflict-resolution UI MUST **launch automatically when a conflict is detected** (the user takes no action to find it). The mechanism: the pipeline's escalation step MUST emit a `com.bos.gitops.conflict.escalated` event (via the 034 event system) carrying the session id, repo identity, and `devopsConversationId`. Build Studio MUST declare a UI handler for this event type in its `AppManifest`; the event system opens or focuses BS and routes it to the conflict-resolution pane.
- **FR-007a**: The conflict-resolution pane MUST live in **Build Studio** as a custom pane in BS's main view (per D1). BS's existing chat MUST connect to the DevOps Agent conversation for the session — the agent↔user channel is the existing Assistant conversation, not a new mechanism.
- **FR-008**: The UI MUST render the session: conflicting files, a 3-way (ours/theirs/base) view at hunk granularity for code and a content comparison for text, and the agent's live status.
- **FR-009**: The UI MUST provide an **agent↔user message channel**: the agent's decision questions are visible, the user can answer them and proactively give direction, and the session reflects each exchange in the decision timeline.
- **FR-010**: The UI MUST provide per-file/per-hunk **decision controls**: accept theirs, accept ours, keep both, edit the hunk manually, and accept the agent's suggested resolution.
- **FR-011**: The UI MUST show the session status (`working` / `awaiting-you` / `resolved` / `failed` / `timed-out`) and a rollback/abandon affordance; resolving the session MUST complete the underlying operation, and abandoning MUST restore the pre-reconciliation state via the rollback tag.

**Call-site conversions (layer 6 — all in scope, no deferrals)**

- **FR-012**: `promoteFeature` (`src/lib/specs/promote.ts`) MUST route its base-into-branch conflict through the pipeline with the **user-specs** working context, instead of `merge --abort` + a static `{ kind: "conflict" }`. (The reported bug.)
- **FR-013**: The "Pull" action (`src/app/api/git-remotes/route.ts` `case "fetch"`) MUST route a conflicted pull through the pipeline with the **target repo's** working context, instead of inline `rebaseOntoRemote` + static `rebaseConflict`.
- **FR-014**: The push-recovery path (`src/app/api/git-remotes/route.ts` `case "push"`, on non-FF rejection) MUST route a conflicted recovery rebase through the pipeline with the **target repo's** working context.
- **FR-015**: Conflicts in the **user-apps** repo (app promote / any user-apps git operation, including Supervisor-process-originated) MUST route through the pipeline with the **user-apps** working context. [architect to pin exact call site(s).]
- **FR-016** (completeness invariant): Every deterministic-conflict path MUST end by routing through the pipeline (which escalates). **No** git conflict path MAY dead-end with a static "resolve manually" / `rebaseConflict` error and no agent. The `design` step MUST sweep the source to enumerate the complete set of such paths; all are in scope.

**Invariants, response shape, safety (layer 5)**

- **FR-017**: For any repo with a linear-main invariant (user-specs `main` fast-forward-only; the source base branch), the invariant MUST hold: the conflict surfaces on the working/feature branch, main is never left conflicted, and a rollback tag is always created before any merge/rebase.
- **FR-018**: The response returned to the caller MUST include the session id (and the agent conversation id) when a conflict is escalated — the Supervisor promote response, the git-remotes fetch response, and the git-remotes push response — so the UI can link into the session.
- **FR-019**: The UI surfaces (`VersionControls.tsx`, `VersionsTab.tsx`, `ConflictResolutionDialog.tsx`, `GitRemotesTab.tsx`) MUST show the session state (agent indicator, link into the resolution UI, conflicting file list, rollback tag) for a conflict escalated from **any** repo.
- **FR-020**: The operation/session response MUST distinguish `awaiting-user` (non-terminal, parked for the user) from `failed`/`timed-out` (terminal, rollback tag available).
- **FR-021**: If the agent's working context cannot actually be written to, the operation MUST fail loudly (clear error + rollback tag); it MUST NOT silently claim success.
- **FR-022**: An unresolvable conflict (e.g. binary) MUST be surfaced as requiring manual handling (rollback tag provided), NOT silently committed.
- **FR-023** (regression): The existing BOS source-repo promote escalation MUST behave identically after this change.
- **FR-025** (configurable conflict agent): The agent id used for conflict resolution MUST be **configurable in Settings → Build Studio** as a second dropdown alongside the existing "Agent" field (which controls the BS chat agent). The setting is stored in the `build-studio` config namespace as `conflictAgent` (default: `"devops"`). The pipeline's escalation step MUST read this value at execution time and use it as the `agentId` for the DevOps Agent conversation it creates — replacing the hard-coded `DEVOPS_AGENT_ID = "devops"` constant in `reconcile.ts`. The dropdown lists all registered sub-agents (same source as the existing BS agent dropdown: `GET /api/subagents`). The user MUST be able to select any agent that has git tool access; agents without git capability will simply fail at the escalation (surfaced as a `failed` session state with a clear error message naming the agent and the missing capability). The setting is read on each escalation — no restart or reload of BS needed to take effect.
- **FR-024** (restart recovery): On BOS restart, the system MUST auto-detect persisted resolution sessions and recover them: a session in `working` whose agent run is dead MUST be re-launched (boot-time sweep); a session in `awaiting-user` MUST be restored without re-launching the agent (it resumes when the user answers). The `com.bos.gitops.conflict.escalated` event MUST be re-emitted on boot for any session still in a non-terminal state, so BS re-launches with the conflict pane (reusing the event system's existing `redispatchPendingOnBoot` mechanism). A browser refresh of an already-running BOS session MUST restore the conflict pane from the persisted session (no event re-emit needed — the pane re-queries the session store on load).

### Key Entities

- **Resolution session**: The first-class, persisted, resumable source of truth for one conflict resolution. Key attributes: id, working context, conflict snapshot (per file/hunk ours/theirs/base), rollback tag, linked agent conversation id, status (`working`/`awaiting-user`/`resolved`/`failed`/`timed-out`), decision timeline.
- **Working context**: The per-repo bundle the escalation provides to the agent at execution time — repo identity (source / user-specs / user-apps / VFS mount / generic), absolute worktree path, the base/branch being reconciled, and the means to read/write within that repo. The single parameter that makes the mechanism repo-agnostic (PR-1).
- **Decision request**: A typed question the agent issues to the user for one file/hunk — options among accept-theirs / accept-ours / keep-both / manual-per-hunk (+ the agent's suggested resolution). Answered by the user through the UI; recorded in the decision timeline.
- **Conflict snapshot**: The captured state of the conflict at detection time — the conflicting file list and, per file/hunk, the ours/theirs/base content — so the 3-way view renders and the session resumes without re-deriving it.
- **Conflict-resolution agent conversation**: The persisted, resumable Assistant conversation scoped to the user-configured conflict-resolution agent (default: DevOps Agent), created by the pipeline's escalation step and linked to the session. (Source case: also carries `activeFeatureBranch` for `dev_delegate`.)

## Success Criteria

### Measurable Outcomes

- **SC-001**: The reported repro (user-specs promote conflict on `034-event-notification-system`) creates a session, the agent's working context targets the user-specs worktree, and the conflict is resolved (autonomously, or via a user decision in the UI) — or the promote fails/times out cleanly with the rollback tag — with **no** manual git CLI.

- **SC-002**: Zero managed-repo conflict paths dead-end with a static "resolve manually" / `rebaseConflict` error. Every known dead-end (user-specs promote, Pull, push-recovery, user-apps) plus any additional one found by the design source-sweep escalates instead.

- **SC-003**: For a **hard** text/code conflict in any managed repo (e.g. incompatible add/add), a user can resolve it end-to-end in the auto-launched UI (see 3-way view → answer the agent's decision / edit a hunk → resolved → operation completes) without opening a terminal.

- **SC-004**: The source-repo promote escalation is unchanged (no regression) — same conversation, response shape, UI state, and agent working context as before.

- **SC-005**: The conflict-resolution UI auto-launches (or is unmissably surfaced) within seconds of conflict detection, for a conflict in any repo.

- **SC-006**: A resolution session interrupted by a process restart can be resumed — reopening restores the working context, conflict snapshot, status, and decision timeline, and the resolution continues.

## Assumptions

- A **single, user-configurable conflict-resolution agent** (default: the DevOps Agent) is the resolver for every repo type (source, user-specs, user-apps, VFS mounts) — not separate per-repo agents (FR-025). The setting is one global value, not per-repo.
- The 5-step pipeline structure (rollback tag → sync → strategy → rebase fallback → escalate) is sufficient; this feature **generalizes the escalation's working context**, adds the **resolution session** and the **agent↔user decision loop**, and converts the dead-end call sites — it does not invent a new pipeline shape.
- **Autonomous-first** (PR-2) is confirmed (D2): the agent decides when it can and asks the user only on genuine ambiguity.
- The **exact tooling** that realizes working-context file access (e.g. a repo-path-scoped agent tool, an extension to `run_command`'s sandbox, or a VFS-backed tool) is a **design decision** the `architect` makes against real source — but it MUST satisfy the generality and no-special-casing constraints (FR-003/FR-004) and the three-way-read + hunk-write capability (FR-005).
- The **UI form factor is decided** (D1): the conflict-resolution pane lives in **Build Studio**, launched via the event system's UI-handler mechanism, using BS's existing chat as the agent↔user channel. The spec captures the *requirements* (FR-007…FR-011, FR-007a); the visual/interaction design is the mockup (the next pipeline step, before `design`, per the agreed UI-acceptance-before-architect process).
- **Dependency on the 034 event system**: the auto-launch (FR-007) and restart re-emit (FR-024) rely on the existing event & notification system's UI-handler dispatch and `redispatchPendingOnBoot`. This feature emits events and declares a UI handler; it does not modify the event system's core dispatch.
- **All in scope, no deferrals**: the completeness invariant (FR-016) means any dead-end path the architect discovers in `design` is covered by this feature.
- The user-apps promote path partly lives in the Supervisor process; the architect MUST pin its conflict call site(s) and how the escalation there reaches the generalized mechanism.
- The `inFlightEscalations` guard, rollback-tag step, and linear-main invariant already exist in `reconcile.ts`; this feature inherits and extends them.

## Resolved decisions

- **D1 — UI surface**: The conflict-resolution UI lives in **Build Studio** as a custom pane in its main view, launched automatically via the event system's UI-handler mechanism (034-event-notification-system). BS registers a UI handler for `com.bos.gitops.conflict.escalated`; when that event is emitted, the event system opens BS (if not already open) or focuses it, passing a component hint that routes BS to the conflict-resolution pane. BS's **existing chat** connects to the DevOps Agent conversation (`devopsConversationId`) — the agent's messages, decision requests, and the user's answers all flow through that existing channel; no new communication mechanism is introduced. The 3-way view, per-hunk decision controls, and session status render in a custom pane in BS's main view, alongside the chat. *(How the session id reaches the pane — component-hint embedding vs. a "what session is active?" query on launch — is an architect detail, not a spec concern.)*
- **D2 — Autonomous-first**: confirmed. The agent decides when it can and asks the user only on genuine ambiguity (PR-2).
- **D3 — `awaiting-user` timeout & restart recovery**: a parked `awaiting-user` session waits **indefinitely** — the 25-min timeout applies to the agent's *working* phases only, never to a parked state. On BOS restart (or browser refresh), the system **auto-detects** sessions and recovers them: a session in `working` whose agent run is dead is re-launched (boot-time sweep + the event system's existing `redispatchPendingOnBoot`); a session in `awaiting-user` is simply restored — the agent does not re-launch until the user answers. The session is persisted (FR-002), so this is restoration of durable state, not re-derivation.
- **D4 — User-apps call site**: architect to pin the exact Supervisor-process conflict path(s) during `design` (recorded as FR-015).
