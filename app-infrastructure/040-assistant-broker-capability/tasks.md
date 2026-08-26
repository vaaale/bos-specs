# Tasks: Assistant Broker Capability

**Input**: [spec.md](./spec.md), [design.md](./design.md), [plan.md](./plan.md)

**Organization**: Tasks grouped by user story so each is independently implementable and testable. The feature is a focused bos-core change — 6 source files (5 modify, 1 create) + 1 companion dev doc. No new dependency, service, or container.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no shared-file dependency)
- **[Story]**: US1–US4 from spec.md
- Exact file paths throughout

---

## Phase 1: Foundational (Blocking Prerequisites)

**Purpose**: The capability plumbing + broker module core that every user story depends on. No story work begins until this phase is done.

- [ ] T001 [P] [Found] Add `"assistant"` to the `AppCapability` union in `src/os/types.ts` (with a comment mirroring `services:read`).
- [ ] T002 [P] [Found] Add `"assistant"` to `VALID_CAPS` in `src/app/api/apps/[id]/capabilities/route.ts`.
- [ ] T003 [Found] Create `src/components/apps/assistant-broker.ts` — the module-level `Map<appId, AppBroker>` registry with `getBroker(appId)`, a per-`AppBroker` run table `Map<runId, RunSession>` (`{events, lowWater, highWater, finished, abort}`), refcounted `windowCount` (increment/decrement + `dispose(appId)` on last-window unmount), the owned-run set, and the bounded ring buffer (~2000 events / ~1 MB default, `lowWater` = min attached-child cursor). No server-only imports.
- [ ] T004 [Found] In `src/components/apps/assistant-broker.ts`, implement the **live server tail**: one `fetch(.../runs/[runId]/events?since=N)` + `ReadableStream` reader loop per run (the `run-client.ts` `attachToRun` shape), appending each parsed `RunEvent` to the session buffer and pushing it to the child via a new unsolicited `{__bos_event: true, runId, event}` `postMessage`. Skip `ping` (no `seq`).
- [ ] T005 [Found] In `src/components/apps/assistant-broker.ts`, implement **replay + authoritative re-fetch**: on attach, serve buffered events with `seq > childCursor` in order, then continue live; if `childCursor < lowWater`, tear down the tail and re-open the server stream at `?since=childCursor`. Enforce single-tail-per-run (no second concurrent subscription).
- [ ] T006 [Found] In `src/components/apps/IframeApp.tsx`, add `assistant:*` entries to `CAP_FOR_METHOD` (all → `"assistant"`); in `handleMessage`, after the existing synchronous cap gate, route `assistant:*` methods to `getBroker(appId)` (passing the window as push target) instead of the stateless `dispatch()`; wire refcount increment on window mount and dispose on last-window unmount.

**Checkpoint**: The capability is typed, validated, and gated; the broker module owns per-run tails and can push events to a child iframe. Stories can now build on it.

---

## Phase 2: User Story 1 - Marketplace app embeds working agentic chat (Priority: P1) 🎯 MVP

**Goal**: An opaque-origin app can start a run and receive a complete streamed assistant reply through the broker.

**Independent Test**: Grant `assistant` to a marketplace-origin app. Open it, send a chat message, verify a streamed reply appears (no direct `/api/assistant` fetch).

### Implementation

