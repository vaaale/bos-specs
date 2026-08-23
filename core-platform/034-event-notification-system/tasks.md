# Tasks: Event & Notification System

**Input**: spec.md, plan.md, design.md, research.md, data-model.md, contracts/event-api.md (all in this directory)

**Tests**: **REQUIRED** — FR-029 (unit) + FR-030 (e2e), **all self-cleaning** (teardown removes every event, handler registration, preference, and UI state the test creates). E2e file MUST be `e2e/034-event-notification-system.spec.ts` (matches the `buildstudio_run_tests` convention).

**Organization**: Tasks are grouped by user story. The **event kernel engine** is the Foundational phase (blocks all stories) because emit/store/dispatch/query underpin every story.

**Story map** (from spec.md): US1 View & Triage (P1) · US2 Emit (P1) · US3 Headless Processing (P2) · US4 Register UI Handlers (P2) · US5 Click-to-Launch (P2) · US6 Ambiguity (P3) · US7 Configure (P4)

## Format: `[ID] [P?] [Story] Description` — **[P]** = parallelizable · **[Story]** = US label · exact file paths throughout

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: File structure + conventions. No new dependencies (Constitution VII).

- [ ] T001 Create the event kernel module directory and empty entry files: `src/lib/events/` (types.ts, store.ts, stream.ts, kernel.ts, dispatch.ts, api.ts, loopback.ts, migrate-integrations.ts) — all `import "server-only"` except `types.ts` (framework-free, per Constitution II)
- [ ] T002 Create the API route tree `src/app/api/events/` (route.ts, stream/route.ts, [id]/route.ts, [id]/ack/route.ts, [id]/read/route.ts, register/route.ts, unregister/route.ts, preference/route.ts, count/route.ts, handlers/route.ts) as thin adapters delegating to `src/lib/events/api.ts`
- [ ] T003 [P] Create the built-in app scaffold `src/apps/event-viewer/` (manifest.ts + index.tsx) — auto-discovered, NO edit to any app registry
- [ ] T004 [P] Create documentation stubs: `docs/dev/events/` (API reference, handler-registration, idempotency, retry/timeout, namespaces, payload-limits, examples) and a `docs/` usage-events section
- [ ] T005 [P] Ensure `data/events/` is gitignored (Constitution V) — add to `.gitignore` if absent; confirm it is runtime file storage, not a VFS store

**Checkpoint**: Structure in place; `tsc` clean; nothing functional yet.

---

## Phase 2: Foundational — Event Kernel Engine (BLOCKS all user stories)

**Purpose**: The in-process pub/sub kernel that every story rides on. ⚠️ No story work begins until this is done.

**⚠️ TDD**: write the unit tests (T013–T016) to FAIL first, then implement to green.

- [ ] T006 Implement shared framework-free types in `src/lib/events/types.ts` — EventRecord, EventState, HandlerAcknowledgment, HandlerRegistration, HandlerPreference, StreamEvent, and the uniform error-code enum (per `data-model.md` §1–5 and `contracts/event-api.md`)
- [ ] T007 Implement persistence in `src/lib/events/store.ts` — per-month immutable JSONL shards (bodies), per-month mutable state file, **warm in-memory unbounded index** flushed on checkpoint (NOT per-emit — the M1 fix), per-type monotonic sequence allocation, the store mutex, and boot repair (index rebuilt from shards) — using `src/os/atomic-write.ts` `writeFileAtomic`
- [ ] T008 [P] Implement the state-change stream in `src/lib/events/stream.ts` — bounded in-memory ring, subscribe, replay-then-tail from a `streamSeq`, 15s heartbeats (design §3.7)
- [ ] T009 Implement the kernel daemon singleton in `src/lib/events/kernel.ts` — `globalThis` singleton, start/stop, the two-axis state machine (processing pending→processed; read unread→read), completion evaluation (all active handlers settled → processed, with `processedReason`), the checkpoint cadence (30s / 500 changes, outside the emit mutex — R3), and boot re-dispatch trigger — the emit critical path stays a single O(1) shard append + in-memory work
- [ ] T010 Implement the dispatch engine in `src/lib/events/dispatch.ts` — fan-out to active headless handlers (concurrent across handlers, **sequential per handler** FIFO), retry with backoff (1s/5s/30s, 3 attempts → permanently_failed), at-least-once re-dispatch (boot + late registration), the exactly-once-settle `callId`/pair guard (data-model §3/§6.3), and the active-set rule (enabled ∧ owner-running; offline/disabled = not active, never blocks completion)
- [ ] T011 Implement the single public API facade in `src/lib/events/api.ts` — `emit`, `query`, `get`, `ack` (ownership-validated), `markRead`, `register`, `unregister`, `setPreference`, `count`, `listHandlers`, `setEnabled` — the one contract (ADR-2); validate payload ≤1MB (413), type ≤256 + namespace shape, and ack-ownership (FR-022)
- [ ] T012 Wire the HTTP routes in `src/app/api/events/*` to delegate to `api.ts` (thin `fetch` adapters, uniform error envelope) — including the NDJSON `/api/events/stream?since=` route and the `/api/events/count` route

