# Tasks: Log Viewer — Shared Expandable Log Record Card

**Input**: Design documents from `/Specs/user-specs/core-platform/039-log-viewer-card/` — `plan.md` (scope contract), `spec.md` (user stories US1–US6), `design.md` (decisions D1–D7), `mockup.html`.

**Branch**: `bos/039-log-viewer-card` · **App Target**: `bos-core` · **Method**: TDD — every story's test is written and confirmed failing **before** its implementation.

**Organization**: Tasks are grouped by user story so each is independently testable. The **foundational** phase builds the shared pieces (stable key, exported clipboard, the card) that all surface stories depend on; US1 (toolbar) is the MVP.

> **Scope contract**: edit **only** the paths in `plan.md` (the two modified surfaces, the new card + key helper, the one-line export, and `docs/usage` for the §VI sync). Nothing outside it.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: which user story it belongs to (US1…US6)

## Path Conventions

Single project — `src/` at repo root, `e2e/` for the Playwright self-test.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: shared test scaffolding the story tests reuse.

- [ ] T001 [P] Create the Playwright self-test file `e2e/039-log-viewer-card.spec.ts` with the shared fixtures/helpers the story tests need (seed a central-log record with a long `err.stack` + large `data`, open the toolbar popover, open Settings → Logs) so US1–US6 tests stay focused on assertions, not setup.

**Checkpoint**: test scaffolding ready.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: the shared, surface-agnostic pieces. **No user-story surface work until this phase is complete.**

- [ ] T002 [P] (TDD) Write a unit test for the stable-key helper, then implement `logRowId(r)` in `src/components/logging/logRowId.ts` returning a deterministic id (e.g. `${ts}:${stream}:${component}:${msg.slice(0,40)}`). Per design D6 it MAY be colocated as an export in `LogRecordCard.tsx` instead — pick one and import it from both surfaces.
- [ ] T003 [P] In `src/components/logging/useLogContextMenu.tsx`, add `export` to `copyToClipboard(text: string): void` (one line; **body unchanged**). `openMenu` already takes `unknown`, so no other change.
- [ ] T004 Write a component test that renders `LogRecordCard` with a full sample canonical `LogRecord` and asserts: full `msg` (word-wrapped, not truncated); `err.message` + **full** `err.stack` in a scrollable, height-capped monospace `<pre>`; `data` as pretty JSON in a height-capped scrollable region; the stream badge; and that **absent** optional fields (`branch`, `versionLabel`, `conversation`, `buildLog`, no `err`) render nothing (no `null`/empty row). Then implement `src/components/logging/LogRecordCard.tsx`: a `"use client"` **presentational** leaf taking the **canonical** `LogRecord` from `@/lib/logging/models/log-models` (design D1); two-column label/value grid; omission rule (D1/FR-006); a local `safeJson`-shaped pretty-printer for `data` (D4); the `STREAM_COLOR` badge (D3); `buildLog` as monospace **text** not a link (D5); a single **Copy JSON** button calling `copyToClipboard(JSON.stringify(record, null, 2))` (T003); theme tokens only (`border-white/10`, `bg-black/35`, `font-mono`, `text-white/40` labels, level/stream colours) with a right-hand chevron that rotates on open (FR-013). No fetch, no store, no own state.

**Checkpoint**: foundation ready — `logRowId`, exported `copyToClipboard`, and the shared `LogRecordCard` exist and pass their tests. Surface integration can now begin.

---

## Phase 3: User Story 1 — Expand a toolbar log row to read the full record (Priority: P1) 🎯 MVP

**Goal**: a toolbar row expands **in place** into the shared card showing the full structured record, with the summary line unchanged.

**Independent Test**: open the toolbar Log popover, click a record with a long `err.stack`, and confirm the card renders the full record + complete stack with no copy and no navigation.