- [ ] T010 [P] [US1] In `src/components/apps/assistant-broker.ts`, implement the broker methods `listAgents` (relay `GET /api/assistant/agent` body per ADR-4), `startRun` (`POST /api/assistant/runs` → `{runId}`; 409 → `{error, activeRunId}`; record the runId in the owned-run set; open the tail), `activeRun` (`GET /api/assistant/runs?conversationId=`), and `cancelRun` (`POST /api/assistant/runs/[runId]/cancel`). Enforce owned-run checks on `startRun`-driven operations (ADR-3).
- [ ] T011 [US1] In `src/components/apps/assistant-broker.ts`, implement `attach(runId, since, windowId)` — the events-attach entry that opens/uses the tail and begins pushing `__bos_event` to that window; return a terminal signal when `run_finished` is delivered (design.md §6).
- [ ] T012 [P] [US1] In `src/lib/iframe-sdk/index.ts`, add the `window.__bos.assistant` group + `__bos_event` listener (returns unsubscribe): `listAgents()`, `startRun({conversationId, agentId, message, surfaceTools?, attachments?}) → {runId}`, `onRunEvent(runId, cb) → unsubscribe` (tracks `lastSeq` per run, calls `attach` on the cursor, dispatches pushed events), `getActiveRun(conversationId)`, `cancelRun(runId)`. Keep dependency-free.
- [ ] T013 [US1] Add the `assistant` row to `ALL_CAPABILITIES` in `src/components/apps/settings/AppsTab.tsx`, rendered **conditionally** — only for apps whose manifest declares `"assistant"` (ADR-6). Reuse the existing `toggleCap`/checkbox pattern.
- [ ] T014 [US1] In `src/app/api/apps/[id]/capabilities/route.ts`, on `PUT` reject `assistant` (drop it with a warning) if the app's manifest does not declare it — server-side declaration gating (ADR-6).

**Checkpoint**: US1 is fully functional — a marketplace app streams a complete assistant reply end-to-end through the broker, with grant/revoke wired.

---

## Phase 3: User Story 2 - Marketplace app participates in frontend tool calls (Priority: P1)

**Goal**: The app receives `tool_call` (frontend) events, executes them locally, and posts results back so the run continues.

**Independent Test**: In a marketplace app with an `agentic_editor_*` surface tool, ask the assistant to "add a paragraph"; verify the tool call arrives, executes, the result posts, and the run completes.

### Implementation

- [ ] T020 [US2] In `src/components/apps/assistant-broker.ts`, implement `toolResult(runId, callId, result)` → `POST /api/assistant/runs/[runId]/tool-results` → `{claimed}` (unknown/finished run → 404/no-op). Verify `tool_call` events with `execution: "frontend"` (name, callId, args) flow through the event buffer to the child unmodified (design.md §3.6).
- [ ] T021 [US2] In `src/lib/iframe-sdk/index.ts`, add `postToolResult(runId, callId, result)` to the `assistant` group (maps to `assistant:tool-result`).
- [ ] T022 [US2] Confirm surface tools ride the `startRun` body via unchanged `startAssistantRun` merge (FR-009) — no new broker method needed; verify `surfaceTools` is passed through `startRun` → `POST /api/assistant/runs`.

**Checkpoint**: US1 + US2 both work — the app is fully agentic (chat + tool round-trips) through the broker.

---

## Phase 4: User Story 3 - Capability is grantable and revocable (Priority: P2)

**Goal**: The `assistant` capability has a correct grant/revoke lifecycle, declaration-gated per ADR-6, with a fast denial when ungranted.

**Independent Test**: In Settings → Apps, toggle `assistant` off for a declaring app; verify broker calls reject; toggle back on; verify they succeed.

### Implementation

- [ ] T030 [P] [US3] Verify the synchronous cap-gate denial path in `src/components/apps/IframeApp.tsx`: an ungranted `assistant:*` call rejects with `Capability "assistant" not granted` before dispatch (SC-002, ≤100 ms). Confirm the gate covers all six methods via `CAP_FOR_METHOD`.
- [ ] T031 [P] [US3] Verify mid-run revoke behavior: revoking the capability does not tear down an in-flight tail (registry decoupled from the cap closure); in-flight deliveries continue, new `startRun` calls reject (design.md §3.7, spec edge case).
- [ ] T032 [US3] Verify declaration-gating end-to-end: an app whose manifest omits `assistant` shows no Settings row and is rejected by `PUT` (T014) and by the gate (T030) — US-3 scenario 3.

**Checkpoint**: US3 works — grant/revoke/declaration-gating all behave per spec.

---

## Phase 5: User Story 4 - Existing same-origin apps are unaffected (Priority: P2)

