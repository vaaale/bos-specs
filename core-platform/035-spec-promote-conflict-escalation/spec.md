# Feature Specification: Spec Promote Conflict Escalation

**Feature Branch**: `035-spec-promote-conflict`

**Created**: 2026-08-24

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "When the Supervisor promotes a feature branch and the user-specs store hits a merge conflict, the conflict currently dead-ends — no agent is triggered. The existing reconcile() pipeline in src/lib/gitops/reconcile.ts already has a 5-step escalation mechanism (rollback tag → sync → merge strategy → rebase fallback → DevOps Agent) that is wired for the BOS source repo but NOT for the user-specs store. This feature wires the user-specs promote path into that same escalation mechanism."

## Background

The Supervisor's promote pipeline handles two separate git repos:

1. **BOS source repo** — already uses `reconcile()` via `/api/gitfs/reconcile`. On conflict, the DevOps Agent is triggered; the UI shows the live conversation (`state: "escalated"`, `devopsConversationId`). This works.

2. **User-specs store** (`data/specs/`) — `src/lib/specs/promote.ts` → `promoteFeature()` does a bare `git merge --no-edit <base>` inside the feature worktree. On conflict it calls `merge --abort` and returns `{ kind: "conflict", files }`. No agent is triggered; the promote is blocked with a static error. This is the bug.

**Repro**: Branch `bos/034-event-notification-system` fails to promote with:
```
promote blocked — user-specs: branch bos/034-event-notification-system conflicts with master:
CONFLICT (add/add): Merge conflict in core-platform/034-event-notification-system/test-results.md
```

**Known architectural gap**: `reconcile()`'s escalation hands the task to the DevOps Agent, whose `dev_delegate` tool is scoped to the BOS source repo's feature-branch worktree. The user-specs store is a separate git repo. The DevOps Agent's `run_command` tool is sandboxed to `/workspace` and `/tmp` and cannot reach the user-specs worktree path on the host filesystem. The exact mechanism by which the escalated agent gains the ability to resolve file-level conflicts in the user-specs worktree is the core design question for this feature. [NEEDS CLARIFICATION: mechanism — see Assumptions]

## User Scenarios & Testing

### User Story 1 — Promote a feature with a spec-store conflict; the agent resolves it (Priority: P1)

The user clicks Promote on a feature branch. The Supervisor's promote pipeline runs. The user-specs store hits a merge conflict (e.g. two branches both added `test-results.md`). Instead of the promote dead-ending with a static error, the system triggers the DevOps Agent to resolve the conflict. The user sees the escalate state appear in the UI and can open the live conversation to watch the agent work. When the agent finishes, the promote either completes (conflict resolved, main fast-forwarded) or reports a clear failure with the rollback tag for manual recovery.

**Why this priority**: This is the exact bug being reported. Without it, every spec-store conflict blocks promote indefinitely and requires manual CLI intervention.

**Independent Test**: Create two feature branches that both add a file with the same path but different content to the user-specs store. Promote one while the other is on main. Verify: (a) the escalate state appears in the UI within seconds, (b) a DevOps Agent conversation is created and visible in the Assistant app, (c) the agent either resolves the conflict (promote completes) or times out with a clear error + rollback tag.

**Acceptance Scenarios**:

1. **Given** a feature branch whose user-specs changes conflict with main, **When** the user clicks Promote, **Then** the Supervisor's promote response includes `devopsConversationId` and the UI transitions to the escalated state within the same request cycle (no separate poll needed to discover the escalation).

2. **Given** the DevOps Agent is actively resolving the conflict, **When** the user opens the conversation in the Assistant app, **Then** they see the agent's in-progress work (tool calls, file reads, merge attempts) in real time, and the conversation is titled after the conflict being resolved.

3. **Given** the DevOps Agent resolves the conflict successfully, **When** the run completes, **Then** the Supervisor's promote completes: main is fast-forwarded to the reconciled branch, the worktree is pruned, and the promote response reports success.

4. **Given** the DevOps Agent times out (default 25 min) or errors, **When** the run reaches a terminal state, **Then** the promote reports failure with the rollback tag name and the conflicting files, and main is left untouched (never in a conflicted state).

---

### User Story 2 — The UI surfaces the escalated state for spec-store conflicts (Priority: P2)