- [ ] T005 [US1] (TDD) e2e in `e2e/039-log-viewer-card.spec.ts`: open the toolbar Log popover → click a row → assert it expands in place to `LogRecordCard` showing the full `msg` and the **full** `err.stack`; assert the summary line for that row is preserved and that right-click → *Copy* still works on the summary. **Confirm it fails** before T006.
- [ ] T006 [US1] In `src/components/desktop/VersionControls.tsx` (`RecentLogPopover`): convert each row from `<div key={i}>` to `<details key={logRowId(r)} onContextMenu={(e) => openMenu(e, r)}>` with the **existing one-line summary markup moved verbatim** into `<summary>` (keep `cursor`, `whitespace-pre-wrap`, level colour, `LEVEL_COLOR` local to the summary); body = `<LogRecordCard record={r} />`. Import the **canonical** `LogRecord` from `@/lib/logging/models/log-models` and **remove the lossy local `interface LogRecord`**; drop now-redundant `r.message` references (`r.msg ?? r.message` → `r.msg`) and optional guards on required `level`/`ts`. Bump the list scroll area `max-h-[340px]` → `max-h-[440px]` (outer width `w-[900px] max-w-[92vw]` **unchanged**). Preserve, untouched: *errors only* toggle, red error badge, 15-minute window, escape-to-close, right-click Copy (FR-007/FR-010/FR-011/FR-015).

**Checkpoint**: US1 works — a toolbar row expands to the full card; right-click Copy and the toolbar chrome are unchanged. (This is the MVP.)

---

## Phase 4: User Story 3 — One shared card in both the toolbar and Settings → Logs (Priority: P1)

**Goal**: the Settings → Logs timeline's expanded body is **the same** `LogRecordCard`, so the two surfaces show an identical in-place detail.

**Independent Test**: open the same record's detail in the toolbar and in Settings → Logs and confirm the card body is identical (only the summary line differs per surface).

- [ ] T007 [US3] (TDD) e2e in `e2e/039-log-viewer-card.spec.ts`: open Settings → Logs, expand a record, assert its detail is `LogRecordCard` and field-for-field identical to the toolbar's for the same record (SC-004). **Confirm it fails** before T008.
- [ ] T008 [US3] In `src/components/apps/settings/LogsTab.tsx`: replace the hand-rolled inline body `<div>` (session/conversation/branch/err/data/buildLog) with `<LogRecordCard record={r} />`; key `{i}` → `logRowId(r)`; import the **canonical** `LogRecord` and drop the local `interface LogRec` where the card replaces its use (remove the now-unused local `safeJson` if nothing else references it); move the `STREAM_COLOR` map into the shared card (D3) so the card owns the stream badge for both surfaces. **Keep unchanged**: the `<summary>` markup (with its `truncate` msg), all filters, the session picker, auto-refresh, the record-count display, and the settings panel (FR-001/FR-007/FR-011).

**Checkpoint**: US3 works — both surfaces render the same `LogRecordCard`.

---

## Phase 5: User Story 2 — Several rows open at once (Priority: P1)

**Goal**: rows toggle **independently** — opening one never closes another.

**Independent Test**: open two rows and confirm both cards are visible at the same time; toggle one closed and confirm the other stays open.

- [ ] T009 [US2] (TDD) e2e in `e2e/039-log-viewer-card.spec.ts`: in the toolbar popover, open rows A and B, assert **both** cards are visible simultaneously; toggle A closed, assert B remains open. This is inherent to native `<details>` (multi-open, no "only one open" logic) — the test locks the behaviour in; add implementation only if the test reveals a regression.

**Checkpoint**: US2 works.

---

## Phase 6: User Story 4 — An open card survives the live 4-second refresh (Priority: P2)

**Goal**: an expanded row stays open and in place for the same record across the periodic re-sort/poll (no snap-shut).

**Independent Test**: open a row, wait through at least two 4 s polls (or trigger a new record that re-sorts the list), and confirm the open card is still open and still attached to the same record.

- [ ] T010 [US4] (TDD) e2e in `e2e/039-log-viewer-card.spec.ts`: open a row in the toolbar popover, advance time / trigger a re-sort so a new record lands above it, and assert the open card remains open and is still the card for the **same** record (SC-003). Realized by the stable `logRowId` key (T002, applied in T006/T008) + native `<details>` DOM-owned state (FR-008/FR-009); the test proves it and, if it fails, the fix is in the keying, not in any React open-state set.

**Checkpoint**: US4 works.

---

## Phase 7: User Story 5 — Copy the record's JSON from the card (Priority: P2)

**Goal**: the card offers *Copy JSON* via the **same** helper as the right-click *Copy*, producing identical pretty JSON.

**Independent Test**: open a card, trigger *Copy JSON*, paste, and confirm the result equals the record pretty-printed and matches the right-click *Copy* output.

- [ ] T011 [US5] (TDD) e2e in `e2e/039-log-viewer-card.spec.ts`: open a card, click *Copy JSON*, assert the clipboard content equals `JSON.stringify(record, null, 2)` and is byte-identical to the toolbar right-click *Copy* output for the same record (FR-012/SC-001). Uses the exported `copyToClipboard` (T003) and the card's button (T004); no new clipboard code.

