# Tasks: Workflow Manager — Service-Owned Workflow Engine (Engine Pivot)

**Input**: Design documents from `/Specs/user-specs/workflow-manager/001-workflow-manager-service-tools/` (spec.md, design.md, plan.md, mockup.html)

**Prerequisites**: plan.md (required), spec.md (required — US1..US9, FR-001..024, NFR-001..005, SC-001..014)

**Tests**: Unit tests (per service module) + Playwright e2e (app) are **explicitly in scope** (user instruction: "include writing unit tests and e2e tests").

**Organization**: Tasks are grouped by user story to enable independent implementation and testing. This is a **dual-scope** feature — a `bos-core` retirement (foundational, on the active feature branch `bos/042-workflow-manager-service`) + a `marketplace-item` re-implementation (service-owned engine under `data/user-apps/items/workflows/`, delivered via `app_build`).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure — the item skeleton and the bos-core retirement baseline.

- [ ] T001 Create the marketplace-item skeleton at `data/user-apps/items/workflows/` with `services/`, `app/`, `config/`, `skills/` subfolders per `target-marketplace-item.md`
- [ ] T002 [P] Author `services/service.json` — `deploymentMode: "tools"`, entry `index.js`, configSchema (`port` default `0`, `host` default `127.0.0.1`, `maxConcurrentSteps`)
- [ ] T003 [P] Author `config/workflows.json` — `{ "port": 0, "host": "127.0.0.1", "maxConcurrentSteps": 5 }`
- [ ] T004 [P] Create the bos-core retirement baseline: snapshot current `src/lib/workflows/*` + `/api/workflows/*` behavior for reference before deletion

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented.

**⚠️ CRITICAL**: The **bos-core retirement** is the foundational blocker — it must land first so the service-owned engine can become the single authority without shadowing. No user story work can begin until this phase is complete.

- [ ] T005 Bos-core retirement: delete `src/lib/workflows/types.ts`, `runner.ts`, `store.ts`, `validate.ts`, `generate.ts` (whole old engine — user-approved pivot)
- [ ] T006 [P] Bos-core retirement: delete `/api/workflows/` routes — `route.ts`, `run/route.ts`, `status/route.ts`, `cancel/route.ts`, `generate/route.ts`, `validate/route.ts`
- [ ] T007 [P] Bos-core retirement: delete `src/lib/assistant/tools/server/workflows.ts` (the `workflowTools()` set) — no longer shadows service tools
- [ ] T008 Bos-core retirement: remove `workflowTools` import + spread from `src/lib/assistant/registry.ts`
- [ ] T009 [P] Bos-core retirement: delete `src/components/agent/WorkflowActions.tsx` and remove its `<WorkflowActions/>` usage from `src/components/agent/CopilotProvider.tsx`
- [ ] T010 [P] Bos-core retirement: remove the 7 static `workflow_*` capability entries from `src/lib/agent/capabilities-registry.ts`
- [ ] T011 Bos-core retirement: update `docs/dev/architecture-overview.md` §14 (workflows subsystem) to note the retirement — docs/spec drift tracking
- [ ] T012 [P] Service foundation: implement `services/vfs.js` — loopback `/api/fs` bridge helpers (ADR-2, real VFS, never host paths)
- [ ] T013 [P] Service foundation: implement `services/engine/node-model.js` — orthogonal node axes (agent source: static/ephemeral × output type: delegate/tool/research/ag-ui)
- [ ] T014 [P] Service foundation: implement `services/engine/validate.js` — DAG acyclicity + node schema validation (re-implemented from retired engine)
- [ ] T015 Service foundation: implement `services/engine/store.js` — workflow + run CRUD over loopback `/api/fs`
- [ ] T016 [P] Service foundation: implement `services/index.js` — worker entry: lifecycle + `tool_declare` + worker-IPC + migration bootstrap

**Checkpoint**: Foundation ready — the bos-core retirement has landed, the service skeleton + node model + validation + store + vfs are in place. User story implementation can now begin.

