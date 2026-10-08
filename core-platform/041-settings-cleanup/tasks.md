# Tasks: Settings cleanup (041-settings-cleanup)

**Input**: Design documents from `/Specs/user-specs/core-platform/041-settings-cleanup/`

**Prerequisites**: [plan.md](./plan.md) (required) · [spec.md](./spec.md) (required) · [design.md](./design.md) (design + verified orphan set + ADRs)

**Tests**: No automated test files are requested by the spec. Verification is the repo-wide `grep` (FR-015 / SC-003), typecheck + lint (FR-016 / SC-004), and manual confirmation of the Settings sidebar in a running build (SC-001).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies).
- **[Story]**: Which user story (US1, US2, US3) the task belongs to.
- Exact file paths are given in every task.

---

## Phase 1: Setup

**Purpose**: Establish the baseline before editing.

- [ ] T001 Confirm active feature branch is `bos/041-settings-cleanup` and the Developer works in the isolated worktree (Constitution IV).
- [ ] T002 Record the current `grep` baseline for the six FR-015 identifiers (`system-tools`, `@/lib/system/bash`, `runBash`, `BashResult`, `api/system/bash`, `KnowledgeBasesGrid`) across `src/`, to diff against after the deletions.

---

## Phase 2: User Story 1 — Remove dead "System Tools" page (Priority: P1) 🎯

**Goal**: The System Tools Settings page, its orphaned API route, and its orphaned server lib are gone; `GET /api/config` omits `system-tools`; `POST /api/system/bash` returns 404. `run_command` / Command Execution page unchanged (FR-001–FR-006).

**Independent Test**: Open Settings → no "System Tools" entry, all other modules render identically including Command Execution. `GET /api/config` returns no `system-tools` namespace. `POST /api/system/bash` returns 404. `run_command` still points to "Settings → Command Execution" when disabled.

### Implementation for User Story 1

- [ ] T003 [US1] Delete the `system-tools` `ConfigRegistration` block (the object with `namespace: "system-tools"`, `order: 37`) from the `REGISTRATIONS` array in `src/lib/config/registry.ts`.
- [ ] T004 [P] [US1] Delete the legacy endpoint file `src/app/api/system/bash/route.ts`.
- [ ] T005 [P] [US1] Delete the now-orphaned server lib `src/lib/system/bash.ts` (`runBash`, `BashResult`, `DEFAULT_TIMEOUT_MS`, `MAX_TIMEOUT_MS`, and the private `truncate` helper).
- [ ] T006 [US1] Confirm `data/config/system-tools.json` is left in place and untouched (FR-004 / ADR-3) — no migration or cleanup code added.

**Checkpoint**: System Tools page removed; orphan set for A closed.

---

## Phase 3: User Story 2 — Remove "Knowledge bases" agent tab (Priority: P2)

**Goal**: The per-agent editor shows only Instructions / Skills / Tools / MCP; the `KnowledgeBasesGrid` component and its barrel re-exports are deleted; the Knowledge Base item, its MCP tools, and the `kbs` allowlist model are unchanged (FR-007–FR-010).

**Independent Test**: Settings → Agents → any agent shows exactly four tabs, no "Knowledge bases" tab. `kb_search` / `kb_retrieve` still work. Typecheck is clean with no dangling references to the removed component.

### Implementation for User Story 2

- [ ] T007 [US2] In `src/components/apps/settings/assistant/AgentDetails.tsx`: remove `"kb"` from the `Tab` union (→ `"instructions" | "skills" | "tools" | "mcp"`), delete the `KnowledgeBasesGrid` import, delete the `<TabButton …>Knowledge bases</TabButton>`, delete the `{tab === "kb" && (… <KnowledgeBasesGrid … />)}` render block, and update the doc comment that says "five sub-tabs (… / Knowledge bases)" to four.
- [ ] T008 [P] [US2] Delete `src/components/apps/settings/assistant/KnowledgeBasesGrid.tsx`.
- [ ] T009 [P] [US2] In `src/components/apps/settings/assistant/index.ts`, remove the two re-exports (`KnowledgeBasesGrid` and `KnowledgeBasesGridProps`).
- [ ] T010 [US2] Confirm the preservation boundary: the `kbs` allowlist model, `src/lib/agent/kb-catalog.ts`, `buildKbIndexBlock` in `src/lib/agent/instructions.ts`, and the agent API's `catalog.kbs` payload are untouched (FR-009 / FR-010 / ADR-2).

