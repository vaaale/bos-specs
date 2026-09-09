# Tasks: file_grep — Single-File Content Search Tool

**Input**: spec.md + plan.md from `/Specs/user-specs/assistant/043-file-grep/` (self-heal case 0002, class **e**, `bos-core`).

**Prerequisites**: plan.md ✅ (this dir), spec.md ✅ (this dir). No design.md (skipped — additive single-subsystem feature), no research.md / data-model.md / contracts/ (none warranted).

**Tests**: REQUIRED — spec NFR-004 mandates tests written **before** implementation that **fail on the unfixed base**, at ≥95% line + branch coverage on every modified file (SC-009). Test file: `tests/assistant/file-grep.test.ts`.

**Organization**: Tasks grouped by user story (US1/US2/US3) per spec. The whole feature is one cohesive tool in one file, so a story's tests + implementation are tightly coupled; each story is independently testable.

## Format: `[ID] [P?] [Story] Description`
- **[P]**: parallelizable (different files, no dependency on an incomplete task)
- **[Story]**: US1/US2/US3 (maps to spec user stories)
- Exact file paths in every description.

---

## Phase 1: Setup

**Purpose**: none required beyond confirming the working set (existing BOS repo, branch `bos/self-heal-0002`).

- [ ] T001 Confirm branch `bos/self-heal-0002` is active and `src/lib/assistant/tools/server/files.ts`, `src/lib/agent/capabilities-registry.ts`, and `tests/assistant/` are present; no `package.json`/lockfile/build-config edits are planned (Constitution VII)

**Checkpoint**: branch + target files confirmed.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: shared scaffolding both P1 stories need before their story-specific logic.

- [ ] T002 Add the output-bound constants next to `MAX_SEARCH_RESULTS`/`MAX_GLOB_RESULTS` in `src/lib/assistant/tools/server/files.ts` (e.g. `MAX_GREP_RESULTS` match-count cap and a per-line length cap reusing the existing `.slice(0, 200)`/`shorten` convention) — the caps chosen at plan; do NOT introduce a `SEARCHABLE_EXT` reference in the `file_grep` path (FR-009)

**Checkpoint**: constants in place; user-story implementation can begin.

---

## Phase 3: User Story 1 — grep one named file, line-numbered matches (P1) 🎯

**Goal**: An agent calls `file_grep(path, pattern[, ignoreCase, context])` on a single file and gets exactly that file's matching lines, each with a correct 1-based line number, in `path:line:text` form (SC-001, SC-004, SC-005).

**Independent Test**: Write a temp VFS file (via `useTestDataDir`) with known matches; call `fileTools().file_grep.execute!(input, ctx)`; assert the returned lines, line numbers, case-insensitivity behavior, and context-window behavior. Must FAIL before implementation (the tool does not exist on the base).

### Tests for User Story 1 (write FIRST, must FAIL on base)
- [ ] T003 [US1] In `tests/assistant/file-grep.test.ts`, set up the hermetic harness (`import "../services/_stub-server-only"`, `useTestDataDir("file-grep")`, invoke `fileTools().file_grep.execute!` with `ctx = { conversationId: RUN_ID, runId: RUN_ID, agentId: "ephemeral-test" }`) and assert the tool returns exactly the matching lines with correct 1-based line numbers in `path:line:text` form, and no content from any other file (FR-001/002/005, SC-001)
- [ ] T004 [P] [US1] In `tests/assistant/file-grep.test.ts`, assert case-sensitive default vs `ignoreCase: true` produce the specified match sets (FR-006, SC-004)
- [ ] T005 [P] [US1] In `tests/assistant/file-grep.test.ts`, assert `context: n` returns up to n preceding + n following lines with the matching line identifiable, and `context` omitted returns 0 extra lines (FR-007, SC-005)
- [ ] T006 [P] [US1] In `tests/assistant/file-grep.test.ts`, assert a valid file with zero matches returns a clean zero-match result distinguishable from an error (FR-010, SC-003); a file engineered to exceed the **match-count** cap returns a bounded result with an explicit truncation note + withheld count (FR-011, SC-006); and a file with a **very long matching line** returns that line truncated to the **per-line length** cap with a marker while the 1-based line number stays correct (FR-011, edge case "Very long lines") — together covering both FR-011 truncation branches NFR-004 requires