### Unit tests for the kernel (FR-029, self-cleaning) ⚠️ write first, must FAIL

> Each test uses a **temp store root** (store takes an explicit root, R9) and, in `afterEach`, deletes the events/handlers/prefs it created — explicit cleanup calls, not just temp-dir teardown.

- [ ] T013 [P] Unit test store in `tests/events/store.test.ts` — shard write/read, warm index + checkpoint flush, per-type sequence monotonicity, boot repair from shards; self-cleaning
- [ ] T014 [P] Unit test kernel in `tests/events/kernel.test.ts` — emit/query/count/read/get, the two-axis state machine, completion transitions + `processedReason`, no-active-handlers → immediately processed (R8); self-cleaning
- [ ] T015 [P] Unit test dispatch in `tests/events/dispatch.test.ts` — fan-out, per-handler FIFO ordering, retry/backoff → permanently_failed, exactly-once-settle (late ack after timeout), at-least-once re-dispatch (simulated restart + late registration), offline/disabled = not active; self-cleaning
- [ ] T016 [P] Unit test API facade in `tests/events/api.test.ts` — emit validation (413 payload-too-large, 400 invalid-type), ack-ownership (403 ack-forbidden), register namespace validation (403 namespace-not-owned), preference validation; self-cleaning

**Checkpoint**: Kernel is engine-complete and unit-green — can emit, store, dispatch, complete, and query in-process. All stories can now begin.

---

## Phase 3: US1 — View & Triage Notifications (P1) 🎯 MVP

**Goal**: The Event Viewer app + topbar bell give the user visibility: unread list (default), historical toggle, detail, live updates.
**Independent Test**: Emit via the kernel → bell increments → open the viewer → event appears unread → click → marked read → bell decrements → appears in historical.

- [ ] T017 [P] [US1] Implement the Event Viewer manifest + app shell in `src/apps/event-viewer/manifest.ts` (id `event-viewer`, icon Bell, singleton) and `src/apps/event-viewer/index.tsx` (Events/Configuration tabs, topbar row) — match `mockup.html`
- [ ] T018 [US1] Implement the event list in `src/apps/event-viewer/index.tsx` — hand-rolled **windowed/virtualized** list (ADR-5, SC-002), unread-by-default reverse-chronological, source icon/name, summary, type, seq, processing badge (spinner vs ✓), "2/3 handlers" substatus, lazy payload (list shows summary only); initial render <500ms at 100k (NFR-002)
- [ ] T019 [US1] Implement the event detail / **generic (default) view** in `src/apps/event-viewer/index.tsx` — full payload as a key/value table, metadata chips (type/seq/id/time), full processing history (per handler: name, mode badge, attempt, status, result payload, error/retry), matching the mockup's generic renderer
- [ ] T020 [US1] Implement the topbar bell in `src/components/desktop/EventBell.tsx` — live unread count with "99+" cap, opens the viewer, and **replace the old `IntegrationsBadge`** import/usage in the topbar
- [ ] T021 [US1] Implement real-time updates in the viewer (NDJSON client on `/api/events/stream`, reconcile a `count`+`query` snapshot then tail) — live badge flip pending→processed, new-event insertion, "Show historical" toggle, "Mark all as read" (NFR-008/009, FR-028)
- [ ] T022 [US1] E2E (create) `e2e/034-event-notification-system.spec.ts` — US1 flow: emit → bell increments → open viewer → event listed unread → click → read → bell decrements → present in historical; **self-cleaning** (delete the emitted event + restore read state in `afterEach`)

**Checkpoint**: US1 standalone-functional — the user can see and triage events (MVP).

---

## Phase 4: US2 — Emit Events from Any BOS Component (P1)

