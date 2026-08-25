# Tasks: Workflow Event Triggers

**Input**: Design documents from `/Specs/user-specs/workflow-manager/002-workflow-event-triggers/`

**Prerequisites**: plan.md (required), spec.md (required), design.md (required)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing. All paths are relative to the item root (`data/user-apps/items/workflows/`).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)

---

## Phase 1: Setup (Read the Existing Item)

**Purpose**: Understand the baseline item structure before adding the trigger engine.

- [ ] T001 Read the existing `services/index.js`, `services/handlers.js`, `services/tools.js`, `services/engine/store.js`, `services/runs.js`, and `services/service.json` to understand the current item layout, the tool handler dispatch pattern, the run entity shape, and the VFS bridge usage.
- [ ] T002 Read the existing `app/src/main.tsx` and its components to understand the app's view structure, the `window.__bos` SDK usage pattern, and where the Triggers panel + ⚡ badges will slot in.

---

## Phase 2: Foundational (Trigger Engine Core)

**Purpose**: Core infrastructure that MUST be complete before any user story can be implemented. The trigger engine, the loopback event client, and the service boot wiring.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T003 [US1] Create `services/triggers.js` — the trigger engine module with:
  - `subscriptionIndex`: `Map<eventType, Set<workflowId>>` (in-memory, derived)
  - `recompute()`: read all workflows from VFS (via loopback `/api/fs`), build the index from every trigger's `eventType`
  - `sync()`: diff the recomputed union against currently-registered handlerIds; for newly-referenced types emit `handler_declare` IPC (`{callId, handlerId: "wft_"+eventType, eventType, displayName, description, timeoutMs:30000}`); for dropped types call loopback `POST /api/events/unregister` with `{handlerId, ownerId}`
  - `onDispatch({callId, eventId, handlerId, record})`: look up `subscriptionIndex[record.type]`, for each workflowId call `startRun(workflowId, {source:"event", trigger:{eventId:eventId, eventType:record.type}, input:{event:{id:record.id, type:record.type, payload:record.payload, ts:record.ts, summary:record.summary, source:record.source}}})`, collect per-workflow outcomes (runId or error), then ack via loopback `POST /api/events/:eventId/ack` with `{handlerId, callerId:<serviceId>, callId, result:{started:[...], failed:[...]}}`
  - `validateEventType(type)`: local ~10-line check — well-formed dot-namespaced (`com.bos.*` prefix) AND within the `eventNamespaces` grant (for `com.bos.*` grant: just check `com.bos.` prefix). Returns `{valid, error}`.
  - Export: `{recompute, sync, onDispatch, validateEventType, subscriptionIndex}`

