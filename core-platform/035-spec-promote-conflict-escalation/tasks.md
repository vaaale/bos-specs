# Tasks: 035 Git Conflict Resolution System

**Branch**: `bos/035-spec-promote-conflict`
**Spec**: `spec.md` | **Design**: `design.md` | **Plan**: `plan.md` | **Data Model**: `data-model.md` | **Quickstart**: `quickstart.md`

**User stories** (from spec, in priority order):
- **US1** (P1): End-to-end promote conflict resolution — the core engine
- **US2** (P1): User interaction in Build Studio — the conflict pane + decision loop
- **US3** (P1): Cross-repo generality — all call-site conversions
- **US4** (P2): Other BOS surfaces show conflict state — FR-019 surfaces
- **US5** (P2): Configurable conflict-resolution agent — FR-025
- **US6** (P2): Restart/recovery — FR-024

---

## Phase 1: Setup

No project initialization needed (existing Next.js project). No new dependencies.

---

## Phase 2: Foundational (blocking — all US phases depend on these)

- [ ] T001 Create session types (`ConflictSession`, `WorkContext`, `ConflictSnapshot`, `ConflictHunk`, `ConflictDecision`, `SessionStatus` enum, `SessionResult`) in `src/lib/gitops/sessions/types.ts`
- [ ] T002 [P] Create the `readFileAtRef(repoPath: string, ref: string, rel: string): Promise<string | null>` helper in `src/lib/gitops/git-ops.ts` — wraps `runGitCommand(["show", `${ref}:${rel}`], { cwd: repoPath })`, returns `null` on "path not found" (add/add base side)
- [ ] T003 [P] Add `build-studio.conflictAgent` (default `"devops"`) to the `build-studio` namespace `load()` in `src/lib/config/registry.ts`
- [ ] T004 Create the session store (`createSession`, `getSession`, `listActiveSessions`, `updateSession`, `transitionSession`) in `src/lib/gitops/sessions/store.ts` — one JSON file per session under `data/gitops/sessions/<id>.json`, written via `writeFileAtomic` from `src/os/atomic-write.ts`
- [ ] T005 Add `getConversationConflictSessionId(conversationId: string): Promise<string | null>` in `src/lib/agent/conversations-server.ts` (mirror `getConversationActiveFeatureBranch` — reads the `conflictSessionId` top-level field from the conversation JSON file in `/Documents/Chats/`)

---

## Phase 3: US1 — End-to-end promote conflict resolution (the core engine)

**Goal**: When `reconcile()` hits a conflict that deterministic steps can't resolve, it creates a session, emits the event, and launches the configured agent with the repo-parameterized working context. The source-repo path (the existing escalation) is the first instance of the general mechanism and MUST NOT regress.

**Independent test**: Run a BOS-source feature-promote that conflicts with `main`. Assert: a session is created (status `working`), the event `com.bos.gitops.conflict.escalated` is emitted, the existing DevOps conversation is created with `activeFeatureBranch` set AND `conflictSessionId` set, the agent run starts, and `main` is unconflicted (rollback tag exists).

