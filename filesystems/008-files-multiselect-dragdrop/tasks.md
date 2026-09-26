# Tasks: Files — Multi-Selection & Drag-and-Drop

**Input**: Feature specification from `specs/filesystems/008-files-multiselect-dragdrop/`
**Prerequisites**: plan.md (required), spec.md (required), design.md (structural design)
**Tests**: YES — the plan's Deliverables & Quality Gates explicitly require an e2e matrix + pure-function unit tests, so test tasks are generated.

**Organized by user story** for independent implementation and testing. Each task is an action the Developer sub-agent executes during `implement` — listing a path here is a task *description*, not something the authoring agent writes now.

---

## Phase 1: Setup

- [ ] T001 Create the five new module files as empty export skeletons per plan §Project Structure so the module graph is established first: `src/apps/files/selection.ts`, `src/apps/files/dnd.ts`, `src/apps/files/vfs-sync.ts`, `src/apps/files/FileList.tsx`, `src/apps/files/ContextMenu.tsx`

## Phase 2: Foundational (blocking prerequisites for all stories)

- [ ] T002 [P] Implement the full selection module in `src/apps/files/selection.ts` — `Selection` type (anchor + Set keyed by absolute `VfsEntry.path`, scoped to `cwd`), pure ops `plainClick` / `shiftRange` / `toggle` / `selectAll` / `marqueeSelect` / `prune`, plus `uniqueSiblingName` and `parentOf` helpers (D1)
- [ ] T003 Implement collision-safe server ops `copy(from, to)` and `move(from, to)` in `src/os/vfs.ts` — `copy` via recursive `fs.cp`, `move` via `fs.rename`, EEXIST retry loop on the target, cross-mount refusal consistent with existing `rename` (ADR-2, D5, R4, R5)
- [ ] T004 [P] Add `case "copy"` and `case "move"` to the POST switch in `src/app/api/fs/route.ts`, delegating to `vfs.copy` / `vfs.move` and returning the resolved destination path (D5)
- [ ] T005 [P] Add `fsClient.copy` and `fsClient.move` to `src/lib/os-client.ts`, mirroring the existing `rename` method shape (D5)
- [ ] T006 [P] Implement the same-document VFS-change channel in `src/apps/files/vfs-sync.ts` — `emitVfsChanged()` dispatcher + `useVfsChanged(cwd)` hook as the **single** refresh trigger for all instances (ADR-1, D6, PD-4/N-4)

## Phase 3: User Story 1 — Multi-Selection (Priority: P1)

**Goal**: Select multiple items via Shift-click range, Ctrl/Cmd+click toggle, marquee rectangle, and Select-all; plain-click clears (FR-001…005, FR-014; FR-011 scope).

**Independent Test**: In a folder with ≥10 items — Shift-click two rows and confirm the contiguous run is selected; Ctrl/Cmd+click to build a non-contiguous set; drag a marquee over empty space and confirm intersected rows select; Ctrl/Cmd+A selects all; plain-click leaves only that row.

- [ ] T007 [US1] Implement the row-list component in `src/apps/files/FileList.tsx` — rows (icon + name + size + mtime), per-row pointer / click / shift-click / ctrl-cmd-click / dblclick / contextmenu gestures, selection highlight, the empty-area marquee (pointer-captured `position:absolute` overlay, **not** `fixed`), and select-all; this replaces the current icon-tile grid (D2, Q1=B). **Rows MUST be width-capped and left-aligned (≈65% of the list width, name truncated within the cap, size/mtime right-aligned inside the cap) so a persistent empty region — right of the rows and below the last row — is always present as the marquee-start zone, regardless of item names** (FR-017, Q3=B, R2 — this is the fix for the as-shipped defect where full-width rows left no reachable empty space and every drag became a file-drag)
- [ ] T008 [US1] Wire selection state in `src/apps/files/index.tsx` — `selection` / `fresh` state, the cwd-change clear effect (FR-011), plain-click re-anchor, the Ctrl/Cmd+A keybinding, and `prune` on every refresh so vanished paths drop out
- [ ] T009 [P] [US1] Add pure-function unit tests for `src/apps/files/selection.ts` — shift-range, toggle, select-all, marquee intersection, prune, and `uniqueSiblingName`
- [ ] T010 [US1] Add e2e for US1 in `e2e/008-files-multiselect-dragdrop.spec.ts` — shift-click range, ctrl/cmd toggle (non-contiguous), marquee over empty space, select-all, and plain-click-clears (SC-001)