**Checkpoint**: KB tab removed; Knowledge Base item runtime intact.

---

## Phase 4: User Story 3 — Sort Settings modules alphabetically (Priority: P3)

**Goal**: `listConfigSchemas()` returns schemas sorted A→Z by display title (case-insensitive, namespace tie-break) for built-ins and plugin panels together, with no client-side change (FR-011–FR-014).

**Independent Test**: Open Settings → sidebar is A→Z by title. Adding a hypothetical module (or a plugin panel) slots it alphabetically with no further code change.

### Implementation for User Story 3

- [ ] T011 [US3] In `listConfigSchemas()` (`src/lib/config/registry.ts`, ~line 942), replace the `.sort((a, b) => (a.order ?? 100) - (b.order ?? 100))` comparator with `a.title.localeCompare(b.title, undefined, { sensitivity: "base" }) || a.namespace.localeCompare(b.namespace, undefined, { sensitivity: "base" })` (ADR-1). Leave the `order` field on each `ConfigSchema` as inert data.

**Checkpoint**: Sidebar renders A→Z automatically.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Verify the whole change is closed, clean, and consistent.

- [ ] T012 Run a repo-wide `grep` over `src/` for `system-tools`, `@/lib/system/bash`, `runBash`, `BashResult`, `api/system/bash`, and `KnowledgeBasesGrid`; confirm zero hits (FR-015 / SC-003).
- [ ] T013 Run `npx tsc --noEmit` and `npm run lint`; confirm both pass with no dangling references (FR-016 / SC-004). Do not run `npm run build` while `next dev` is running (Constitution VII).
- [ ] T014 Verify no regression: `run_command`, the Command Execution page, and the Knowledge Base item (`kb_search` / `kb_retrieve`) behave exactly as before (SC-004).
- [ ] T015 Confirm no end-user or dev doc describes the removed System Tools page or KB tab as a supported feature; if any does, update it in the same change or record the drift in `/Specs/user-specs/discrepancies.md` (Constitution VI).

**Checkpoint**: All three cleanups complete; gates green.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately.
- **User Story 1 (Phase 2)**: Depends only on Setup. `T004`/`T005` (file deletions) are independent of `T003` (registry edit) and can run in parallel.
- **User Story 2 (Phase 3)**: Independent of US1 — different files. `T008`/`T009` are independent of `T007` and can run in parallel.
- **User Story 3 (Phase 4)**: Independent of US1/US2, but `T011` edits the same file as `T003` (`src/lib/config/registry.ts`) — sequence it after US1's registry edit to avoid a same-file conflict, or apply both as one coordinated edit.
- **Polish (Phase 5)**: Depends on all three user stories being complete.

### User Story Dependencies

- **US1 (P1)**: Independent — can be done alone.
- **US2 (P2)**: Independent of US1 (disjoint files).
- **US3 (P3)**: Shares `registry.ts` with US1; otherwise independent.

### Within the Feature

- Deletions (US1, US2) must be preceded by the orphan-closure confirmation (T002 baseline + T010) so a deletion does not strand a live import.
- All three stories land before the final `grep`/typecheck gates (T012, T013).

### Parallel Opportunities

- `T004` + `T005` (US1 deletions) in parallel.
- `T008` + `T009` (US2 deletions/exports) in parallel with `T007`.
- US1, US2, and US3 are disjoint enough to proceed in any order; the only same-file interaction is `registry.ts` (US1 `T003` + US3 `T011`).

---

## Implementation Strategy

### Deliver in one pass

The feature is small and cohesive (deletions + one re-sort). Implement US1 → US2 → US3, then run the Phase 5 gates once. Each story is independently verifiable against its Independent Test, but the final `grep` + typecheck confirm the whole orphan set is closed together.

### Notes

- [P] tasks = different files, no dependencies.
- Each task is scoped to one file (or one logical edit) so it can be executed without further context.
- `T011` and `T003` both touch `src/lib/config/registry.ts`; apply them as adjacent edits in the same file to avoid a merge conflict.