**Goal**: All real producers (services, apps, assistant) emit through the one public API; payload/namespace validation enforced.
**Independent Test**: A service / the assistant emits a real event through the transport; it appears in the viewer with correct source attribution.

- [ ] T023 [US2] Implement the agent tools in `src/assistant/tools/server/events.ts` — `serverTool`s `emit_event`, `query_events`, `get_event`, `ack_event` (owner-checked), `mark_events_read`, `set_event_preference`, `list_event_handlers`, schemas mirroring `contracts/event-api.md` 1:1 (FR-001)
- [ ] T024 [US2] Implement the worker-thread loopback transport in `src/lib/events/loopback.ts` — base URL `http://127.0.0.1:${process.env.PORT}` (R2, **[verify in implement]** the exact Supervisor preview env var), headless-auth header, and typed `emit`/`ack`/`query` client wrappers
- [ ] T025 [US2] Re-point the GSuite emitters to `api.emit` in `src/app/api/integrations/[id]/services/[serviceId]/poll/route.ts` and the `webhook/test` + `webhooks/...` routes (replace `emitNotification`)
- [ ] T026 [US2] Re-point `src/lib/integrations/scheduler/jobs.ts` and `src/lib/integrations/services/telegram/notification-handler.ts` to `api.emit`
- [ ] T027 [P] [US2] Unit test `tests/events/emit-surfaces.test.ts` — payload-size 1MB rejection, namespace/length validation, source snapshot capture for app/service/assistant emitters; self-cleaning

**Checkpoint**: Every production emitter goes through the one API; the assistant can emit/query as a tool.

---

## Phase 5: US3 — Automatic Headless Event Processing (P2)

**Goal**: Real services register headless handlers and are auto-dispatched by the kernel; acks over loopback; legacy inbox migrated.
**Independent Test**: A service registers a headless handler, an event is emitted, the handler is invoked, acks (with a result), and the event transitions to processed; a failing handler retries then permanently-fails without blocking.

- [ ] T028 [US3] Add the worker-IPC message types `event_dispatch` and `handler_declare` (with `callId`) in `src/core/service/types.ts`
- [ ] T029 [US3] Implement `dispatchEventToWorker` + `handler_declare` handling in `src/core/service/workerIpc.ts` and `src/core/service/ServiceManager.ts` — route dispatched events to the worker, ingest runtime-declared handlers into the kernel registry, and mark handlers **not active** on worker stop/crash (ADR-3)
- [ ] T030 [US3] Wire the service ack path through the loopback `POST /api/events/:id/ack` with ownership validation (FR-022) — a service acks only its own handler
- [ ] T031 [US3] Retire the legacy notifications store in `src/lib/integrations/notifications/` — remove the store + route once re-pointed (US2) and migration (T032) confirm no `emitNotification` **writes** remain (grep to verify; reads may remain transiently)
- [ ] T032 [US3] Implement the legacy-inbox migration in `src/lib/events/migrate-integrations.ts` — idempotent, guarded by marker file `data/events/.migrated-integrations`, stable derived ids `legacy-<index>` (R6), emits read-only-sourced legacy events into the kernel
- [ ] T033 [US3] Wire boot order in `src/instrumentation.ts` — `startAll()` services → run migration → `startEventKernel()` (so the first dispatch cycle sees migrated events)
- [ ] T034 [P] [US3] Unit test `tests/events/headless.test.ts` — end-to-end kernel+dispatch headless flow (register→emit→invoke→ack→processed), retry→permanent-fail non-blocking, offline/disabled = not active, at-least-once re-dispatch on restart, migration idempotence (marker + stable ids); self-cleaning
- [ ] T035 [US3] E2E (append to `e2e/034-event-notification-system.spec.ts`) — a registered headless handler processes an emitted event and completes it; a failing handler retries then permanently-fails (event still completes); **self-cleaning**

**Checkpoint**: Headless automation works end-to-end from real services; legacy inbox is migrated.

---

## Phase 6: US4 — Register UI Handlers (P2)

**Goal**: Apps declare UI handlers (manifest) that surface into the registry; namespace/grant validation enforced.
**Independent Test**: An app declares a UI handler for a type in its manifest; after boot the registration is present in the registry for that type.