---

## Phase 3: User Story 1 — 039-Compliant Tool Surface (Priority: P1) 🎯 MVP

**Goal**: The workflow tools are exposed as service-declared native tools (039), replacing the retired server tools — the assistant can list/create/read/modify/run/status/cancel/delete/export/validate workflows.

**Independent Test**: Launch the service; confirm 12 tools are declared (`workflow_list/create/read/modify/run/status/cancel/delete/export/validate/run_list/run_get`); a workflow can be created, listed, read, and deleted via the tools without the UI.

### Tests for User Story 1 (in scope) ⚠️

> **NOTE: Write these FIRST, ensure they FAIL before implementation**

- [ ] T017 [P] [US1] Unit test for tool declarations + schemas in `<item>/services/__tests__/tools.test.js`
- [ ] T018 [P] [US1] Unit test for tool handlers in `<item>/services/__tests__/handlers.test.js`

### Implementation for User Story 1

- [ ] T019 [P] [US1] Implement `services/tools.js` — 12 tool declarations + input JSON-schemas (FR-002/014/016/024)
- [ ] T020 [US1] Implement `services/handlers.js` — tool handlers wired to the engine (create/read/modify/delete/export/validate/run/status/cancel/list/run_list/run_get) (depends on T019, T015)
- [ ] T021 [US1] Implement `workflow_list` returning `workflow_id`, `status` (running/idle), `run_id` when running (FR-024) in `services/handlers.js`
- [ ] T022 [US1] Add validation + error handling for all 12 tool handlers (per 039 R10, no leaked pending waits)

**Checkpoint**: US1 fully functional — the assistant can control workflows end-to-end via the 12 tools, independent of the UI.

---

## Phase 4: User Story 2 — Fire-and-Poll Run + Status (Priority: P1)

**Goal**: `workflow_run` is asynchronous fire-and-poll (returns `runId` immediately, ADR-8); `workflow_status`/`workflow_run_get` report live progress; `workflow_list` exposes running state for agent polling (FR-024).

**Independent Test**: Run a workflow via `workflow_run`; confirm it returns a `runId` in <1s (not blocking); poll `workflow_status` to observe progress; cancel it; confirm `workflow_list` reflects `running` → `idle`.

### Tests for User Story 2 (in scope) ⚠️

- [ ] T023 [P] [US2] Unit test for executor (delegate contract) in `<item>/services/__tests__/executor.test.js`

### Implementation for User Story 2

- [ ] T024 [P] [US2] Implement `services/engine/executor.js` — loopback `POST /api/subagents/delegate` per node (ADR-1), NDJSON, fire-and-poll `runId` (depends on T012)
- [ ] T025 [US2] Implement run-state tracking + `workflow_status`/`workflow_run_get` live status in `services/engine/store.js` + `services/runs.js` (depends on T024)
- [ ] T026 [US2] Implement cancellation (Option (b)): worker aborts its in-flight delegate fetch; inner-loop linked-abort settles `cancelled` — no bos-core delegate-route change (depends on T024)

**Checkpoint**: US1 + US2 work — workflows can be run asynchronously, polled, and cancelled via tools.

---

## Phase 5: User Story 3 — Real-VFS Persistence + Historical Runs (Priority: P2)

**Goal**: Workflows persist to real VFS `/Workflows/` via loopback `/api/fs` (ADR-2, never host paths); each run is a first-class persisted entity (`/Workflows/.runs/<workflowId>/<runId>.json`) — historical replay via `workflow_run_list`/`workflow_run_get` (US6/FR-015/016/017).

**Independent Test**: Create + run a workflow; confirm the workflow JSON + run log exist in the real VFS; list + read the historical run via `workflow_run_list`/`workflow_run_get`; delete a workflow and confirm its run log is removed.

### Tests for User Story 3 (in scope) ⚠️

