# Tasks: Live Per-Tool Updates & Rich Recursive Tool-Call Rendering

**Input**: `plan.md`, `spec.md`, `design.md` (all in this directory).

**Prerequisites**: plan.md ✅, spec.md ✅. (No research.md / data-model.md / contracts/ — justified in plan.md.)

**Tests**: included and **TDD-ordered** — the spec's success criteria (SC-001…SC-010) are measurable, and Constitution VII requires the candidate's self-tests to pass before promote. Write each test FIRST, confirm it FAILS on the current code, then implement.

**Organization**: by user story (US1 / US2 / US3) for independent implementation and testing. The one shared dependency is the `NestedEvent` payload widening (Foundational) — it gates US2 (producer) and US3 (consumer) but not US1.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: different files, no dependencies — can run in parallel.
- **[Story]**: US1 / US2 / US3 / SHARED.
- Exact file paths in every description.

---

## Phase 1: Setup

**Purpose**: sanity baseline before touching code.

- [ ] T001 [SHARED] Confirm clean baseline on branch `bos/045-chat-live-tool`: `npx tsc --noEmit` and `npm run lint` pass with no changes, so post-change failures are attributable to this feature.

**Checkpoint**: baseline green.

---

## Phase 2: Foundational (Blocking for US2 & US3)

**Purpose**: the shared data contract. Both US2 (encode nested results into the terminal payload) and US3 (rebuild terminal child cards from that payload on reload) depend on the widened `NestedEvent`. US1 does **not**.

**⚠️ CRITICAL**: US2 and US3 must not begin rendering/streaming per-child data until the payload shape exists.

- [ ] T002 [P] [SHARED] **Test first**: round-trip unit test for the widened `NestedEvent` at `src/lib/agent/__tests__/nested-events.test.ts` — `encodeNested` → `parseNested` preserves `result?`, `status?`, and a `nested?` child list at depth ≥3; the marker byte is unchanged. Written first; FAILS until the type is widened.
- [ ] T003 [SHARED] Widen `NestedEvent` in `src/lib/agent/nested-events.ts`: add optional `result?` (string), `status?` (`"running" | "done" | "cancelled"`), `nested?` (`NestedEvent[]`); update `encodeNested`/`parseNested` to round-trip them (marker unchanged); export the type. Makes T002 pass. (design ADR-3 / B1.)

**Checkpoint**: shared payload contract ready — US2 and US3 can now begin (in parallel with US1).

---

## Phase 3: User Story 1 — Per-Settle Individual Completion (P1) 🎯 MVP

**Goal**: each parallel-safe call flips done individually on its own settle; the persisted transcript stays in original call order (deterministic replay).

**Independent Test**: one turn issuing a batch of ≥2 parallel-safe calls with staggered completion — each card flips at its own settle; reloading/replaying shows tool-result messages in original order regardless of live completion order. (SC-001, SC-002, SC-004.)

### Tests for US1 (write first, confirm FAIL)

- [ ] T004 [US1] e2e test in `e2e/045-chat-live-tool.spec.ts`: (a) a staggered parallel batch emits a distinct `tool_result` per `callId` **as each settles** (not one burst after the slowest); (b) the persisted run log's `message` events for that batch are in **original call order** even when live completion order differs; (c) cancelling the run mid-batch emits `tool_cancelled` once per still-running call promptly. Written first; (a) and (c) FAIL on the current batched loop.

### Implementation for US1

- [ ] T005 [US1] Rework the batch section of `src/lib/assistant/agent-loop.ts`: replace `await Promise.all(batch.map(executeCall))` + trailing emit loop with a **per-settle** `tool_result` emission (completion order), while keeping `message` + `io.saveMessages` in a trailing **original-order** loop. Preserve the per-call `.catch` isolation (defensive wrapper inside the `map`, not an outer catch). Cancelled calls emit `tool_cancelled` **exactly once** at settle; the trailing loop persists them without re-emitting a terminal event. (design ADR-1.)

**Checkpoint**: US1 independently functional — cards complete individually; transcript order invariant.

---