**Goal**: Direct-HTTP assistant consumers (same-origin apps, BOS's own chat) are untouched (NFR-004 regression protection).

**Independent Test**: Confirm a same-origin app still uses `fetch("/api/assistant/runs")` successfully after the change; confirm BOS's own chat is unaffected.

### Implementation

- [ ] T040 [P] [US4] Verify no change to `src/app/api/assistant/**` or `src/lib/assistant/run-manager.ts` (integration points only, design.md §5). Confirm a same-origin app that does NOT declare `assistant` still reaches the run API via direct `fetch` (US-4 scenario 1–2).
- [ ] T041 [P] [US4] Verify the BOS UI's own chat (`src/lib/assistant/client/run-client.ts` path) is unaffected — the broker is a parallel path, additive only (NFR-004).

**Checkpoint**: All four user stories independently functional; no regressions.

---

## Phase 6: Verification & Polish

**Purpose**: Cross-cutting validation, docs, and correctness checks.

- [ ] T050 [P] [Polish] Write the companion dev doc `docs/dev/assistant/assistant-broker.md`: the capability, the six broker methods, the parent-push + bounded-buffer model, the four(+SDK) wiring places, reconnect semantics, per-app isolation, and the single-tail ordering constraint (design.md §8).
- [ ] T051 [P] [Polish] Add a Playwright e2e test (repo `e2e/` suite) exercising an opaque-origin app driving the assistant through the broker: start run → receive streamed `text_delta` events → execute a frontend `tool_call` → post result → `run_finished`; plus the ungranted-denial case (SC-001, SC-002). Model it on the existing app e2e patterns.
- [ ] T052 [Polish] Run `tsc --noEmit` (strict) clean and confirm NFR-001 (≤500 ms broker event latency) and NFR-003 (non-blocking) hold by inspection of the single-hop push path.
- [ ] T053 [P] [Polish] Record the pre-existing `storage`/`VALID_CAPS` drift in `/Specs/user-specs/discrepancies.md` (design.md §8) — noted, not fixed, to keep blast radius minimal.

**Checkpoint**: Feature complete — all stories verified, docs in sync, typecheck clean.

---

## Dependencies & Execution Order

- **Phase 1 (Foundational)**: no story work before it. T001/T002 parallel; T003 → T004 → T005 (same file, sequential); T006 depends on T003.
- **US1 (Phase 2)**: depends on Phase 1. T010/T012 parallel (different files); T011 depends on T004/T005; T013/T014 parallel (different files).
- **US2 (Phase 3)**: depends on US1's tail + SDK. T020 depends on T004; T021 depends on T012; T022 verification-only.
- **US3 (Phase 4)**: depends on Phase 1 gate + US1's Settings row/PUT. T030/T031 parallel; T032 depends on T014 + T030.
- **US4 (Phase 5)**: verification-only, depends on the full feature. T040/T041 parallel.
- **Phase 6 (Polish)**: depends on all stories. T050/T051/T053 parallel; T052 last.

### Parallel Opportunities
- T001, T002 (independent capability constants)
- T010 ∥ T012, and T013 ∥ T014 (distinct files)
- T030 ∥ T031; T040 ∥ T041; T050 ∥ T051 ∥ T053

## Implementation Strategy

Full-scope, single pass (per the spec's full-implementation preference): complete Phase 1 → US1 → US2 → US3 → US4 → Polish, validating at each story checkpoint. Because all work touches a small, known set of files, the Developer implements it in one delegated pass; the task phasing above documents the dependency order and independent-test boundaries rather than implying separate deployments.

## Notes

- All broker methods require the `assistant` capability; the gate is synchronous and pre-dispatch (design.md §3.7).
- The events-attach path is the only non-request/response broker interaction (the unsolicited `__bos_event` push); everything else uses the existing correlated `__bos_call`/`__bos_response` channel.
- Do NOT add a mid-run `push-surface-tools` method (out of scope, design.md §8) — tools ride the `startRun` body.
- Do NOT replicate the pre-existing `storage` gap — `assistant` is added to both `VALID_CAPS` and the conditional `AppsTab` row (ADR-6).
