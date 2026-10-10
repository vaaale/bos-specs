# Tasks: Browser Fill Form Schema Fix

**Input**: Design documents from `/Specs/user-specs/assistant/046-browser-fill-form-schema/`

**Prerequisites**: plan.md (required), spec.md (required for user stories)

**Tests**: REQUESTED — FR-006 mandates a TDD regression test (fail-first) and the FR-014 brief mandates ≥60% line & branch coverage on the modified file.

**Organization**: Tasks grouped by user story. Both stories are delivered by the single `ref` → `target` rename; US1 is the behavioral proof, US2 is the family-consistency proof.

**Hard scope (FR-014)**: The only files that may be modified are `src/lib/assistant/tools/server/browser.ts` and `tests/assistant/browser-tools.test.ts`. Any other file requires a stop + justification report.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2)

---

## Phase 1: User Story 1 — Agent fills a multi-field form in one call (Priority: P1) 🎯 MVP

**Goal**: The declared `browser_fill_form` schema names the per-field element key `target` (not `ref`) and requires `{ name, type, target, value }`, so an agent following the declared contract succeeds against the proxied `@playwright/mcp` server.

**Independent Test**: `browserTools().browser_fill_form.parameters.properties.fields.items` declares `target`, not `ref`, with `required: ["name","type","target","value"]` — and the existing suite is green.

### Tests for User Story 1 (WRITE FIRST — MUST FAIL on the unfixed base) ⚠️

- [ ] T001 [US1] Add a regression test in `tests/assistant/browser-tools.test.ts` that asserts the DECLARED schema of `browser_fill_form`: read `browserTools().browser_fill_form.parameters.properties.fields.items`; expect `properties` to define a `target` key (string) and to NOT define a `ref` key; expect `required` to be exactly `["name","type","target","value"]`. Run `npx playwright test -c playwright.unit.config.ts tests/assistant/browser-tools.test.ts` and confirm this new test FAILS against the current (unfixed) code (it declares `ref`), for the right reason.
- [ ] T002 [P] [US1] Add a regression test in `tests/assistant/browser-tools.test.ts` that asserts every element-targeting `browser_*` tool in `browserTools()` names its element parameter `target` and none names it `ref` — covering `browser_click`, `browser_type`, `browser_hover`, `browser_select_option`, `browser_evaluate`, `browser_take_screenshot`, AND `browser_fill_form` (via the `fields.items` key). This is the family-consistency guard (SC-002).

### Implementation for User Story 1

- [ ] T003 [US1] In `src/lib/assistant/tools/server/browser.ts`, in the `browser_fill_form` `serverTool`'s `schema(...)` `fields.items`: rename the per-field property `ref` → `target`, set its description to "Exact element reference from the latest browser_snapshot (e.g. \"e12\"), or a unique selector" (FR-004), and change the item `required` array from `["name","type","ref","value"]` to `["name","type","target","value"]` (FR-001, FR-002, FR-003). Do NOT change the executor (still `proxy("browser_fill_form", input, ctx)`) and do NOT add any rename/translation layer (FR-005).
- [ ] T004 [US1] Re-run `npx playwright test -c playwright.unit.config.ts tests/assistant/browser-tools.test.ts`; confirm T001 + T002 now PASS and the whole file is green.

**Checkpoint**: `browser_fill_form`'s declared contract matches the proxied server and the rest of the family. US1 independently verified.

---

## Phase 2: User Story 2 — Schema stays in lock-step with the browser tool family (Priority: P2)

**Goal**: After the fix, no element-targeting `browser_*` tool names its element key `ref`; the family is uniformly `target`.

**Independent Test**: The T002 test passes on its own — it enumerates the element key of each element-targeting tool and asserts uniformity.

### Implementation for User Story 2

- [ ] T005 [US2] Verify (no code change expected) that the T003 rename leaves all other `browser_*` tools' declared schemas byte-identical in effect (NFR-001): the only textual diff in `browser.ts` is inside the `browser_fill_form` `fields.items` block. If T003 introduced any incidental change to another tool, revert it.

**Checkpoint**: US1 and US2 both verified.

---

## Phase 3: Polish & Cross-Cutting (FR-014 brief)

- [ ] T006 [P] Run `npx tsc --noEmit` and `npm run lint`; both must be clean (NFR-003, Constitution VII).
- [ ] T007 Confirm coverage on the modified file `src/lib/assistant/tools/server/browser.ts` is ≥60% line and ≥60% branch; if below, add targeted assertions to `tests/assistant/browser-tools.test.ts` for uncovered branches (e.g. the `FOLLOW_WITH_SNAPSHOT` follow-up path for `browser_fill_form`, using the existing composition-test pattern).
- [ ] T008 [P] Confirm no file outside the hard scope (`src/lib/assistant/tools/server/browser.ts`, `tests/assistant/browser-tools.test.ts`) was modified; if any other file changed, STOP and report the justification before proceeding.
- [ ] T009 Confirm documentation needs no change (see plan "Documentation"): `docs/dev/automation/browser-automation.md` uses `ref` only as the snapshot *display* token, never as a `browser_fill_form` parameter. If the Developer finds any docs naming `ref` as a parameter, correct that page in the same change (Constitution VI / FR-006).

---

## Dependencies & Execution Order

### Phase Dependencies

- **US1 (Phase 1)**: T001/T002 (tests, fail-first) → T003 (fix) → T004 (tests green).
- **US2 (Phase 2)**: T005 depends on T003 (verifies the rename left the family untouched). T002 (the US2 test) may be written alongside T001.
- **Polish (Phase 3)**: T006–T009 depend on T004 (implementation + tests green). T006 and T008/T009 can run in parallel.

### Within Each User Story

- Tests MUST be written and confirmed to FAIL before the fix (T001 before T003).
- The fix (T003) is the single source change; it must not touch the executor or other tools.

### Parallel Opportunities

- T002 (US2 test) can be authored in parallel with T001 (US1 test).
- T006, T008, T009 can run in parallel once T004 is green.

---

## Notes

- [P] tasks = different files, no dependencies.
- Single-file source change; the bulk of the work is the TDD test that proves the declared contract.
- **Commit after each task or logical group** (tests, then fix, then polish).
- This is a self-heal scope-class-e (bos-core) fix on branch `bos/self-heal-0044`; do NOT call `dev_branch_request` (the branch is already active server-side).