## Phase 4: User Story 2 — Live Nested Output for Delegations (P1)

**Goal**: a local/ephemeral/surface delegation streams each nested tool **result** live via the per-call `tool_progress` channel; the Claude/OpenCode CLI path stays starts-only.

**Independent Test**: delegate to a local subagent whose inner run calls ≥2 tools — the parent card's live progress shows each nested result as it completes while the delegation is still running; a Claude delegation shows nested starts (no per-child results). (SC-003.)

**Depends on**: Phase 2 (T003).

### Tests for US2 (write first, confirm FAIL)

- [ ] T006 [US2] e2e test (append to `e2e/045-chat-live-tool.spec.ts`, after T004 creates it): a local/ephemeral delegation forwards nested `tool_result` events live through `tool_progress` as each nested tool settles; the terminal `tool_result` payload's `nested?` list carries per-child `result?`/`status?`. Written first; FAILS until the inner loop forwards results.

### Implementation for US2

- [ ] T007 [US2] In `src/lib/assistant/inner-loop.ts`: forward nested `tool_result` / `tool_cancelled` (server-executed calls) via `ctx.onEvent`, not only nested `tool_call` starts; carry a per-`callId` `{ result, status }` list on the returned `InnerLoopResult` (the data is already computed in the inner loop's own `messages`/`outcomes` — currently discarded). (design ADR-3.)
- [ ] T008 [US2] In `src/lib/assistant/tools/server/delegate-common.ts`: pass the per-child result list from `InnerLoopResult` into the terminal `encodeNested` call so the persisted payload carries per-child `result?`/`status?`. The claude/OpenCode leg passes through its starts-only `toolCalls` unchanged (no new CLI `tool_result` parsing — out of scope).

**Checkpoint**: US1 + US2 both functional.

---

## Phase 5: User Story 3 — Recursive, Navigable Tool-Call Card (P2)

**Goal**: the card becomes a single recursive component (header action summary; independently-collapsible Input / Output; content-type Output; child cards recurse), matching `mockup.html`.

**Independent Test**: any completed tool call — expand header, expand Input (structured key-value + raw toggle) and Output (content-type) independently; for a done delegation, the Output shows full child cards, navigable ≥3 levels deep. (SC-006…SC-010.)

**Depends on**: Phase 2 (T003, for terminal child-card data). Render live nested cards from US2's `progress` when present (FR-016).

### Tests for US3 (write first, confirm FAIL)

- [ ] T009 [P] [US3] Component test at `src/components/agent/v2/__tests__/ToolCallCard.test.tsx`: (a) header shows a human-readable action summary (or tool-name fallback) with no raw JSON; (b) Input renders structured key-value with the primary arg emphasized, plus a raw-JSON toggle; (c) Output renders by content-type precedence (MCP-UI iframe → child cards → highlighted JSON → markdown) with a copy button on JSON/code; (d) Input and Output collapse independently; (e) a done delegation's Output renders full child cards navigable ≥3 levels from the terminal payload; (f) an empty result shows no Output section; (g) a running delegation renders live child cards (US2 progress) with one still running. Written first; FAILS until the card is reworked.

### Implementation for US3

- [ ] T010 [P] [US3] **New** `src/components/agent/v2/ToolCardSummary.ts`: per-tool action summary (verb + key argument, e.g. "Read file /path", "Web search for …", "Delegate to <agent>"), tool-name fallback, no JSON blob. (FR-012.)
- [ ] T011 [P] [US3] **New** `src/components/agent/v2/ToolCardSectionState.ts`: card-local per-`callId` Input/Output open state (module-side map, keyed by `callId`; header accordion stays on `card-collapse.ts`). (design ADR-2, option B; FR-015.)
- [ ] T012 [US3] **New** `src/components/agent/v2/ToolCardSections.tsx`: the **Input** section (structured key-value rows, primary arg emphasized, raw-JSON toggle — FR-013) and the **Output** section (content-type precedence: MCP-UI iframe → recursive child cards → highlighted JSON → markdown via the existing chat renderer, with copy affordance and a content-type label — FR-014). Depends on T010, T011.
- [ ] T013 [US3] Rework `src/components/agent/v2/ToolCallCard.tsx` into the single recursive component: header (summary + status dot) expands to the Input/Output sections; for a delegation the Output section is a list of **child `ToolCallCard`s** (same component, recursed, visually indented); live nested results (US2) render as child cards while running (FR-016); empty result shows no Output. Depends on T012. (FR-011, FR-015.)
- [ ] T014 [US3] Update `src/components/agent/v2/MessageListV2.tsx`: scope the header accordion per card (pass the `callId`-keyed scope) and pass the live `progress` and `resultText` into the card so it can render child cards in both the live and terminal states. Depends on T013.

**Checkpoint**: all three stories independently functional.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: complete the change per Constitution VI (docs sync) and VII (gates).

- [ ] T015 [P] [SHARED] **Docs — usage**: update `docs/usage` (the chat/assistant section) for the tool-call card rework: recursive card, per-tool individual completion, live delegation output, content-type rendering. (Constitution VI — user-visible change.)
- [ ] T016 [P] [SHARED] **Docs — dev**: update `docs/dev` (the assistant section) for the run-loop per-settle emission (completion vs. original order), inner-loop nested-result forwarding, and the widened `NestedEvent` payload shape. (Constitution VI — dev-facing change.)
- [ ] T017 [SHARED] Full e2e run: `e2e/045-chat-live-tool.spec.ts` green; verify SC-001…SC-010 hold end-to-end (deterministic replay, per-settle completion, live nested output, recursion depth, empty/MCP-UI/claude edge cases).
- [ ] T018 [SHARED] Quality gates: `npx tsc --noEmit` and `npm run lint` pass for all changed code; confirm **no** change landed in `run-events.ts`, `chat-store.ts` schema, `run-manager.ts`, `run-client.ts`, `Message.tsx`, or CopilotKit (FR-007 not-touched guard).

**Checkpoint**: change complete — spec + docs in sync, gates green, self-tests pass.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: no dependencies — start immediately.
- **Phase 2 (Foundational)**: depends on Setup. **Blocks US2 and US3**, not US1.
- **US1 (Phase 3)**: depends only on Setup — **runs in parallel with Phase 2 + US2/US3**.
- **US2 (Phase 4)**: depends on Phase 2 (T003).
- **US3 (Phase 5)**: depends on Phase 2 (T003); renders US2's live progress when present (FR-016) but is testable from the terminal payload alone.
- **Polish (Phase 6)**: depends on all three stories.

### Within Each Story (TDD)

- Tests written first and confirmed failing, then implementation.
- US3: T010 ∥ T011 (different files) → T012 → T013 → T014.
- US2: T007 → T008 (T008 consumes T007's `InnerLoopResult` list).

### Parallel Opportunities

- **US1 (T004/T005)** ∥ **Foundational (T002/T003) + US2 (T006–T008) + US3 (T009–T014)** — different files throughout, no shared write.
- Within US3: T010 and T011 in parallel; T015 and T016 (docs) in parallel.
- The single e2e file `e2e/045-chat-live-tool.spec.ts` is written sequentially (T004 → T006 append), so US1's and US2's e2e tasks are **not** parallel with each other.

---

## Implementation Strategy

### MVP First (US1 Only)

1. Phase 1 → 2 (or skip 2 for US1) → Phase 3 (US1).
2. **STOP and validate**: cards complete individually, transcript order invariant (SC-001/002/004). This is the headline fix and is independently shippable.

### Incremental Delivery

1. US1 (individual completion) → demo.
2. + US2 (live delegation output) → demo.
3. + US3 (recursive card) → demo.
4. + Polish (docs + gates) → change complete.

---

## Notes

- [P] = different files, no dependencies.
- Each `[Story]` label gives traceability back to spec.md.
- Commit after each task or logical group; stop at each checkpoint to validate the story independently.
- The **not-touched** guard (FR-007) is a hard constraint: `run-events.ts`, `chat-store.ts` schema, `run-manager.ts`, `run-client.ts`, `Message.tsx`, and CopilotKit are out of scope — T018 verifies this.