- [ ] T036 [US4] Extend the manifest types in `src/os/types.ts` — `AppManifest.eventHandlers?` (UI: `{ id, type, appId, displayName, description?, icon? }[]`) and `eventNamespaces?: string[]` (granted prefix list, FR-023)
- [ ] T037 [US4] Surface installed apps' UI `eventHandlers` into the kernel registry at boot in `src/lib/apps/store.ts` — built-in apps and installed (iframe) apps; validate against owned root `com.bos.<appId>.*` + granted namespaces
- [ ] T038 [P] [US4] Unit test `tests/events/ui-registration.test.ts` — UI handler registration from manifest, granted-namespace acceptance, non-owned rejection (403); self-cleaning

**Checkpoint**: UI handlers are declared and discoverable in the registry.

---

## Phase 7: US5 — Click an Event to Launch Its UI Handler (P2)

**Goal**: Clicking an event marks it read and launches its UI handler (or the generic view when none is registered).
**Independent Test**: Click an event with a UI handler → handler app launches with event params; click an event with none → generic view.

- [ ] T039 [US5] Implement click-to-launch routing in `src/apps/event-viewer/index.tsx` — on click: mark read, resolve the single/default UI handler, `launch(appId, { event: { id, type, seq } })` (R4 — pass the **id**, not the payload)
- [ ] T040 [US5] Implement the generic-view fallback in `src/apps/event-viewer/index.tsx` — when no UI handler is registered, show the "No UI handler registered" banner + the default structured renderer (FR-017)
- [ ] T041 [US5] Implement marketplace (iframe) param delivery — the app embed passes `{ id, type, seq }` to the iframe launch channel **[verify in implement]** the existing embed param mechanism; the iframe reads the params and `GET /api/events/:id` for the full event
- [ ] T042 [US5] E2E (append to `e2e/034-event-notification-system.spec.ts`) — click → UI handler launches + event read; click (no handler) → generic view shown; **self-cleaning**

**Checkpoint**: Clicking an event is actionable — deep-link to handler or generic view.

---

## Phase 8: US6 — Resolve UI Handler Ambiguity (P3)

**Goal**: Multiple UI handlers for a type → selection dialog; "Always use" sets a persistent default.
**Independent Test**: Register two UI handlers for a type, click an event → dialog with both + "Always use"; choosing with the checkbox sets the default so the next click skips the dialog.

- [ ] T043 [US6] Implement the "Open with…" selection dialog in `src/apps/event-viewer/index.tsx` — list matching UI handlers (icon, name, description), radio select, "Always open with this app for this event type" checkbox (matches mockup)
- [ ] T044 [US6] Wire `setPreference` in the viewer → `POST /api/events/preference` (set/clear default UI handler), persisted across sessions (FR-010/027); single/default resolution consults the preference first
- [ ] T045 [US6] E2E (append to `e2e/034-event-notification-system.spec.ts`) — dialog appears for multi-handler type; select + "Always use" sets default; subsequent click skips dialog; clear default re-enables prompt; **self-cleaning**

**Checkpoint**: Ambiguity resolved in one interaction; preference sticks.

---

## Phase 9: US7 — Configure Event Handlers (P4)

**Goal**: Configuration tab to view all handlers, enable/disable headless, set/clear UI defaults, see failure counts.
**Independent Test**: Open config → see grouped handlers; disable a headless handler → new events of that type no longer invoke it and complete without it; set/clear a UI default.

- [ ] T046 [US7] Implement the Configuration tab in `src/apps/event-viewer/index.tsx` — handlers grouped by event type, headless rows (enable/disable toggle, failure badge, timeout), UI rows (set/clear default), matching the mockup config view (FR-018)
- [ ] T047 [US7] Implement `listHandlers` + `setEnabled` — `GET /api/events/handlers` (grouped registry with `recentFailures`) and `POST /api/events/handlers` (owner-checked enable/disable, headless only); disabling re-evaluates pending events (data-model §6.1)
- [ ] T048 [US7] E2E (append to `e2e/034-event-notification-system.spec.ts`) — config lists handlers; disabling a headless handler stops invocation + unblocks completion; re-enable resumes; set/clear UI default; **self-cleaning**

**Checkpoint**: Full user control over handler routing.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Docs, performance proof, the GSuite handler payoff, full validation.