- [ ] T027 [P] [US3] Unit test for vfs bridge in `<item>/services/__tests__/vfs.test.js`
- [ ] T028 [P] [US3] Unit test for run persistence in `<item>/services/__tests__/runs.test.js`
- [ ] T029 [P] [US3] Unit test for migration (additive, idempotent) in `<item>/services/__tests__/migration.test.js`

### Implementation for User Story 3

- [ ] T030 [P] [US3] Implement `services/runs.js` — run entity persistence + `workflow_run_list`/`workflow_run_get` (US6, FR-016) (depends on T015, T025)
- [ ] T031 [US3] Implement `services/migration.js` — additive legacy-workflow migration (ADR-5, copy + archive, never delete) (depends on T015)
- [ ] T032 [US3] Wire real-VFS workflow + run storage into `handlers.js` (create/read/delete/export read/write `/Workflows/*`) (depends on T030)

**Checkpoint**: US1 + US2 + US3 work — workflows and runs persist to the real VFS, historical runs are tool-reachable.

---

## Phase 6: User Story 7 — Dynamic Routing (Priority: P1)

**Goal**: A node may declare candidate sub-agents; the node's agent selects the appropriate candidate(s) as its last action — exactly one in the single-choice case, several in the parallel case — with retry-loop enforcement (US7/FR-018); single-child nodes delegate automatically (FR-019).

**Independent Test**: Build a workflow whose node has multiple candidate sub-agents; run it; confirm the node's agent selects a valid candidate as its last action and the run proceeds; confirm invalid/missing selection is retried (retry-loop).

### Tests for User Story 7 (in scope) ⚠️

- [ ] T033 [P] [US7] Unit test for dynamic router (candidate selection + retry-loop) in `<item>/services/__tests__/router.test.js`

### Implementation for User Story 7

- [ ] T034 [P] [US7] Implement `services/engine/router.js` — dynamic routing: candidate selection as last action, retry-loop enforcement, auto-delegate for single child (depends on T013, T024)
- [ ] T035 [US7] Wire router into the scheduler for multi-candidate nodes (depends on T034)
- [ ] T036 [US7] Add validation for `candidateAgents[]` in `validate.js` (must exist, must be resolvable)

**Checkpoint**: US1+2+3+7 — workflows can dynamically route to the appropriate sub-agent at run time.

---

## Phase 7: User Story 8 — Parallel Execution + Ephemeral Agents (Priority: P1)

**Goal**: Independent ready branches + Research-node fan-out run concurrently up to `maxConcurrentSteps` (scheduling semantics, FR-020); nodes are configurable as ephemeral agents (task + tools + skills) when no existing BOS agent matches (FR-021); Research output type fans out sub-agents in parallel (FR-022).

**Independent Test**: Build a workflow with multiple independent Research children; run it; confirm parallel branches execute concurrently (up to the cap) rather than serially. Configure an ephemeral-agent node (task + tools + skills); confirm it executes.

### Tests for User Story 8 (in scope) ⚠️

- [ ] T037 [P] [US8] Unit test for scheduler (ready-set, concurrency cap, research fan-out) in `<item>/services/__tests__/scheduler.test.js`
- [ ] T038 [P] [US8] Unit test for node model (agent source × output type) in `<item>/services/__tests__/node-model.test.js`

### Implementation for User Story 8