- [ ] T006 Generalize `reconcile.ts` step 5 (escalate): accept a `WorkContext` parameter (or derive it from `repoPath` + the pipeline's existing `remote`/`branch`/`sourceRef` fields); replace the hard-coded `DEVOPS_AGENT_ID` with `getConfigValue("build-studio", "conflictAgent")`; call `createSession()` with the working context + snapshot (refs: `base = merge-base`, `ours = branch`, `theirs = sourceRef`; file list from `diff --name-only --diff-filter=U`); tag the conversation file with `conflictSessionId` (alongside the existing `activeFeatureBranch`); emit `com.bos.gitops.conflict.escalated` with payload `{ sessionId, repoLabel, featureBranch, rollbackTag, repoKind }`; extend `inFlightEscalations` value to carry `sessionId` and handle the `awaiting-user` state in the concurrent-op guard
- [ ] T007 [P] Create the six `conflict_*` server tools in `src/lib/assistant/tools/server/conflict-resolve.ts` using the `serverTool(...)` pattern from `src/lib/assistant/tools/server/util.ts`: `conflict_read(file?)` (derives 3-way via `readFileAtRef` for `base`/`ours`/`theirs` refs, returns `ConflictHunk[]`), `conflict_write(file, resolution)` (applies the user/agent resolution to the working tree via git ops), `conflict_decision(question, options, file?, hunk?)` (records the decision, transitions session to `awaiting-user`, ends the run), `conflict_status(file?)` (returns per-file resolution progress), `conflict_complete(summary)` (transitions session to `resolved`, completes the reconciliation), `conflict_abandon(reason?)` (resets working tree to `rollbackTag`, transitions session to `abandoned`). Each tool reads `conflictSessionId` from `ctx.conversationId` via `getConversationConflictSessionId`
- [ ] T008 Register the `conflict-resolve.ts` module in `src/lib/assistant/registry.ts` (spread into the combined server tools map)
- [ ] T009 Add the six `conflict_*` tool ids to the `devops` agent's `tools` array (find the exact seed file in `src/lib/agent/` or the bundled seed — verify the path during implementation); add a backfill in the agent store (`src/lib/agent/` store) so a pre-existing `data/agents/devops/AGENT.md` gains the tools on upgrade (R-upgrade risk)
- [ ] T010 Add the `sessionId` field to the promote response in `tools/supervisor/lib/promote.mjs` (FR-018) — when the reconcile job returns `status: "escalated"` with a `devopsConversationId`, the session id is also returned and included in the promote response JSON
- [ ] T011 Unit tests: `reconcile()` generalization (each step's outcome + the new session creation path), `readFileAtRef` (incl. add/add base-side catch → null), session store transitions + `writeFileAtomic` atomicity, `inFlightEscalations` guard with `awaiting-user` state

---

## Phase 4: US2 — User interaction in Build Studio (conflict pane + decision loop)

**Goal**: Build Studio auto-launches on the conflict event, shows the conflict pane (matching the accepted mockup), and the user's answers flow through the existing chat to re-wake the agent run.

**Independent test**: Trigger a conflict → BS opens/focuses with the conflict pane visible → the agent's decision question appears in the chat and the pane → user clicks "accept theirs" → the session transitions back to `working`, the agent run re-launches, and the hunk is marked resolved.

- [ ] T012 [P] Add `eventHandlers: [{ type: "com.bos.gitops.conflict.escalated", component: "conflict" }]` and `eventNamespaces: ["com.bos.gitops.*"]` to `src/apps/build-studio/manifest.json`
- [ ] T013 Create the topbar subscriber `src/components/desktop/ConflictLaunch.tsx` — a sibling of `EventBell.tsx` that calls `subscribeEventStream` for `com.bos.gitops.conflict.escalated` and on receipt calls `launch("build-studio", { pane: "conflict", sessionId: payload.sessionId })`
- [ ] T014 Read `pane` and `sessionId` params in `src/apps/build-studio/index.tsx` — when `pane === "conflict"`, render the conflict pane in the center column (replacing the artifact viewer); re-point the existing chat to the session's conversation id (via the session API)
- [ ] T015 Create the conflict pane component tree under `src/apps/build-studio/conflict/` (match the accepted mockup): `ConflictPane.tsx` (layout: status header + files list + 3-way view + chat), `ConflictStatusHeader.tsx` (status badge: working/awaiting-user/resolved/failed/timed-out; repo label; branch; rollback tag; "Abandon & roll back" button), `ConflictFileList.tsx` (clickable file rows with per-file status chips), `ConflictFileView.tsx` (3-way view: unified with markers + 3-way columns toggle; per-hunk decision controls: accept ours/theirs/keep both/edit manually/accept agent's suggestion), `ConflictDecisionCard.tsx` (the agent's pending decision question rendered in the pane with answer controls; amber-styled for `awaiting-user`)
- [ ] T016 Wire the decision answer: when the user clicks a decision control, `PATCH /api/gitops/sessions/<id>` with the answer; the API records the decision, transitions the session back to `working`, and re-launches the agent run via `startAssistantRun({ conversationId, agentId, message: "<answer>" })` (park-and-rewake, design R3)
- [ ] T017 Create the session API route `src/app/api/gitops/sessions/route.ts`: `GET` (list non-terminal), `GET ?id=` (one session), `PATCH ?id=` (status transition + decision answer — the only mutation the client can make; all git ops stay server-side via the agent tools)
- [ ] T018 Unit tests: session API transitions, the park-and-rewake re-launch (mock `startAssistantRun`), the topbar subscriber's `launch` call

---

## Phase 5: US3 — Cross-repo generality (all call-site conversions)

**Goal**: Every deterministic-conflict dead-end in the system routes through the generalized pipeline with the correct `WorkContext`. No dead-end remains (FR-016 completeness invariant).

**Independent test**: For each call site below, trigger the specific conflict scenario and assert: a session is created with the correct `WorkContext.repoKind` and `repoPath`, the event is emitted, the agent launches, and `main`/base is unconflicted.

- [ ] T019 [US3] Convert `coupledConflicts` in `tools/supervisor/lib/coupled-repos.mjs` (the Supervisor pre-check, **FR-012a / the reported bug**): instead of `throw new Error(\`promote blocked — ${conflict}\`)`, call the reconcile job via the existing loopback HTTP (`/api/gitfs/reconcile`), passing the correct `WorkContext` for each affected store (user-specs: `repoPath = <worktree>/specs/user-specs`, `repoKind = "user-specs"`; user-apps: `repoKind = "user-apps"`). The pre-check creates the session + escalates instead of throwing. `promote()` in `promote.mjs` returns the `sessionId` in its response (FR-018) instead of throwing
- [ ] T020 [US3] Convert `promoteCoupled` in `tools/supervisor/lib/coupled-repos.mjs`: replace the working-tree path's `merge --abort` + warning and the plumbing path's no-handling with a route-through-the-pipeline call (same loopback, same `WorkContext` as T019)
- [ ] T021 [P] [US3] Convert `promoteFeature` in `src/lib/specs/promote.ts` (**FR-012**): replace `merge --abort` + `return { kind: "conflict", files }` with a call to `reconcile()` using the user-specs `WorkContext` (`repoPath = <worktree>/specs/user-specs`, `sourceRef = <feature-branch>`, `strategy = "merge"`). On escalation, return `{ kind: "escalated", sessionId }` instead of `{ kind: "conflict" }`
- [ ] T022 [P] [US3] Convert `case "fetch"` in `src/app/api/git-remotes/route.ts` (**FR-013**): replace the `rebaseOntoRemote` → `rebaseConflict: true` dead-end with a `reconcile()` call using the repo's `WorkContext`. On escalation, return `{ sessionId }` in the response instead of the static `rebaseConflict` message
- [ ] T023 [P] [US3] Convert the `case "push"` recovery path in `src/app/api/git-remotes/route.ts` (**FR-014**): same as T022 for the non-FF recovery rebase
- [ ] T024 [P] [US3] Convert `appPromote` in `tools/supervisor/lib/app-candidate.mjs` (**FR-015**): replace the raw `git merge --no-edit` with a route-through-the-pipeline call via loopback, `WorkContext.repoKind = "user-apps"`. On escalation, return `sessionId` in the app-promote response
- [ ] T025 [US3] Unit tests: each call-site conversion (mock the reconcile call, assert the correct `WorkContext` is passed and the dead-end path is removed); the VFS-mounted repo case (S6 — `WorkContext.repoPath` points to the mount root, same tool mechanism)

---

## Phase 6: US4 — Other BOS surfaces show conflict state (FR-019)

**Goal**: The four existing surfaces that can trigger or observe a conflict show the live session state (status badge, file count, rollback tag, "Open resolution" button) instead of static dead-end messages.

**Independent test**: For each surface, trigger the conflict that would have produced the old static message, and assert the new session-aware UI is shown with a working "Open resolution" button that launches BS with the conflict pane.

- [ ] T026 [P] Update `src/components/desktop/VersionControls.tsx`: replace the static "escalated to DevOps Agent" text (line ~343) with a session-aware indicator — read `sessionId` from the promote response (FR-018), poll `GET /api/gitops/sessions/<id>` for status, show the status badge + conflicting file count + rollback tag + "Open resolution" button (calls `launch("build-studio", { pane: "conflict", sessionId })`)
- [ ] T027 [P] Update `src/components/apps/settings/VersionsTab.tsx`: when a promote response carries `sessionId`, show the session state in the tab (same indicator as T026)
- [ ] T028 [P] Update `src/components/apps/settings/versions/ConflictResolutionDialog.tsx`: replace the static conflict file list with a session-aware version — live session status, file list from the session snapshot, and a link to the Build Studio conflict pane
- [ ] T029 [P] Update `src/components/apps/settings/versions/GitRemotesTab.tsx`: replace the static `rebaseConflict: true` message (line ~693) with the session state — when the fetch/push response carries `sessionId`, show the session status badge + "Open resolution" button instead of "resolve manually, or force-push"

---

## Phase 7: US5 — Configurable conflict-resolution agent (FR-025)

**Goal**: The user can select which agent resolves conflicts via a second dropdown in Settings → Build Studio. The setting is read at escalation time; no reload needed.

**Independent test**: Set `build-studio.conflictAgent` to a non-default agent → trigger a conflict → assert the escalation conversation is scoped to the selected agent. If the selected agent lacks `conflict_*` tools, assert the session is `failed` with a clear message naming the agent.

- [ ] T030 Add a second dropdown ("Conflict resolution agent") to `src/components/apps/settings/BuildStudioTab.tsx` — same `<select>` + Save pattern as the existing "Agent" field, storing to `build-studio.conflictAgent` (default `"devops"`), populated from `GET /api/subagents`
- [ ] T031 [P] Unit test: FR-025 — mock `getConfigValue("build-studio", "conflictAgent")` returning a non-default agent; assert `reconcile.ts` step 5 uses that agent id for the escalation conversation. Also test the loud-fail path: agent without `conflict_*` tools → session `failed` with the correct error message

---

## Phase 8: US6 — Restart/recovery (FR-024)

**Goal**: A session in `working` whose agent run is dead (BOS restart) is re-launched by the boot sweep. A session in `awaiting-user` is restored as-is (no re-launch until the user answers). A browser refresh restores the pane from the session store.

**Independent test**: Create a session, kill the process (or simulate via the boot sweep test), restart → assert: `working` session's run is re-launched on the same conversation; `awaiting-user` session is restored with the pending decision visible; the snapshot is re-derived from refs (identical content).

- [ ] T032 Create `recoverSessions()` in a new module `src/lib/gitops/sessions/recover.ts`: on boot, `listActiveSessions()` → for each session in `working` with a dead `runId` (no live run in `runManager`), re-launch the agent run via `startAssistantRun({ conversationId, agentId, message: "Resuming conflict resolution. Check the current session state via conflict_status and continue." })` and re-emit `com.bos.gitops.conflict.escalated` (the topbar subscriber re-launches the pane; the double-emit is benign per design S2). For each session in `awaiting-user`, re-emit only (no re-launch — the agent waits for the user's answer)
- [ ] T033 Call `recoverSessions()` from `src/instrumentation.ts` `register()` (the documented Next.js server-boot hook), after `startEventKernel()` and `registerAllUiHandlers()`
- [ ] T034 In `src/apps/build-studio/index.tsx`: on mount, if the app has no `pane` param but there's an active session for this repo (query `GET /api/gitops/sessions`), auto-switch to the conflict pane (handles the browser-refresh case — no event re-emit needed, the pane re-queries the session store on load)
- [ ] T035 [P] Unit tests: `recoverSessions()` — mock a `working` session with a dead run → assert `startAssistantRun` is called with the correct conversation id + re-emit; mock an `awaiting-user` session → assert only re-emit (no re-launch); mock the double-emit case (session created just before restart) → assert it's benign (singleton `launch` idempotent)

---

## Phase 9: Polish & cross-cutting

- [ ] T036 [P] Write `docs/dev/features/git-conflict-resolution.md` — developer-facing: the architecture (5-step pipeline + session + agent tools), the working-context parameterization, the park-and-rewake model, the call-site table, the Settings field, the event type + payload, the session state machine, and how to add a new call site
- [ ] T037 [P] Add a section to `docs/dev/build-studio.md` — the conflict pane (what it is, when it launches, how to interact with it) and the "Conflict resolution agent" Settings field
- [ ] T038 Write `e2e/035-conflict-resolution.spec.ts` covering quickstart scenarios S1–S12 (Playwright). Each scenario: set up the conflict condition, trigger the operation, assert the session is created + event emitted + BS pane visible, answer the decision, assert resolution + `main` unconflicted. S9 (restart recovery) requires a process-kill + restart cycle in the test harness
- [ ] T039 Run quality gates: `tsc --noEmit`, `lint`, unit tests, e2e suite — all green before promotion (constitution VII)
- [ ] T040 Verify the completeness invariant (FR-016): grep the codebase for any remaining `merge --abort` / `rebase --abort` / `REBASE_CONFLICT` / `MERGE_CONFLICT` error paths that do NOT route through `reconcile()`. If any are found, add them to the conversion list and re-run T019–T025 for those paths

---

## Dependency graph

```
Phase 2 (Foundational)
  T001 (types) ──────────────────────────────────────────────────────────────────┐
  T002 (readFileAtRef) ──────────────────────────────────────────────────────────┤
  T003 (config field) ───────────────────────────────────────────────────────────┤
  T004 (session store) ──────────────────────────────────────────────────────────┤
  T005 (conversation getter) ────────────────────────────────────────────────────┤
                                                                                 │
Phase 3 (US1 — core engine)                                                      │
  T006 (reconcile generalization) ── depends on T001, T002, T004, T005 ─────────┤
  T007 (conflict tools) ── depends on T001, T002, T004, T005 ───────────────────┤
  T008 (tool registration) ── depends on T007 ──────────────────────────────────┤
  T009 (devops seed) ── depends on T007 ────────────────────────────────────────┤
  T010 (promote response) ── depends on T006 ───────────────────────────────────┤
  T011 (unit tests) ── depends on T006, T007, T004 ─────────────────────────────┤
                                                                                 │
Phase 4 (US2 — BS conflict pane)                                                 │
  T012 (manifest) ── depends on nothing (Phase 2) ──────────────────────────────┤
  T013 (topbar subscriber) ── depends on T012 ──────────────────────────────────┤
  T014 (BS index read params) ── depends on T012 ───────────────────────────────┤
  T015 (conflict pane components) ── depends on T014, T001 ─────────────────────┤
  T016 (decision answer wiring) ── depends on T015, T004, T005 ─────────────────┤
  T017 (session API route) ── depends on T004 ──────────────────────────────────┤
  T018 (unit tests) ── depends on T016, T017 ───────────────────────────────────┤
                                                                                 │
Phase 5 (US3 — cross-repo)                                                       │
  T019 (coupledConflicts) ── depends on T006, T010 ─────────────────────────────┤
  T020 (promoteCoupled) ── depends on T019 ─────────────────────────────────────┤
  T021 (promoteFeature) ── depends on T006 [P with T022, T023, T024] ───────────┤
  T022 (git-remotes fetch) ── depends on T006 [P] ──────────────────────────────┤
  T023 (git-remotes push) ── depends on T006 [P] ───────────────────────────────┤
  T024 (appPromote) ── depends on T006 [P] ─────────────────────────────────────┤
  T025 (unit tests) ── depends on T019–T024 ────────────────────────────────────┤
                                                                                 │
Phase 6 (US4 — FR-019 surfaces)                                                  │
  T026–T029 [all P] ── depends on T017 (session API), T010 (promote response) ──┤
                                                                                 │
Phase 7 (US5 — configurable agent)                                               │
  T030 (Settings dropdown) ── depends on T003 ──────────────────────────────────┤
  T031 (unit test) [P] ── depends on T006, T030 ────────────────────────────────┤
                                                                                 │
Phase 8 (US6 — restart recovery)                                                 │
  T032 (recoverSessions) ── depends on T004, T013 ──────────────────────────────┤
  T033 (boot hook) ── depends on T032 ──────────────────────────────────────────┤
  T034 (BS mount restore) ── depends on T014, T017 ─────────────────────────────┤
  T035 (unit tests) [P] ── depends on T032 ─────────────────────────────────────┤
                                                                                 │
Phase 9 (Polish)                                                                 │
  T036 (docs) [P] ── depends on T006–T035 (all implementation done) ────────────┤
  T037 (BS docs) [P] ── depends on T030, T015 ──────────────────────────────────┤
  T038 (e2e tests) ── depends on T006–T035 ─────────────────────────────────────┤
  T039 (quality gates) ── depends on T038 ──────────────────────────────────────┤
  T040 (completeness invariant sweep) ── depends on T019–T025 ──────────────────┘
```

## Parallel execution opportunities

- **Phase 2**: T002 and T003 are independent of each other and of T001/T004/T005 — all five can run in parallel.
- **Phase 3**: T006 and T007 are independent of each other (different files) — parallel.
- **Phase 5**: T021, T022, T023, T024 are all independent of each other (different files, same dependency on T006) — four-way parallel.
- **Phase 6**: T026, T027, T028, T029 are all independent (different files) — four-way parallel.
- **Phase 9**: T036 and T037 are independent of each other.

## MVP scope

**US1 + US2** (the core engine + the Build Studio pane) is the MVP — it makes the reported bug (S1) work end-to-end: promote → conflict → escalate → BS pane → user answers → resolved. US3 (cross-repo) is the next increment; US4–US6 are P2 polish.
