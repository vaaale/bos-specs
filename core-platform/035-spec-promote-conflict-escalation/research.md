# Research / Decision Log: 035 Git Conflict Resolution System

Date: 2026-08-24 · Source of truth for *why* the design chose what it chose. All claims grounded in `design.md`, `design-review.md`, and source read during `design`.

## R1. Snapshot from **refs**, not live `:1/:2/:3` merge stages

**Decision**: the conflict snapshot is derived from `base = git merge-base <branch> <sourceRef>`, `ours = <branch>`, `theirs = <sourceRef>`, with per-file content via `git show <ref>:<rel>`.

**Why**:
- The existing `reconcile()` pipeline **aborts the merge/rebase before escalating** (`attemptStrategy` → `merge --abort` / `resetHardAndClean`; `attemptRebaseFallback` → `rebase --abort` + state-dir removal). So at escalate time there are **no live index stages** to read. Capturing `:1/:2/:3` in `escalate()` would fail (review M1).
- Refs are **uniform** across working-tree and plumbing modes (plumbing has no worktree at all — refs are the only source), **restart-safe** (FR-002: a resumed session re-derives identical content from the recorded refs), and **correct for add/add** (base side is absent at the merge-base → caught as empty).
- The conflicting-**file list** comes from `git diff --name-only --diff-filter=U` (working-tree paths; `promoteFeature` already uses it) or by parsing `git merge-tree --write-tree` output (dry-run / pre-check paths; `coupledConflicts` already does this). The existing codebase treats `merge-tree --write-tree` as a **boolean** conflict signal (non-zero exit) and takes the merged tree sha from stdout line 1 — it does not parse per-file hunks, so per-file *content* is obtained from `git show <ref>:<rel>`, which is simpler and uniform.

**Rejected**: live-stage capture (broken for the reasons above); storing full file content in the session record (duplicates git's data, not restart-safe on a clean clone).

## R2. `add/add` base side is empty

**Decision**: `readFileAtRef(repoPath, baseRef, rel)` catches the "file not found at ref" error and returns empty/`undefined` for the base side.

**Why**: the repro conflict (`CONFLICT (add/add)` on `test-results.md`) is exactly this — both branches add the file, so it is absent at the merge-base. `git show <mb>:<rel>` throws; the catch is mandatory, not optional. The schema anticipates it (`ConflictHunk.base?: string`).

## R3. **Park-and-rewake** across decision requests (not in-run blocking)

**Decision**: when the agent issues `conflict_decision`, its run **ends** (`awaiting-user`); the user's answer **re-launches** a new run on the **same** conversation id, whose first user message is the answer.

**Why** (verified in source, design-review §2):
- `startAssistantRun({ conversationId, agentId, message })` **continues** an existing conversation — `runAgentLoop` loads the full transcript (`io.loadMessages()`) then appends the new user message. The agent "remembers" prior work from the transcript, so continuing is genuinely satisfied (not just message-injection).
- `runManager.create` throws `ActiveRunError` only while a run is `"running"`; `finish` frees the slot immediately. Park (run ends) → rewake (new run) has **no run-slot race**.
- In-run `await` (blocking on an event/tool-result) was **rejected**: it would hold a run/loop open indefinitely (the `awaiting-user` state waits *indefinitely* by design — D3), pin the process, and complicate restart (a blocked in-process await can't survive a restart). Park-and-rewake makes `awaiting-user` trivially restart-safe: the session is on disk, no live run to lose.

## R4. Auto-launch via the 034 event system (emit + UI handler + topbar subscriber)

**Decision**: the pipeline emits `com.bos.gitops.conflict.escalated`; Build Studio declares a UI handler for it (`AppManifest.eventHandlers` + `eventNamespaces: ["com.bos.gitops.*"]` grant); a new topbar subscriber (sibling of `EventBell`) consumes it and calls `launch("build-studio", { pane: "conflict", sessionId })`.

**Why** (verified, design-review §4):
- `launch` on the `build-studio` **singleton** focuses the existing window and **merges `params`** (or opens it if closed) — exactly the "fire up / focus + show pane" behavior D1 needs.
- `ownsNamespace` returns true **only with** the `eventNamespaces` grant — Build Studio owns `com.bos.build-studio.*`, not `com.bos.gitops.*`, so the grant is genuinely required.
- A separate subscriber (not reusing `EventBell`'s connection) is correct because `subscribeEventStream` is per-caller.
- The **Supervisor** (separate Node process) emits via its existing loopback HTTP (the same path it uses for `/api/gitfs/reconcile`) — it never imports BOS `@/` source.

## R5. The boot re-emit is **explicit**, not `redispatchPendingOnBoot`

**Decision**: the boot sweep (`recoverSessions()` in `src/instrumentation.ts`) explicitly re-emits the event for each non-terminal session.

**Why**: `redispatchPendingOnBoot` re-enqueues **pending** events to **active headless** handlers only. This event is UI-only → it is `processed/no-active-handlers` at emit → it is **never `pending`** → `redispatchPendingOnBoot` will never re-fire it. So the spec's earlier "reuse `redispatchPendingOnBoot`" wording was wrong (corrected in spec FR-024/D3/Assumptions). The double-emit (original + boot) for a session created just before restart is **benign** because singleton `launch` is idempotent.

## R6. Session storage = `data/gitops/sessions/` (not VFS, not a spec store)

**Decision**: one JSON file per session under `data/gitops/sessions/<sessionId>.json`, written via `writeFileAtomic`.

**Why**: a resolution session is **runtime state** (constitution V: "All runtime state persists as files under ./data"). It is not user-authored versioned content (so not a GitFS content repo) and not the user's sandbox (so not `data/vfs`). Per-file (not one big array) keeps a concurrent-session write from clobbering siblings; `writeFileAtomic` prevents a torn write on crash. The **snapshot is not stored** — only the refs + file list — so a session record stays small and re-derivable.

## R7. The conflict agent is **user-configurable**, default `devops` (FR-025)

**Decision**: `build-studio.conflictAgent` (default `"devops"`) is read **at escalation time** and used as the `agentId` for the escalation conversation, replacing the hard-coded `DEVOPS_AGENT_ID = "devops"` in `reconcile.ts`.

**Why**: one global, user-tunable resolver for every repo (the user explicitly rejected per-repo agents). Reading on each escalation means no reload needed. **Consequence (R-upgrade risk)**: the resolved agent **must** have the `conflict_*` tool ids in its allowlist (the tool gate in `gate.ts` enforces `agent.tools`), so the default `devops` seed must gain them — and a pre-existing `data/agents/devops` must be backfilled.

## Open (pinned in design, tracked as tasks)

- Exact `devops` seed file path + tools-array field (design S4; verify against `src/lib/agent/` seed location in `tasks`).
- The `getConversationConflictSessionId` getter (mirror `getConversationActiveFeatureBranch` in `src/lib/agent/conversations-server.ts`) — design S1.