- [ ] T039 [P] [US8] Implement `services/engine/scheduler.js` — DAG scheduler: ready-set, `maxConcurrentSteps` cap, research fan-out (depends on T013, T035)
- [ ] T040 [US8] Implement research output type — fan out multiple sub-agents in parallel, collect outputs (depends on T039)
- [ ] T041 [US8] Implement ephemeral agent node execution — task + tools + skills via the delegate route (ADR-1, open item #3: skill scoping) (depends on T039)

**Checkpoint**: US1+2+3+7+8 — workflows can route dynamically AND run branches/sub-agents in parallel, with ephemeral-agent nodes.

---

## Phase 8: User Story 4 + 5 — Graph UI with Active-Step Highlight (Priority: P1)

**Goal**: The app renders each workflow as an interactive graph (nodes + dependency edges, branching support) — FR-012; during a run the graph highlights the active step + live per-step status (pending/running/completed/failed/cancelled) — FR-013. The UI is a parallel surface to the tools (FR-014).

**Independent Test**: Open the app; see the list of workflows; open one to see the graph; run it and confirm the active step highlights + per-step statuses update live; cancel and confirm the graph reflects it.

### Tests for User Story 4+5 (e2e, in scope) ⚠️

- [ ] T042 [P] [US45] e2e test for graph views (list/detail/run + active-step highlight) in `e2e/001-workflow-manager-service-tools.spec.ts`
- [ ] T043 [P] [US45] e2e test for service-stopped banner + empty state in `e2e/001-workflow-manager-service-tools.spec.ts`

### Implementation for User Story 4+5

- [ ] T044 [P] [US45] Build the app facet `app/src/main.tsx` — list view, service pill, stopped banner, empty state (mockup-driven, `mockup.html` is the binding UI contract)
- [ ] T045 [US45] Implement the graph view in `app/src/` — nodes + dependency edges, branching, active-step highlight + live per-step status (FR-012/013) (depends on T044)
- [ ] T046 [US45] Add tool-affordance labels on primary action buttons (`workflow_run`/`workflow_cancel`/`workflow_create`/etc.) per FR-014 (depends on T044)

**Checkpoint**: US1+2+3+7+8+45 — the graph UI works end-to-end, mirroring the tool surface.

---

## Phase 9: User Story 6 — Historical-Run Inspection (Priority: P2)

**Goal**: The app provides a run selector on the detail view; an opened historical run renders each step's final outcome (executed/completed/failed/cancelled/neutral) from the persisted run log + replays the event stream (US6/FR-017). The mockup's historical-runs UI is the binding contract.

**Independent Test**: Run a workflow to completion; reopen it via the run selector; confirm the graph reflects each step's final outcome from the run log and the event stream replays from the log.

### Tests for User Story 6 (e2e, in scope) ⚠️

- [ ] T047 [P] [US6] e2e test for historical-run replay (run selector → graph outcomes + replayed event stream) in `e2e/001-workflow-manager-service-tools.spec.ts`

### Implementation for User Story 6

- [ ] T048 [P] [US6] Implement run selector in the detail view `app/src/` (lists historical runs with id/timestamp/state) (depends on T044, T030)
- [ ] T049 [US6] Implement historical-run graph replay — per-step final outcome from the run log, distinct from live state; replayed event stream (depends on T048)

**Checkpoint**: US1+2+3+7+8+45+6 — historical runs are inspectable in the app, replaying from the persisted log.

---

## Phase 10: User Story 9 — Workflow Manager Skill (Priority: P2)

**Goal**: The app ships a skill instructing the assistant how to build/execute/retrieve workflows via the workflow tools (US9/FR-023).

**Independent Test**: Load the Workflow Manager skill; confirm it instructs building/executing/retrieving workflows via the tools; build a workflow from a natural-language trigger.

### Implementation for User Story 9

- [ ] T050 [P] [US9] Author `skills/workflow-manager/SKILL.md` — instructions for building workflows (ephemeral/Research nodes, candidate agents, required tools/skills), executing, retrieving results, historical-run inspection (depends on T019, T020)
- [ ] T051 [US9] Wire the skill into the item so the assistant can load it (per `target-marketplace-item.md` skill bundling)

**Checkpoint**: US1+2+3+7+8+45+6+9 — the assistant can build/execute/retrieve workflows guided by the bundled skill.

---

## Phase 11: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories + final assembly.

- [ ] T052 [P] Wire `workflow_create` generation path — confirm whether generation uses the delegate route (ephemeral planner) or the workflow-builder agent constructs directly (open item #2); implement chosen path in `engine/generate.js`
- [ ] T053 [P] Resolve ephemeral-node skill scoping (open item #3) — fold skills into task text if the delegate route can't scope per-node skills
- [ ] T054 [P] Confirm + finalize the 5 net-new tool names/schemas (`list`/`read`/`delete`/`run_list`/`run_get`) (open item #4)
- [ ] T055 [P] Document capability grants (`services:read`, `fs:read`) for the app in the item README + settings note (open item #5)
- [ ] T056 Run the full e2e suite + unit tests; fix failures; verify service restart + migration (ADR-5) + cancelled-run settling
- [ ] T057 Final assembly: `app_build` the item; verify `workflow_list` live status + run_id (FR-024), service-stopped banner, and graph views

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup; **BLOCKS all user stories** (the bos-core retirement must land so the service owns the surface without shadowing)
- **User Stories (Phase 3+)**: All depend on Foundational
- **Polish (Final Phase)**: Depends on all user stories

### User Story Dependencies

- **US1 (P1)**: After Foundational — tool surface (no dependencies on other stories)
- **US2 (P1)**: After US1 — fire-and-poll run (depends on executor T024)
- **US3 (P2)**: After US1/US2 — real-VFS persistence + runs (depends on store T015)
- **US7 (P1)**: After US2 — dynamic routing (depends on router T034)
- **US8 (P1)**: After US7 — parallel execution + ephemeral agents (depends on scheduler T039)
- **US4+5 (P1)**: After US1 — graph UI (depends on app T044)
- **US6 (P2)**: After US4+5 — historical-run inspection (depends on T044, T030)
- **US9 (P2)**: After US1 — skill (depends on T019/T020)

### Within Each User Story

- Tests (in scope) MUST be written and FAIL before implementation
- Node model → scheduler/router → executor → handlers
- Core engine before integration
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup + Foundational [P] tasks can run in parallel (bos-core retirement files are disjoint from service files)
- Once Foundational completes, US1/US3/US9 can start in parallel (US1 is the MVP)
- Tests within a story marked [P] run in parallel
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all tests for US1 together:
Task: "Unit test for tool declarations + schemas in tools.test.js"
Task: "Unit test for tool handlers in handlers.test.js"

# Launch the 12-tool surface:
Task: "Implement services/tools.js — 12 tool declarations + JSON schemas"
Task: "Implement workflow_list with live status (FR-024)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (**CRITICAL** — bos-core retirement + service skeleton)
3. Complete Phase 3: User Story 1 (tool surface)
4. **STOP and VALIDATE**: Test US1 independently (12 tools work via the service)
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → Foundation ready (old engine retired, service skeleton + node model + store in place)
2. US1 (tool surface) → Test → MVP
3. US2 (fire-and-poll run) → Test → poll workflows via tools
4. US3 (real-VFS persistence + historical runs) → Test → data survives + replayable
5. US7 (dynamic routing) → Test → workflows adapt mid-run
6. US8 (parallel execution + ephemeral agents) → Test → branches/sub-agents run concurrently
7. US4+5 (graph UI) → Test → graph with active-step highlight
8. US6 (historical-run inspection) → Test → replay from log
9. US9 (skill) → Test → assistant builds/executes via the skill
10. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: US1 (tool surface) → US2 (fire-and-poll)
   - Developer B: US3 (persistence + runs) → US9 (skill)
   - Developer C: US7 (routing) → US8 (parallel)
   - Developer D: US4+5 (graph UI) → US6 (historical replay)
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story is independently completable and testable
- Verify tests fail before implementing
- Commit after each task or logical group
- The bos-core retirement (T005..T011) is on the active feature branch `bos/042-workflow-manager-service`; the marketplace item is built via `app_build` (no branch needed)
- The mockup (`mockup.html`) is the binding UI contract for US4+5 and US6 — the app must match it
- Docs drift (T011) + spec/code drift tracking per constitution
