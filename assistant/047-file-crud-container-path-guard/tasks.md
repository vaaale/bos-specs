# Tasks: Wire rejectContainerPath into the VFS CRUD file tools

**Input**: Design documents from `/Specs/user-specs/assistant/047-file-crud-container-path-guard/`

**Prerequisites**: plan.md (required), spec.md (required for user stories)

**Tests**: MANDATORY for this feature (FR-009, FR-014) — written FIRST, observed to fail, then made to pass.

**Organization**: Tasks grouped by user story, dependency-ordered, [P] for parallelizable.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story it belongs to
- Exact file paths included in every task.

---

## Phase 1: Foundational — failing regression tests (BLOCKING)

**Purpose**: Establish the regression guard BEFORE touching the implementation. These tests MUST fail against the unfixed code and for the right reason (the unfixed CRUD tools return a VFS-shaped outcome, not the diagnostic).

> **⚠️ TDD: Write these first. Watch them fail. Only then move to Phase 2.**

- [ ] T001 [US1] In `src/lib/assistant/tools/server/__tests__/files.test.ts` (new file), import `fileTools` (or the relevant exported handlers), `isRealContainerPath`, and `containerPathDiagnostic` from `../files`. Write a test for **`file_read`**: invoking it with `path = "/worktrees/bos/052-spec-tree-delete/specs/user-specs/project.json"` MUST reject, and the rejection's error message MUST equal `containerPathDiagnostic("/worktrees/bos/052-spec-tree-delete/specs/user-specs/project.json")`. Assert the message contains `"is a real on-disk container path, not a VFS path"`. **Run it; it MUST fail** (unfixed `file_read` reaches `vfs.readText` and surfaces an `ENOENT`/VFS outcome, not the diagnostic).
- [ ] T002 [P] [US1] In the same test file, add a **no-op** test for `file_read`: a path that `isRealContainerPath` classifies as NOT a container path (e.g. a clean `/Specs/…` or `/Documents/…` path) MUST NOT reject with the namespace diagnostic (it must produce the tool's normal VFS outcome — a read result or the existing ENOENT — never the diagnostic string).
- [ ] T003 [P] [US2] In the same test file, add a reject test for **`file_list`**: `path = "/app/data/foo"` rejects with `containerPathDiagnostic("/app/data/foo")`.
- [ ] T004 [P] [US2] In the same test file, add a reject test for **`file_write`**: `path = "/worktrees/bos/…/x.txt"` (plus any `content`) rejects with `containerPathDiagnostic(that path)`.
- [ ] T005 [P] [US2] In the same test file, add a reject test for **`file_mkdir`**: `path = "/tmp/somewhere"` rejects with `containerPathDiagnostic("/tmp/somewhere")`.
- [ ] T006 [P] [US2] In the same test file, add a reject test for **`file_delete`**: `path = "/home/user/stuff"` rejects with `containerPathDiagnostic("/home/user/stuff")`.
- [ ] T007 [P] [US2] In the same test file, add a reject test for **`file_rename`**: `path = "/var/log/a.log"`, `to = "/var/log/b.log"` rejects with `containerPathDiagnostic("/var/log/a.log")` (the FIRST container path, `path`, wins).
- [ ] T008 [P] [US2] In the same test file, add the **`file_rename` second-branch** test: `path` clean (non-container), `to` a container path (e.g. `path = "/Specs/user-specs/x.md"`, `to = "/var/log/b.log"`); the tool MUST reject with `containerPathDiagnostic("/var/log/b.log")` (the `to` value, with its own diagnostic naming `to`).
- [ ] T009 [P] [US2] In the same test file, add no-op tests for each of the five US2 tools mirroring T002: a clean VFS path MUST NOT produce the namespace diagnostic for `file_list`, `file_write`, `file_mkdir`, `file_delete`, and `file_rename` (first-segment-clean `path`/`to`).

**Checkpoint**: All Phase 1 tests written and confirmed FAILING against the unfixed code. T001 in particular must fail on the exact case-0053 call (SC-001).

---

## Phase 2: US1 — `file_read` (Priority: P1) 🎯

**Goal**: `file_read` rejects a real container path with the self-repairing diagnostic instead of a bare ENOENT.

**Independent Test**: T001 + T002 pass.

### Implementation for User Story 1

- [ ] T010 [US1] In `src/lib/assistant/tools/server/files.ts`, in the `file_read` handler (currently `async (input, ctx) => inScope(ctx, () => vfs.readText(String(input.path ?? "")))`), capture `const path = String(input.path ?? "")`, call `rejectContainerPath(path);`, then `return vfs.readText(path);` — keeping the whole body inside the existing `inScope(ctx, …)` wrapper (FR-001, FR-007). Match the shape the search/edit tools already use.
- [ ] T011 [US1] Run the Phase 1 tests for `file_read` (T001, T002); they MUST now pass.

**Checkpoint**: US1 done and independently testable.

---

## Phase 3: US2 — the other five CRUD tools (Priority: P2)

**Goal**: `file_list`, `file_write`, `file_mkdir`, `file_delete`, `file_rename` uphold the same contract.

**Independent Test**: T003–T009 pass.

### Implementation for User Story 2

- [ ] T012 [P] [US2] In `files.ts`, in the `file_list` handler, call `rejectContainerPath(path)` before `vfs.list`, where `path` is the input `path` coerced to a string and defaulted to `"/"` (guard runs against the defaulted value; FR-002). Keep inside the existing `inScope` wrapper.
- [ ] T013 [P] [US2] In `files.ts`, in the `file_write` handler, call `rejectContainerPath(path)` before `vfs.writeText` (FR-003).
- [ ] T014 [P] [US2] In `files.ts`, in the `file_mkdir` handler, call `rejectContainerPath(path)` before `vfs.mkdir` (FR-004).
- [ ] T015 [P] [US2] In `files.ts`, in the `file_delete` handler, call `rejectContainerPath(path)` before `vfs.remove` (FR-005).
- [ ] T016 [US2] In `files.ts`, in the `file_rename` handler, call `rejectContainerPath(path)` and then `rejectContainerPath(to)` (in that order) before `vfs.rename` (FR-006). The guard is a pure reject-on-first-hit; checking `path` then `to` gives the first-container-path-wins behavior the tests assert.
- [ ] T017 [US2] Run the Phase 1 tests for the five US2 tools (T003–T009); they MUST all pass.

**Checkpoint**: US1 + US2 both done; all eleven file tools in the module call `rejectContainerPath` (SC-003).

---

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Coverage target, docs sync, verification.

- [ ] T018 [P] Verify coverage: line AND branch coverage on `src/lib/assistant/tools/server/files.ts` is ≥60% (SC-002). If below target, add targeted tests for uncovered branches (notably `file_rename`'s `path`-clean/`to`-container branch and each tool's no-op branch) before reporting done.
- [ ] T019 Search `docs/dev/` and `docs/usage/` for any page describing the current per-tool container-path behavior (e.g. a page listing which file tools carry the guard, or documenting `file_read`'s failure modes). If found, update it in the same change to reflect that all six CRUD tools now reject a real container path with the namespace diagnostic (FR-011, Constitution VI). If none exists, record "no docs page to update" in the implementation report — do not invent a new docs page.
- [ ] T020 Run the full verification gates: `npx tsc --noEmit` and `npm run lint` pass on the changed code; the `files.test.ts` suite is green. Confirm the fix did NOT modify `rejectContainerPath`, `isRealContainerPath`, or `containerPathDiagnostic` (FR-010).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Foundational tests)**: No dependencies — start immediately. BLOCKS Phase 2/3 (tests must exist and fail before the fix).
- **Phase 2 (US1 `file_read`)**: Depends on Phase 1.
- **Phase 3 (US2 other five)**: Depends on Phase 1. Independent of Phase 2 (different handlers), but same file — T012–T016 are marked [P] for logical parallelism yet physically edit the same `files.ts`, so the Developer applies them as one coherent edit set.
- **Phase 4 (Polish)**: Depends on Phases 2 + 3.

### Within Each Phase

- Tests (Phase 1) MUST be written and observed to FAIL before implementation.
- The guard runs inside each tool's existing feature-scope wrapper, before the `vfs.*` call.
- A clean VFS path MUST remain behaviorally identical to today.

### Parallel Opportunities

- T002–T009 (Phase 1 test authoring) are independent test cases in one new file — they can be written together.
- T012–T016 (Phase 3 handlers) are independent handlers, but all in `files.ts` — apply as a single coherent edit.

---

## Implementation Strategy

### Single-file, one-pass (this is a well-specified, obviously-scoped fix)

1. Phase 1: write `files.test.ts` with T001–T009; run; confirm all FAIL (T001 fails on the exact case-0053 call).
2. Phase 2: add the guard to `file_read` (T010); run T001/T002; confirm PASS.
3. Phase 3: add the guard to the other five CRUD tools (T012–T016); run T003–T009; confirm PASS.
4. Phase 4: coverage (T018), docs sync (T019), gates (T020).
5. STOP and report: tests green, coverage ≥60%, docs checked, guard functions untouched.

---

## Notes

- [P] tasks = different files / no dependencies.
- [Story] label maps each task to a user story for traceability.
- The hard scope constraint is the plan's file list (`files.ts`, `__tests__/files.test.ts`, conditional docs page). Any file outside it requires a stop-and-justify.
- The mechanism touches NO Supervisor (`src/lib/assistant/tools/supervisor/**`) and NO build config (FR-023). If it did, stop and escalate.
- Verify tests fail before implementing (TDD is non-negotiable — FR-014).