The `ConflictResolutionDialog` (shown in VersionControls / VersionsTab when a promote hits a conflict) currently shows a static list of conflicting files with a message like "resolve manually or force-push." With this feature, when the conflict has been escalated to an agent, the dialog MUST reflect that: show the live conversation link, the agent's current state (working / completed / timed out), and the rollback tag. The user does not need to hunt for the conversation in the Assistant app — the dialog links to it directly.

**Why this priority**: The P1 story is functionally complete without this, but the user experience is poor — they'd have to know to go look in the Assistant app. The existing source-repo promote path already does this (VersionControls.tsx line 343 handles `devopsConversationId`); this story makes the spec-store path match it.

**Independent Test**: Trigger a spec-store promote conflict. Verify the ConflictResolutionDialog shows: (a) the "handed to the DevOps Agent" indicator, (b) a clickable link that opens the conversation in the Assistant app, (c) the conflicting file list, (d) the rollback tag name.

**Acceptance Scenarios**:

1. **Given** a promote is in the escalated state (agent working), **When** the ConflictResolutionDialog is rendered, **Then** it shows an amber "handed to the DevOps Agent" indicator with a link to the live conversation, and the conflicting file list.

2. **Given** the agent's run has completed (success or failure), **When** the user re-opens or refreshes the dialog, **Then** it shows the terminal state (resolved / timed out / error) and the rollback tag for manual recovery if needed.

---

### User Story 3 — The source-repo promote path is unaffected (Priority: P3)

The existing `reconcile()` escalation for the BOS source repo must continue to work identically. This feature changes `promoteFeature` (user-specs path) to call `reconcile()`; it does NOT change `reconcile()` itself in a way that alters the source-repo path's behavior. The `featureBranchForDelegate` field (already in `ReconcileOptions`) distinguishes the two cases: source-repo promotes pass it, user-specs promotes do not (or pass it with a value that tells the agent the worktree is the user-specs store, not the source repo).

**Why this priority**: Regression protection. The source-repo path is the one that already works; it must not break.

**Independent Test**: Trigger a source-repo promote conflict (as before this feature). Verify the escalation behavior is unchanged: same conversation creation, same `devopsConversationId` in the response, same UI state.

**Acceptance Scenarios**:

1. **Given** a source-repo promote conflict (as today), **When** the promote is escalated, **Then** the behavior is identical to before this feature (conversation created, `devopsConversationId` in response, UI shows escalated state).

2. **Given** the `reconcile()` function, **When** called with `featureBranchForDelegate` set (source-repo path), **Then** the escalation task text tells the agent the worktree IS the Supervisor-tracked feature branch (existing behavior, unchanged).

3. **Given** the `reconcile()` function, **When** called for the user-specs path (new), **Then** the escalation task text tells the agent the worktree is the user-specs store and `dev_delegate` does NOT apply — the agent must use whatever mechanism this feature provides to access the user-specs worktree.

---

### Edge Cases

- **Concurrent promotes**: Two promotes racing against the same user-specs branch. The existing `inFlightEscalations` map in `reconcile.ts` (keyed by `repoPath`) should prevent a second escalation from starting while one is in flight — verify this works for the user-specs repo path too.
- **Agent resolves the conflict but the result still doesn't merge cleanly**: The agent's resolution commit is itself a valid merge; if it's wrong, the user can use the rollback tag to restore the pre-reconciliation state and retry.
- **The user-specs worktree doesn't exist when promote starts**: `promoteFeature` already calls `ensureWorktree` — this is unchanged.
- **The conflict is in a binary file**: The agent may not be able to resolve it intelligently. The fallback is the same as today: rollback tag + manual intervention.
- **`main` is already up to date (no actual divergence)**: `promoteFeature`'s merge is a no-op; `reconcile()` handles this (the `hasStagedChanges` check in `attemptStrategy`).

## Requirements

### Functional Requirements

- **FR-001**: When `promoteFeature` (in `src/lib/specs/promote.ts`) encounters a merge conflict during the base-into-branch merge, it MUST route the conflict through the `reconcile()` pipeline (in `src/lib/gitops/reconcile.ts`) instead of calling `merge --abort` and returning a static `{ kind: "conflict" }` result.

- **FR-002**: The `reconcile()` call for the user-specs path MUST be a purely-local reconciliation (no remote sync step): `sourceRef` = the base branch name, `remote` omitted.

- **FR-003**: When the `reconcile()` pipeline's automated steps (merge strategy + rebase fallback) both fail on the user-specs path, it MUST escalate to the DevOps Agent — same as the source-repo path.