### Implementation for User Story 1
- [ ] T007 [US1] Implement the `file_grep` handler core in `src/lib/assistant/tools/server/files.ts` inside `fileTools()` (sibling of `file_search` `:156`): literal-substring line scan (no regex, FR-003), 1-based line numbers (FR-005), `ignoreCase` case-folding (FR-006), `context` windows de-duplicated (FR-007), single `vfs.readText` (NFR-001, no `walkFiles`), output bound to the T002 caps with truncation note (FR-011), returned as `JSON.stringify` in `path:line:text` form (NFR-003)
- [ ] T008 [US1] Wrap it as `parallel(serverTool("file_grep", <single-file description>, schema({path, pattern, ignoreCase, context}, ["path","pattern"]), handler))` in `src/lib/assistant/tools/server/files.ts`, with the handler body inside `withFeatureScope({ conversationId: ctx.conversationId }, ...)` (FR-001/002/004)

**Checkpoint**: `file_grep` returns correct line-numbered matches for a single file; US1 tests pass; US1 independently verifiable.

---

## Phase 4: User Story 2 — non-file path fails loudly, never a silent `[]` (P1)

**Goal**: A `path` that is not a readable regular file (directory, missing, unreadable, or undecodable/binary) returns an explicit in-band error — the exact defect that motivated the feature (SC-002).

**Independent Test**: Call `file_grep` with (a) a directory path, (b) a non-existent path, (c) an unreadable path, (d) a binary/undecodable file. Each must yield an explicit error naming the problem; never a silent empty array.

### Tests for User Story 2 (write FIRST, must FAIL on base)
- [ ] T009 [US2] In `tests/assistant/file-grep.test.ts`, assert a **directory** path returns an explicit "not a file" error (pointing to `file_search`), not `[]` (FR-008, SC-002)
- [ ] T010 [P] [US2] In `tests/assistant/file-grep.test.ts`, assert a **non-existent** path returns an explicit "no file at path" error, not `[]` (FR-008, SC-002)
- [ ] T011 [P] [US2] In `tests/assistant/file-grep.test.ts`, assert an **unreadable** path returns an explicit permission error, and a **binary/undecodable** file returns an explicit decode error (never silent empty) (FR-008/FR-009, edge case)

### Implementation for User Story 2
- [ ] T012 [US2] Implement the target-validation preamble in the `file_grep` handler in `src/lib/assistant/tools/server/files.ts`: call `await vfs.stat(path)` before reading; throw (→ in-band `Error: file_grep: …` via `serverTool`, `util.ts:16-34`) for missing, directory, and permission cases; make the `vfs.readText` decode failure throw an explicit binary/undecodable error instead of being swallowed (FR-008/FR-009)

**Checkpoint**: US1 + US2 both pass independently; non-file inputs fail loudly.

---

## Phase 5: User Story 3 — discoverable as the single-file grep (P2)

**Goal**: `file_grep` is registered in the FILES group with a "single file" description, and discovery routes single-file-grep intent to it while keeping `file_search` for directory search (SC-007, FR-012/FR-013).

**Independent Test**: With `file_grep` registered, inspect `listCapabilities()` for the `files` group: a `file_grep` row whose description says "single file" exists; a single-file-grep discovery query surfaces `file_grep`; a directory-search query still surfaces `file_search`.

### Tests for User Story 3 (write FIRST, must FAIL on base)
- [ ] T013 [US3] In `tests/assistant/file-grep.test.ts` (or extend `tests/assistant/discovery-modes.test.ts`), assert a `file_grep` capability exists in group `files` with a description stating **single file**, and that a single-file-grep intent surfaces `file_grep` while a directory-subtree intent still surfaces `file_search` — the two distinguishable, not collapsed (FR-012/FR-013, SC-007)