## Phase 4: User Story 2 — Context Menu on the Whole Selection (Priority: P2)

**Goal**: One right-click action applies to all selected items; menu shows set-appropriate availability (FR-006, FR-007); one-item case is a no-regression (FR-013).

**Independent Test**: Select 3 items → right-click → Copy and confirm a 3-item copy; select a folder + file → Move and confirm both move; with >1 selected, Rename/Open are greyed; with exactly 1, the menu behaves exactly as today (US2 scenario 4).

- [ ] T011 [US2] Implement the full context menu in `src/apps/files/ContextMenu.tsx` — generalizes `CursorMenu` (keep `document.body` portal + viewport clamp + `data-testid="files-context-menu"`); items Open / Rename / Copy / Move / Delete / Download; single-only items (Open, Rename) greyed for a multi-item selection; right-click on an unselected row promotes it to the selection, on a selected row keeps the whole selection (D3, Q2=A)
- [ ] T012 [US2] Add the batch action handlers in `src/apps/files/index.tsx` — `batchCopy` / `batchMove` / `batchDelete` / `batchDownload` (per-item `fsClient` calls, Promise.all), `batchOpen` (single-only), Move opens a folder picker, and the async open-with path reusing `fileHandlersClient` (D3)
- [ ] T013 [P] [US2] Add e2e for US2 in `e2e/008-files-multiselect-dragdrop.spec.ts` — multi-item menu set-enabling (Rename/Open disabled, batch enabled), one batch Copy/Move/Delete applies to the whole selection with correct count feedback, and the single-item menu no-regression = US2 scenario 4 (SC-002, SC-005)

## Phase 5: User Story 3 — Drag-and-Drop Move/Copy, Within and Across Instances (Priority: P3)

**Goal**: Drag one item or a selection to move (different folder) or copy (same folder); work across two open Files instances; collision-safe naming; invalid drop is a no-op (FR-008…012, FR-015, FR-016; SC-003, SC-004, SC-006).

**Independent Test**: Drag 2 items into a sibling folder → both move; drag an item back into its own folder → a `name (1)` copy appears, original intact; drag from one instance to another (non-overlapping windows) → both views update without a manual refresh.

- [ ] T014 [US3] Implement the drag/drop module in `src/apps/files/dnd.ts` — `VFS_DND_MIME`, `encodeDragPayload` / `decodeDragPayload`, `isVfsDrag` / `isUploadDrag`, `resolveDest` (validity **first**: drop-on-a-file-row / on-the-dragged-item-itself / on-a-descendant-of-a-dragged-folder → clean no-op **before** the op-decision; mid-drag cwd guard), the move-vs-copy op decision, and `buildDragImage` (ghost in the rendered DOM, not `display:none`) (D4, D6, PD-3/SF-3, N-2)
- [ ] T015 [US3] Add drag-start in `src/apps/files/FileList.tsx` — rows are `draggable`; `onDragStart` sets the VFS payload + `setDragImage` (whole selection, with a count badge); the pointer-down-on-row (move/copy) vs pointer-down-on-empty (marquee) split (D2, D4)
- [ ] T016 [US3] Re-dispatch the drag handlers in `src/apps/files/index.tsx` — `onDragOver` checks `isVfsDrag` **first** and calls `e.preventDefault()` (PD-1/SF-1) with `dropEffect` move/copy; `onDrop` resolves the dest, runs the batch copy/move, then emits the VFS-change (single trigger, PD-4); the `isUploadDrag` branch keeps the existing `uploadFiles` path byte-for-byte unchanged (D4, D6, FR-013)
- [ ] T017 [US3] Subscribe to the VFS-change in `src/apps/files/index.tsx` — `useVfsChanged(cwd)` re-fetches + prunes so **both** source and destination instances reflect the shared change without a manual refresh (FR-016, SC-006); the `fresh` row chip carries a reason — moved vs created (N-3)
- [ ] T018 [P] [US3] Add e2e for US3 in `e2e/008-files-multiselect-dragdrop.spec.ts` — in-window move (different folder) and copy (same folder, collision-safe naming); cross-instance move/copy in **non-overlapping side-by-side windows** (PD-2/SF-2, SC-006); an assertion that `drop` actually **fires** on both a same-instance and a cross-instance target (PD-1/SF-1); descendant + self drop → no-op (PD-3/SF-3); invalid drop → no-op (FR-012 — a file row, the item itself, a descendant, or a mid-drag-changed dest; an empty-area drop in the current folder is a same-folder *copy*, not a no-op); and a **50-item** selection dragged and dropped, asserting it completes within the responsiveness budget (SC-003, "Large selection" edge case)
- [ ] T019 [P] [US3] Add pure-function unit tests for `src/apps/files/dnd.ts` — payload encode/decode round-trip, `isVfsDrag` vs `isUploadDrag` mutual exclusivity, the `resolveDest` validity matrix, and the move-vs-copy op decision

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T020 Update `docs/usage` (Files app user doc) and `docs/dev` (developer doc) for the new multi-selection, drag-and-drop, full context menu, and the `fsClient.copy` / `fsClient.move` ops — in the same change as the code (Constitution Principle VI)
- [ ] T021 Add no-regression e2e in `e2e/008-files-multiselect-dragdrop.spec.ts` — the existing `data-testid` contract is intact, the external-file upload drop path is unchanged, and single-item open/rename/delete/download behaviors are preserved (FR-013, R3, SC-005)
- [ ] T022 Run quality gates — `npx tsc --noEmit` and `npm run lint` pass on all changed files, and the existing Files tests still pass (Constitution VII)