- **FR-004**: The escalated DevOps Agent run MUST have a working mechanism to read and write files in the user-specs worktree (at `data/specs/.worktrees/<encoded-branch>/`). [NEEDS CLARIFICATION: the exact mechanism — see Assumptions. Candidate options: (a) a new agent tool that operates on a specified repo path, (b) extending `run_command`'s sandbox to include the data directory, (c) a dedicated "spec-conflict" tool that performs targeted file merges programmatically, (d) the DevOps Agent uses an existing tool that can reach VFS paths.]

- **FR-005**: The Supervisor's promote response (the `PostResult` returned to the UI) MUST include `devopsConversationId` when a user-specs conflict is escalated, so the UI can surface the live conversation — same shape as the source-repo path already uses.

- **FR-006**: The `main` branch of the user-specs store MUST never be left in a conflicted state. The existing invariant in `promoteFeature` (main only fast-forwards; conflicts surface on the feature branch) MUST be preserved.

- **FR-007**: A rollback tag MUST be created in the user-specs repo before any merge/rebase attempt (the `reconcile()` pipeline already does this — FR-001 inherits it).

- **FR-008**: The `ConflictResolutionDialog` (or equivalent UI in VersionControls / VersionsTab) MUST display the escalated state for user-specs conflicts: the "handed to the DevOps Agent" indicator, a link to the conversation, the conflicting file list, and the rollback tag. This matches the behavior the source-repo path already has in `VersionControls.tsx`.

- **FR-009**: The source-repo promote path's escalation behavior MUST be unchanged. The `reconcile()` function's behavior when called with `featureBranchForDelegate` set (source-repo case) MUST be identical to before this feature.

- **FR-010**: The `promoteFeature` result type MUST be extended (or the Supervisor's promote logic updated) to distinguish between "escalated, agent working" (the promote is still in progress — not yet a terminal failure) and "escalated, agent failed/timed out" (terminal failure, rollback tag available). The UI needs this distinction to show the right state.

### Key Entities

- **User-specs promote conflict**: A merge conflict in the user-specs git store (at `data/specs/`) between a feature branch and the base branch, detected during `promoteFeature`. Key attributes: conflicting file paths, the feature branch name, the rollback tag.
- **DevOps Agent conversation**: A persisted, resumable Assistant conversation scoped to the DevOps Agent, created by `reconcile()`'s escalation step. Key attributes: conversation ID, title (titled after the conflict), `activeFeatureBranch` (for source-repo case; absent for user-specs case), agent ID (`devops`).

## Success Criteria

### Measurable Outcomes

- **SC-001**: A user-specs promote conflict that previously dead-ended (static "promote blocked" error, no agent) now triggers a DevOps Agent conversation within the same request cycle, and the UI shows the escalated state without requiring a manual refresh.

- **SC-002**: The user can resolve a spec-store promote conflict end-to-end (promote → escalate → agent resolves → promote completes) without opening a terminal or running any git commands manually, for at least text-file conflicts (e.g. two branches adding the same Markdown file with different content).

- **SC-003**: The source-repo promote conflict escalation path is unchanged — a source-repo promote conflict produces the same UI state, conversation, and response shape as before this feature (no regression).

- **SC-004**: When the agent times out or fails, the promote reports a clear error including the rollback tag name, and `git log --oneline` on the user-specs `main` branch shows it was never in a conflicted state (the fast-forward invariant held).

## Assumptions

- The `reconcile()` pipeline's 5-step structure (rollback tag → sync → merge strategy → rebase fallback → agent escalation) is the correct and sufficient mechanism for the user-specs path. No new pipeline steps are needed; the existing escalation is what the user wants.
- The DevOps Agent is the right agent for this (not a new, purpose-built agent). The same agent that resolves source-repo conflicts should resolve user-specs conflicts.
- The user-specs worktree path (`data/specs/.worktrees/<encoded-branch>/`) is accessible to the agent via SOME mechanism — the exact mechanism is the core design question and is marked [NEEDS CLARIFICATION] in FR-004. The Developer (via the `design` step) must resolve this before `plan`.
- The "pull from git" path (`/api/git-remotes/route.ts` `case "fetch"`) also has the same dead-end pattern (inline rebase, no escalation), but is OUT OF SCOPE for this spec. It should be a follow-up feature that reuses the same mechanism.
- The UI changes (FR-008) are an enhancement to the existing `ConflictResolutionDialog` / `VersionControls.tsx` — no new window or app is needed. The existing "handed to the DevOps Agent" UI pattern (already used for source-repo conflicts) is the model.