### Implementation for User Story 3
- [ ] T014 [US3] Add the `file_grep` capability row to the FILES block of `src/lib/agent/capabilities-registry.ts` (next to `file_search` `:117` / `file_glob` `:118`) with `group: "files"`, `context: "both"`, a description that says **single file**, and single-file-grep aliases (FR-012)
- [ ] T015 [US3] Apply the FR-013 alias correction in `src/lib/agent/capabilities-registry.ts`: make single-file-grep intent satisfiable by `file_grep` (e.g. the bare `"grep"` no longer claimed by `file_search` alone for the single-file case) while `file_search` remains the directory-subtree tool; do not collapse the two (SC-007)

**Checkpoint**: All three user stories independently pass.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T016 [P] Update `docs/dev/assistant/actions-and-tools.md` (the "Adding a tool (recipe)" section and/or tool-group listing) to document `file_grep` and how it differs from `file_search` (NFR-005)
- [ ] T017 [P] Regression check: confirm `file_search`, `file_glob`, and the other existing FILES tools are byte-unchanged in behavior — run their existing tests and confirm 0 regressions (SC-008, FR-014)
- [ ] T018 Run the full unit suite `npm run test:unit` and confirm the new `tests/assistant/file-grep.test.ts` passes AND fails on the unfixed base (SC-009)
- [ ] T019 Verify coverage on every modified file (`src/lib/assistant/tools/server/files.ts`, `src/lib/agent/capabilities-registry.ts`) is ≥95% line and ≥95% branch, covering the FR-008 non-file branches, FR-011 truncation branches, FR-006/007 flag branches, and the FR-009 binary branch (NFR-004)
- [ ] T020 Run `npx tsc --noEmit` and `npm run lint`; fix what this change broke; do NOT run `npm run build` while `next dev` is active (Constitution VII)

**Checkpoint**: docs in sync, no regressions, gates green. Feature complete.

---

## Dependencies & Execution Order

- **Phase 1 (Setup)**: none — start immediately.
- **Phase 2 (Foundational)**: T002 (constants) — needed by T007/T008.
- **User Story 1 (P1)**: T003–T006 (tests, fail on base) → T007–T008 (impl). Depends on T002.
- **User Story 2 (P1)**: T009–T011 (tests) → T012 (impl). Depends on T007/T008 (handler core must exist to add the validation preamble).
- **User Story 3 (P2)**: T013 (test) → T014–T015 (impl). Independent of US1/US2 handler logic (capability registry is a separate file), but the tool must be defined for the discovery test to resolve it — sequence after T008.
- **Polish (Phase 6)**: after all stories. T016–T020.

### Parallel Opportunities
- T004, T005, T006 (US1 test cases) are independent `[P]` within their file.
- T010, T011 (US2 test cases) `[P]`.
- T016 and T017 (docs / regression) `[P]` against each other in Phase 6.

---

## Implementation Strategy

Per the user's standing preference, deliver **full scope in one pass** (US1 + US2 + US3 together) rather than an incremental MVP — the feature is a single cohesive tool, and the spec's P1/P2 split is about priority, not staged value. Sequence within the pass: T002 → US1 (T003–T008) → US2 (T009–T012) → US3 (T013–T015) → Polish (T016–T020). TDD discipline holds: each story's tests are written and observed to FAIL on the base before its implementation.

---

## Notes

- **Additive only (FR-014 / SC-008)**: the only edits to `files.ts` are the new `file_grep` tool + the T002 constants; the only edits to `capabilities-registry.ts` are the new `file_grep` row + the FR-013 alias strings. `walkFiles`, `globToRegExp`, `relativeToRoot`, `SEARCHABLE_EXT`, `MAX_SEARCH_RESULTS`, `MAX_GLOB_RESULTS`, and every existing tool are untouched.
- **In-band errors, not throws**: `serverTool` (`src/lib/assistant/tools/server/util.ts:16-34`) converts executor throws into `Error: file_grep: <msg>` strings, so the FR-008 validation can `throw` internally and still satisfy "explicit in-band error, never a silent `[]`".
- **No new group / no new package / no build-config change**: the `files` group already exists (`src/lib/agent/tool-groups.ts:53`); `parallel`+`serverTool`+`withFeatureScope` are existing utilities; `package.json`/lockfiles untouched.
- **Verification of FR-013 is a test**, not hand-rolled routing code: discovery is driven by id/description/aliases via `src/lib/agent/discovery-score.ts`; the capability row + aliases are the implementation, the T013 assertion is the proof.