---

## Dependencies & Execution Order

```
T001 (setup)
  └─► T002 selection.ts ──► T007 FileList ──► T008 index state ──┐
      T003 vfs ops ──► T004 [P] api, T005 [P] client ────────────┼─► US2 (T011,T012,T013[P])
      T006 [P] vfs-sync ──────────────────────────────────────────┘        │
                                                                            ▼
                                              US3 (T014,T015,T016,T017, T018[P], T019[P])
                                                                            │
                                                                            ▼
                                              Polish (T020,T021,T022)
```

- **Foundational (Phase 2) blocks everything.** T002 (selection model) → US1/US2/US3. T003 (server ops) → US2 batch actions + US3 drop. T006 (vfs-sync) → US3 cross-instance.
- **US1 (P1) before US2 (P2) before US3 (P3).** US2 reuses US1's selection state + FileList; US3 reuses both and adds the dnd module + server ops.
- **Polish (Phase 6)** runs last: docs (T020), no-regression e2e (T021), quality gates (T022).

## Parallel Opportunities

- **Phase 2**: T002, T004, T005, T006 are `[P]` (distinct files: `selection.ts`, `route.ts`, `os-client.ts`, `vfs-sync.ts`). T003 must land before T004/T005 wire it, but once the `copy`/`move` signatures are agreed, the API + client + sync-channel tasks run in parallel.
- **Within each story**: the test tasks are `[P]` against their story's source tasks (different files) — e.g. T009 ∥ (T007,T008), T013 ∥ (T011,T012), T018/T019 ∥ (T014–T017).
- **Cross-story (advanced)**: once T002+T003+T006 are done, US2 and US3 could be parallelized by two workers (US2 = ContextMenu + batch handlers; US3 = dnd + drag), but US3's FileList drag-start touches the same file US1 built — so keep them sequential unless workers coordinate on `FileList.tsx`.

## Implementation Strategy

**MVP (User Story 1)**: T001–T010 — the selection model + row list + all four selection gestures + tests. Independently testable: select and verify selection state.

**Recommended first full increment (US1 + US2)**: T001–T013 — selection is only useful once it can drive a batch action, so US1+US2 together is the first *valuable* shippable slice (select → right-click → batch copy/move/delete). This is the natural first Promote.

**Full scope (US1+US2+US3)**: all of T001–T022. Per the user's stated preference for full-scope one-pass delivery on well-specified features, the intended deliverable is the complete feature — all three stories in a single implementation pass, gated by the Phase 6 quality gates and the cross-instance e2e (SC-006).

**Incremental path**: US1 → US2 → US3, each an independently testable increment, then polish. Each phase ends with its e2e green before the next begins.

## Task Format Reminder

Every task above follows `- [ ] T### [P?] [USx?] Description with file path`:
- Checkbox `- [ ]` (always)
- Task ID `T###` (sequential in execution order)
- `[P]` only when parallelizable (distinct files, no unmet dependency)
- `[USx]` only in user-story phases (Setup/Foundational/Polish carry no story label)
- Exact file path in the description