- [ ] T049 [P] Write developer documentation in `docs/dev/events/` — API reference (all 10 ops + stream), headless/UI handler registration, **idempotency requirement**, retry/timeout semantics, namespace/grant conventions, payload limits, and working integration examples (FR-026a)
- [ ] T050 [P] Write user documentation in the `docs/` usage-events section — viewing pending/historical, processing-status badges, configuration, enabling/disabling handlers, interpreting processing history and failures (FR-026b)
- [ ] T051 [P] Add a "How it works" section to the architecture overview doc and an "Add an event handler" recipe to the extending-BOS doc (FR-026 / Constitution VI)
- [ ] T052 [P] Performance test `tests/events/perf.test.ts` — seed 100k events, assert emit p99 <100ms (NFR-001), initial list query <500ms (NFR-002), dispatch begin <500ms (SC-006); **self-cleaning** (wipe the temp root)
- [ ] T053 Implement the GSuite email UI handler (R5) — declare `com.bos.gsuite.email.received` in the GSuite app manifest; on launch with a mail-id param, open the existing email-detail view (or render a thin detail from the payload); this is the concrete migration payoff
- [ ] T054 Run the full suite (unit + `e2e/034-event-notification-system.spec.ts`) and confirm **all tests self-clean** — no residual events/handlers/prefs in the store after the run (FR-029/030)
- [ ] T055 Final review — confirm no `emitNotification` write path remains, no `package.json`/lockfile/build-config change (Constitution VII), feature-branch-only changes (IV), and re-run the Constitution gate

**Checkpoint**: Feature complete, documented, tested, and Constitution-compliant.

---

## Dependencies & Execution Order

### Phase dependencies
- **Setup (P1)**: none — start immediately.
- **Foundational (P2)**: depends on Setup — **BLOCKS all stories**.
- **US1 / US2 (P1)**: depend on Foundational. US2's re-pointing (T025/026) is independent of US1's UI; they can proceed in parallel once the kernel is green.
- **US3 (P2)**: depends on Foundational + US2 (loopback transport T024, re-pointed emitters) for the service ack + migration.
- **US4 (P2)**: depends on Foundational only (manifest types + registry) — independent of US1/US2/US3.
- **US5 (P2)**: depends on US4 (UI handlers must be in the registry) + US1 (viewer shell/detail).
- **US6 (P3)**: depends on US5 (click routing) + US4.
- **US7 (P4)**: depends on US3 (headless active-set semantics) + US4 (UI defaults) + US1 (viewer shell).
- **Polish (P10)**: depends on all stories.

### User-story independence
- US1 is the **MVP** (viewer + bell) — independently demoable once the kernel + a test emitter exist.
- US2, US4 are independent of each other (different files: emitters/tools vs. manifest/registry).
- US3, US5, US6, US7 build on earlier stories but each adds a self-contained, independently testable increment.

### Within each story
- Unit/e2e tests are written to FAIL before the implementation they target (TDD) — except e2e scenarios that append to a single file (sequential, no [P]).
- Types/models → engine/services → routes → UI → integration.
- Commit after each task or logical group.

### Parallel opportunities
- Setup: T003, T004, T005 in parallel.
- Foundational: T008 (stream) parallel with T006–T007 once types exist; T013–T016 (unit tests) in parallel.
- US1: T017 parallel with the kernel-finalization; T018/T019/T021 sequential (same `index.tsx` file).
- US2: T023, T024, T025, T026, T027 largely parallel (different files).
- US3: T028 → T029 → T030 sequential (IPC → manager → ack); T031/T032/T033 near-parallel; T034 parallel.
- US4: T036 → T037; T038 parallel.
- Polish: T049, T050, T051, T052 in parallel.

> Note: tasks marked [P] touch **different files**. Multiple e2e tasks (T022/T035/T042/T045/T048) all edit the single file `e2e/034-event-notification-system.spec.ts`, so they are **sequential** despite being in different story phases.

---

## Implementation Strategy

**Full-scope delivery — all user stories, one pass.**

1. Setup (P1) → Foundational kernel (P2) — green unit tests.
2. US1 + US2 (P1 pair) — viewer, bell, emission surfaces, agent tools.
3. US3 (P2) — headless automation, worker-IPC dispatch, migration.
4. US4 + US5 + US6 (P2/P3) — UI handler declaration, click-to-launch, ambiguity resolution.
5. US7 (P4) — configuration.
6. Polish (P10): docs, perf proof, GSuite handler, full self-cleaning suite, Constitution re-check.

No stopping between stories. All 55 tasks are delivered in sequence (respecting dependencies), with checkpoints for validation at each phase boundary but no delivery gates. The feature is not complete until Phase 10 (T054–T055) passes.