- [ ] T004 [P] [US1] Create `services/events.js` — the item-local loopback event-API client (the service cannot import BOS's `src/lib/events/loopback.ts`):
  - `unregister(handlerId, ownerId)`: `POST http://127.0.0.1:<PORT>/api/events/unregister` body `{handlerId, ownerId}`
  - `ack(eventId, {handlerId, result, callerId, callId})`: `POST http://127.0.0.1:<PORT>/api/events/:eventId/ack` body `{handlerId, result, callerId, callId}`
  - Reuse the baseline's origin resolution pattern (read `process.env.PORT` for the loopback origin)
  - Export: `{unregister, ack}`

- [ ] T005 [P] [US1] Modify `services/service.json` — add `"eventNamespaces": ["com.bos.*"]` to the manifest (the subscription grant, ADR-1). Everything else unchanged.

- [ ] T006 [US1] Modify `services/index.js` — wire the trigger engine into the service lifecycle:
  - On boot (after `initialized` message handled, after the existing tool declarations): call `triggers.recompute()` then `triggers.sync()` to re-subscribe from persisted config
  - Add an `event_dispatch` message handler in the worker's message switch: on receiving `{type:"event_dispatch", payload}`, call `triggers.onDispatch(payload)`
  - After any tool handler that mutates workflows (create/modify/delete), call `triggers.sync()` (the handlers.js changes in US2 will trigger this, but the wiring point is here)

**Checkpoint**: Foundation ready — the service can subscribe to event types, receive dispatches, and fan out runs. User story implementation can now begin.

---

## Phase 3: User Story 1 - Auto-Run on Trigger Event (Priority: P1) 🎯 MVP

**Goal**: When a matching event is published, the workflow's run starts automatically with the event payload injected and provenance recorded.

**Independent Test**: Configure a workflow with a trigger on a test event type; emit that event type via `emit_event`; confirm a run starts, the run has `startSource:"event"` + `trigger` + `input.event`, and the run is inspectable.

### Implementation for User Story 1

- [ ] T007 [US1] Modify `services/runs.js` — extend the run entity for event-triggered runs:
  - Add `startSource: "event" | "manual"` field (default `"manual"` for existing runs)
  - Add `trigger: {eventId, eventType}` field (present only when `startSource === "event"`)
  - Add `input: {event: {id, type, payload, ts, summary, source}}` field (present only when event-triggered)
  - For event-triggered runs, seed the run's event log with a first entry: `{type:"triggered", eventType, eventId, ts: Date.now()}`
  - `startRun(workflowId, options)`: accept `{source, trigger, input}` in the options; when `source === "event"`, populate `startSource`, `trigger`, `input` on the run record before starting execution
  - `run_list` / `run_get` responses include `startSource` and (when event) `trigger.eventType`

- [ ] T008 [US1] Verify the `onDispatch` → `startRun` → ack path end-to-end: the trigger engine's `onDispatch` calls `startRun` with the correct options (source, trigger, input.event), `startRun` returns a `runId` promptly (fire-and-poll), and the ack is sent with the correct `callerId` and `callId`. The run continues asynchronously after the ack.

**Checkpoint**: US1 is fully functional — emitting a matching event auto-starts a run with provenance.

---

## Phase 4: User Story 2 - Triggers Configurable via Tools (Priority: P1)

**Goal**: The workflow tools create/modify/read triggers, and the service re-syncs subscriptions on every mutation.

**Independent Test**: Use `workflow_create` with a `triggers` array; confirm the trigger is persisted in the VFS workflow JSON; use `workflow_read` and confirm triggers are returned; use `workflow_modify` to change the event type; confirm the service re-subscribes (new type declared, old type unregistered).

### Implementation for User Story 2

- [ ] T009 [P] [US2] Modify `services/engine/store.js` — persist the `triggers` array in the workflow JSON:
  - The workflow JSON schema gains `"triggers": [{id, eventType}]` (an array of trigger objects)
  - On `create`/`modify`, validate each trigger: `eventType` must be a non-empty string; call `triggers.validateEventType(eventType)` and reject with a clear error if invalid (FR-014)
  - On `delete`, the workflow (and its triggers) are removed; the caller (handlers.js) triggers `sync()`

- [ ] T010 [US2] Modify `services/handlers.js` — wire trigger authoring through the tool handlers:
  - `workflow_create` handler: accept `triggers` in the config body; validate; persist; call `triggers.sync()`
  - `workflow_modify` handler: accept `triggers` in the JSON-merge patch (add/edit/remove individual triggers); validate new/changed types; persist; call `triggers.sync()`
  - `workflow_read` handler: return the full workflow JSON including `triggers` (satisfies FR-008 "a read/inspection tool MUST return a workflow's triggers")
  - `workflow_delete` handler: after deletion, call `triggers.sync()` (to unregister types no longer referenced)
  - `workflow_run` handler: when starting a run manually, pass `source: "manual"` (no trigger/input.event) — distinct from event-triggered runs

- [ ] T011 [US2] Modify `services/tools.js` — extend the JSON schemas for the tool surface:
  - `workflow_create` schema: add `triggers` to the config body (array of `{id: string, eventType: string}`)
  - `workflow_modify` schema: add `triggers` to the patch body (array, or `null` to clear)
  - `workflow_read` response: document that `triggers` is returned
  - `workflow_run_list` response: add `startSource` and `eventType` (when event) per run
  - `workflow_run_get` response: add `trigger: {eventId, eventType}` and `input.event` (when event)
  - `workflow_status` response: add `startSource` and `eventType` (when event) for the running run

**Checkpoint**: US2 is fully functional — triggers are authorable via tools and subscriptions stay current.

---

## Phase 5: User Story 3 - App Triggers Panel + Provenance UI (Priority: P2)

**Goal**: The app's workflow detail view shows a Triggers panel (add/edit/remove/list) and the run history + live run show ⚡ provenance for event-triggered runs.

**Independent Test**: Open a workflow in the app; add a trigger via the panel; confirm it persists on reload; emit a matching event; confirm the run shows a ⚡ badge in history and the "Started by event" banner.

### Implementation for User Story 3

- [ ] T012 [P] [US3] Modify `app/src/main.tsx` (and/or add a `TriggersPanel` component) — implement the Triggers panel in the workflow detail view, per the mockup:
  - A "Triggers" section (below the graph, amber `zap` header + active-count chip)
  - Trigger list: each row shows the `eventType` in mono with a small icon + a "listening" status dot
  - "Add trigger" button → reveals an inline input (placeholder `com.bos.assistant.task.done`) + Add/Cancel buttons
  - Per-trigger: hover-revealed edit (pencil) + remove (trash) buttons; edit switches to inline input with Save/Cancel
  - Empty state: "No event triggers" dashed card with an Add button
  - All mutations go through `window.__bos` → `workflow_modify` (the mockup's `workflow_trigger_add` hint is a UI affordance name, not a tool)
  - List view: workflow cards show a "N triggers" / "manual only" indicator

- [ ] T013 [P] [US3] Modify `app/src/main.tsx` (and/or the run-history component) — implement ⚡ provenance rendering, per the mockup:
  - Run history rows: event-triggered runs show an amber ⚡ chip + the `eventType` in small mono text; manual runs show "manual"
  - Live run header: an "event" chip + `⌁ <type>` when the run is event-triggered
  - Historical replay panel: a "Started by event `<type>`" banner (amber) shown only for event-triggered runs
  - Event stream: the first line for event-triggered runs is `⚡ triggered · <type>` (from the seeded log entry)
  - All data comes from `workflow_run_list` / `workflow_run_get` / `workflow_status` (US4 provenance fields)

**Checkpoint**: US3 is fully functional — the app configures and displays triggers and provenance.

---

## Phase 6: User Story 4 - Run Provenance (Priority: P2)

**Goal**: Each run records its start source and, when event-triggered, the triggering event's id + type. Provenance is surfaced through run_list/run_get tools and the app.

**Independent Test**: Fire a workflow via an event; open the resulting run; confirm `startSource:"event"`, `trigger:{eventId, eventType}`, and the seeded `triggered` log entry. Confirm a manual run has `startSource:"manual"` and no trigger.

### Implementation for User Story 4

- [ ] T014 [P] [US4] Verify run provenance is complete and consistent:
  - `runs.js`: `startSource`, `trigger`, `input.event` are persisted on the run JSON in VFS
  - `runs.js`: the seeded `triggered` first log entry is written for event runs
  - `tools.js`: `workflow_run_list` returns `startSource` (+ `eventType` when event); `workflow_run_get` returns `trigger` + `input.event` + the seeded entry
  - `tools.js`: `workflow_status` (live) returns `startSource` (+ `eventType`) for the running run
  - Manual runs: `startSource:"manual"`, no `trigger`, no `input.event` (US4 AC2)

**Checkpoint**: US4 is fully functional — provenance is recorded and inspectable via tools and the app.

---

## Phase 7: User Story 5 - Bounded Behavior (Priority: P2)

**Goal**: Trigger matching and firing behavior is predictable and safe under real event traffic: re-entrancy is always-start-a-new-run, bursts don't crash, errors are logged.

**Independent Test**: Publish the same event type multiple times in quick succession; confirm each starts a new independent run (no coalescing), no crash. Stop the service; confirm no dispatch. Restart; confirm re-subscription.

### Implementation for User Story 5

- [ ] T015 [P] [US5] Verify re-entrancy + error handling in `triggers.js`:
  - `onDispatch`: no coalescing/queuing — each matching event starts fresh runs (FR-013, always-start-a-new-run)
  - `onDispatch`: if `startRun` throws for a specific workflow, catch it, log the error (NFR-001), record it in the `failed` array of the ack result, and continue to the next workflow (one workflow's failure doesn't block others)
  - `onDispatch`: if the entire `onDispatch` handler throws (unexpected), the 034 dispatch engine's 3× retry + `permanently_failed` path handles it — the service doesn't need special handling
  - `recompute`/`sync`: if a VFS read fails, log the error and return the previous index (don't lose the subscription set on a transient error)
  - Stop: no explicit unregister needed (034 active-set rule, FR-010) — verify the service doesn't call `events.unregister` on stop
  - Restart: `recompute()` + `sync()` re-subscribes from persisted config (NFR-002)

- [ ] T016 [P] [US5] Verify burst safety (NFR-004):
  - `onDispatch` does bounded work per event (iterate the matching workflowIds, call `startRun` for each, ack) — no unbounded fan-out
  - The 034 per-handler FIFO queue serializes dispatches to the same handler (same event type), so a burst on one type is processed sequentially per handler
  - Different event types have different handlers → parallel dispatch (no cross-type blocking)

**Checkpoint**: US5 is fully functional — the trigger engine is safe and predictable under real traffic.

---

## Phase 8: Polish & E2E

**Purpose**: End-to-end validation and final integration.

- [ ] T017 [P] E2E test: create a workflow with a trigger via `workflow_create` (tool), emit a matching event via `emit_event`, verify the run starts with `startSource:"event"` + `trigger` + `input.event`, verify `workflow_run_get` returns the provenance, verify `workflow_run_list` shows the ⚡/event distinction. **Negative path (SC-002):** also emit an event of a *different* type that no trigger references and verify zero runs start (no false positives). Use a real or test-emitted event type (NOT `com.bos.assistant.task.done` which is illustrative only — use `com.bos.gitops.conflict.escalated` or emit a custom test type within the `com.bos.*` grant).
- [ ] T018 [P] E2E test: modify a workflow's trigger event type via `workflow_modify`, verify the service re-subscribes (new type fires, old type doesn't). Verify `workflow_read` returns the updated triggers.
- [ ] T019 [P] E2E test: stop the service, emit a matching event, verify no run starts. Restart the service, verify it re-subscribes from persisted config. Emit again, verify the run starts.
- [ ] T020 Verify the app UI renders the Triggers panel and ⚡ provenance correctly (visual check against the mockup).

**Checkpoint**: All user stories are independently functional and the E2E tests pass.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup — BLOCKS all user stories
  - T003 (triggers.js) and T004 (events.js) can run in parallel
  - T005 (service.json) can run in parallel with T003/T004
  - T006 (index.js wiring) depends on T003 + T004
- **US1 (Phase 3)**: Depends on Phase 2 (triggers.js, events.js, index.js wiring)
  - T007 (runs.js) can start as soon as Phase 2 is done
  - T008 (verification) depends on T007
- **US2 (Phase 4)**: Depends on Phase 2 (for `sync()`) + T007 (for `startRun` options)
  - T009 (store.js) and T011 (tools.js) can run in parallel
  - T010 (handlers.js) depends on T009 + T011
- **US3 (Phase 5)**: Depends on US2 (tools must work for the app to call them)
  - T012 (TriggersPanel) and T013 (⚡ UI) can run in parallel
- **US4 (Phase 6)**: Depends on T007 (runs.js provenance) — can run in parallel with US3
  - T014 (verification) depends on T007 + T011
- **US5 (Phase 7)**: Depends on Phase 2 (triggers.js) — can run in parallel with US3/US4
  - T015 and T016 can run in parallel
- **Polish (Phase 8)**: Depends on all user stories

### Parallel Opportunities

- T003, T004, T005 can all run in parallel (Phase 2)
- T009, T011 can run in parallel (US2)
- T012, T013 can run in parallel (US3)
- T014 (US4) can run in parallel with T012/T013 (US3)
- T015, T016 can run in parallel (US5)
- T017, T018, T019 can run in parallel (Polish E2E)

### Implementation Strategy

This is a **single-item build** — the Developer writes the complete updated item into a staging directory and `app_build` installs it. The phases above describe the logical order of implementation within that single build; they do not imply separate delegations. The Developer should:
1. Read the existing item (Phase 1)
2. Implement the foundational trigger engine (Phase 2)
3. Extend runs/handlers/tools for provenance + trigger authoring (Phases 3–4)
4. Add the app UI (Phase 5)
5. Verify bounded behavior (Phase 7)
6. Wire it all in `index.js` (Phase 2, T006, done last within Phase 2 since it depends on the modules)

---

## Notes

- All paths are relative to `data/user-apps/items/workflows/` (the item root)
- The service is unbundled JavaScript — no `@/` imports, no TypeScript, no `npm install`
- The app facet is TypeScript/React, bundled by esbuild via `app_build`
- The item's `service.json` is the ONLY manifest change (add `eventNamespaces`)
- No BOS-source files are modified
- `app_build` installs the full item (app + service facets) in one call
- The Developer writes the updated item into a **fresh staging directory** (never edits `data/user-apps/items/workflows/` in place) and reports the staging directory path; Build Studio calls `app_build` with that path