**Checkpoint**: US5 works.

---

## Phase 8: User Story 6 — Accessible, theme-consistent interaction (Priority: P3)

**Goal**: rows are keyboard-togglable and the card reads as native BOS (existing tokens, no new palette).

**Independent Test**: Tab to a row, press Enter/Space to expand, confirm the expanded state is exposed, and confirm the card's colours/type match the surrounding surface.

- [ ] T012 [US6] (TDD) e2e/accessibility in `e2e/039-log-viewer-card.spec.ts`: focus a row with the keyboard (Tab), toggle it with Enter/Space (no mouse), and assert the native `<details>` exposes the expanded state; assert the card uses only the existing surface tokens (`border-white/10`, `font-mono`, level/stream colours, surface background) and the chevron rotates on open (SC-006/FR-013). Inherent to native `<details>` + token reuse; the test locks it in.

**Checkpoint**: all six user stories independently functional.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: quality gates, docs sync, and final self-test.

- [ ] T013 Update `docs/usage` for the Settings → Logs page and the top-toolbar Log control to document in-place row expansion and the shared record card (Constitution §VI — specs & docs stay in sync, same change as the code).
- [ ] T014 Run `npx tsc --noEmit` and `npm run lint` on the changed files; fix **every** error before finishing (Constitution §VII quality gates). (Do not run `npm run build` while `next dev` is running.)
- [ ] T015 [P] Run the full Playwright self-test suite `e2e/039-log-viewer-card.spec.ts` (US1–US6) and confirm all scenarios are green.
- [ ] T016 Cross-artifact consistency sweep: confirm the implementation matches `spec.md`/`plan.md`/`design.md` with no drift; if any divergence is found, record it in `/Specs/bos-system-specs/discrepancies.md` (Constitution §VI / spec-kit `analyze`).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies.
- **Foundational (Phase 2)**: depends on Setup — **blocks all user stories**. T002/T003/T004 are the shared card + helpers; surface work cannot begin until they pass.
- **User Stories (Phases 3–8)**: all depend on Foundational. US1 (toolbar) is the MVP; US3 (Settings) shares the card. US2/US4/US5/US6 are behaviour locks on the integrated result.
- **Polish (Phase 9)**: depends on all desired stories.

### User Story Dependencies

- **US1 (P1)**: after Foundational — no other-story dependency. **MVP.**
- **US3 (P1)**: after Foundational — shares `LogRecordCard`; independent of US1's surface but both need the card.
- **US2 (P1)**: after US1 (needs at least one surface using `<details>` to assert multi-open).
- **US4 (P2)**: after US1 (needs stable keys in place to assert survival).
- **US5 (P2)**: after Foundational (needs exported `copyToClipboard` + the card's button).
- **US6 (P3)**: after US1/US3 (needs both surfaces to assert theme consistency).

### Within Each User Story

- The test (TDD) is written and **confirmed failing** before its implementation task.
- Shared foundational pieces (card, key, clipboard) precede their use.
- Surface migration (US1/US3) precedes the behaviour locks (US2/US4/US5/US6).

### Parallel Opportunities

- T001, T002, T003 are mutually independent (different files) — start together.
- T015 (run e2e) is independent of T013/T014 ordering once all stories are done.
- US1 (toolbar) and US3 (Settings) touch different files after the foundational card exists — they can proceed in parallel.

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1: Setup (T001).
2. Phase 2: Foundational (T002, T003, T004) — **CRITICAL, blocks all stories**.
3. Phase 3: US1 (T005 test → T006 toolbar integration).
4. **STOP and validate**: a toolbar row expands to the full card; right-click Copy and toolbar chrome unchanged.

### Incremental Delivery

5. Phase 4: US3 — Settings → Logs adopts the shared card.
6. Phase 5–8: US2 → US4 → US5 → US6 — lock in multi-open, stable-refresh, card Copy, a11y/theme (each test-first).
7. Phase 9: Polish — docs sync, `tsc`/`lint`, full e2e, consistency sweep.

---

## Notes

- [P] = different files, no dependency.
- [Story] label maps each task to a user story for traceability.
- TDD: every story's test must fail before its implementation; do not "make the test pass" by weakening the assertion.
- Commit after each task or logical group; stay on `bos/039-log-viewer-card`.
- Keep edits inside the `plan.md` scope contract; anything outside it requires coming back to the spec.
