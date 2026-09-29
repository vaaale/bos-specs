# Tasks: Fix Spec-Tree Deletion in Build Studio

**Input**: Design documents from `/Specs/user-specs/build-studio/052-spec-tree-delete/`

**Prerequisites**: `plan.md`, `spec.md`, `design.md`, `mockup.html` (binding UI contract)

**Organization**: Tasks grouped by user story. No Setup phase is needed — this is an
in-repo fix to three existing files, not a new project. The two new server ops are
**Foundational** because both US2 and US3 depend on them.

**Tests**: The spec's success criteria (esp. SC-002 "100% of failed-delete cases") and the
confirm-gate / count discipline on the new server ops warrant unit tests for the *only*
genuinely new server logic. The client dialog fix (US1) is verified against the acceptance
scenarios and `mockup.html`; it is not a separate code-test surface here.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: parallelizable (different files, no dependency on an incomplete task)
- **[Story]**: `US1` / `US2` / `US3`
- Exact file paths are in-repo (BOS root), per `plan.md` → Project Structure.

---

## Phase 1: Foundational (Blocking Prerequisites)

**Purpose**: The folder-delete op pair — shared by US2 and US3, and the only new server
logic. Nothing in US2/US3 can be wired until these exist.

**⚠️ CRITICAL**: US2 and US3 depend on this phase.

- [ ] T001 [P] Implement `describeDirDeletion(storePrefixedPath)` in
  `src/lib/specs/lifecycle.ts` — resolves the store via `resolveOpStore`/`resolveInStore`,
  walks the store subtree under `path`, returns `{ deletion: { label, files, units } }`
  counting files and feature units (read-only). (design.md §3.4)
- [ ] T002 [P] Implement `deleteDirIn(storePrefixedPath, ctx)` in
  `src/lib/specs/lifecycle.ts` — refuses with `409` + the describe count unless
  `confirm: true`; applies `assertEditablePath(path)` (frozen/terminal gate) and the branch
  gate; then calls `specfs.remove(path, { branch })` (already recursive + `commitOnSave`).
  (design.md §3.4, ADR-2/ADR-5)
- [ ] T003 Add the `describe-dir-deletion` and `delete-dir` cases to the POST op dispatch in
  `src/app/api/specs/route.ts`, mirroring the existing `describe-project-deletion` /
  `delete-project` handlers (including the 409-unconfirmed guard and `assertEditablePath`).
  (design.md §3.4)
- [ ] T004 [P] Add unit tests for the new ops in the repo's existing unit-test location for
  `src/lib/specs` (co-located `*.test.ts` per current convention): (a) `describe-dir-deletion`
  returns a correct file/unit count for a nested folder and `0` for an empty folder;
  (b) `delete-dir` without `confirm` is refused with a 409 carrying the count; (c)
  `delete-dir` with `confirm: true` removes the directory recursively and leaves it absent;
  (d) `delete-dir` on a frozen/terminal path is refused by `assertEditablePath`.

**Checkpoint**: server can describe and perform a recursive, confirm-gated folder delete.

---

## Phase 2: User Story 1 — A file delete never traps the user in a re-appearing dialog (Priority: P1) 🎯 MVP

**Goal**: make the file-delete dialog outcome symmetric — success and failure both close the
dialog; failure posts its reason **once** to the persistent error banner (no re-fire, no
re-appearing box). (FR-001 / FR-002 / FR-003 / FR-011)

**Independent Test**: with a writable store + feature branch, delete a file (success: gone
from tree, dialog closed). Then with no branch (or a frozen section), confirm a delete:
dialog closes, file remains, one banner line names the reason, and re-attempting re-prompts
rather than silently re-firing.

- [ ] T005 [US1] In `src/apps/build-studio/index.tsx`, rewrite
  `deleteFileAction.onConfirm` to the symmetric form (design.md §3.1): on `!r.ok`, `throw`
  the server error; in the `catch`, call `setConfirmDialog(null)` **and** `setError(e.message)`
  (do **not** `setDialogError` and leave the box open); in `finally`, clear `dialogBusy`.
  Success path: `setConfirmDialog(null)`, clear `activePath` if it equals an ancestor of the
  deleted path (FR-011), then `loadTree()`.
- [ ] T006 [US1] In `src/apps/build-studio/index.tsx`, pre-disable *Delete…* with a hint when
  the node's store is not on the active branch (`group.liveBranch` empty) — reuse the
  existing disabled-action pattern — so a guaranteed refusal is not sent (design.md §3.2,
  ADR-4). Leave the no-branch-at-all case to the existing branch-elicitation flow (FR-010).

**Checkpoint**: a failed file delete closes the dialog and shows the reason once; the
re-appearing-confirm loop is gone (SC-001, SC-002).

---

## Phase 3: User Story 2 — Any folder can be right-clicked and deleted (Priority: P1)

**Goal**: generalize the container context menu to every writable container
(`project | dir | feature`) and wire *Delete…* on a `dir`/`feature` to the folder describe/
perform flow with a confirm count. (FR-004 / FR-005 / FR-006 / FR-007 / FR-008)

**Independent Test**: in a writable store with a feature branch, right-click a plain `dir`
(menu now shows Delete…); on a `feature` folder with several files, choose Delete… — the
dialog first shows a count, and on confirm the folder and all its files are gone from the
tree. In a read-only store, no Delete… is offered on any container.

- [ ] T007 [US2] In `src/apps/build-studio/index.tsx`, replace the `renderNode` container
  menu gate `node.type === "project"` with `isWritableContainer(node)`
  (`project | dir | feature` **and** `group.writable`); route it to a generalized
  `openContainerMenu(e, node)` (design.md §3.3, FR-004). Keep Read-only and item-owned rows
  unmenued (FR-009).
- [ ] T008 [US2] In `src/apps/build-studio/index.tsx`, generalize `openProjectMenu` into
  `openContainerMenu`: for `project` keep today's menu (Rename… + Delete… via the existing
  `delete-project` op); for `dir`/`feature` offer **Delete…** (no Rename, since it did not
  previously apply) plus the existing View-history read action (design.md §3.3, FR-005).
- [ ] T009 [US2] In `src/apps/build-studio/index.tsx`, wire *Delete…* on a `dir`/`feature`
  to the folder flow (design.md §3.5, FR-006): open the confirm dialog immediately
  ("Checking what this would remove…"), call `describe-dir-deletion` to fill the
  count/units (`dialogBusy` keeps Confirm disabled until it arrives), and on confirm call
  `delete-dir { path, confirm: true }`. Route success and failure through T005's symmetric
  handler (close dialog; failure → banner once).
- [ ] T010 [US2] In `src/apps/build-studio/index.tsx`, apply T006's branch-uncoupled
  pre-disable to the container *Delete…* items as well, so a folder in a store not on the
  active branch is offered disabled-with-hint rather than sent as a guaranteed refusal
  (design.md §3.2, ADR-4).
- [ ] T011 [US2] Verify `describe-dir-deletion` / `delete-dir` are invoked with a
  **store-prefixed path** (not a project id) end-to-end from the menu → dialog → route →
  `lifecycle.ts` (ADR-3), and that `delete-dir` success triggers `loadTree()` so the folder
  and its contents disappear from the tree (FR-007, SC-003/SC-004).

**Checkpoint**: all three container kinds delete in a writable store (0 → 3 of 3, SC-003);
the confirmation quotes a count matching what is removed (SC-004).

---

## Phase 4: User Story 3 — A deletion that would remove nothing dangerous still asks once (Priority: P2)

**Goal**: consistency and safety — an empty (0-count) folder delete goes through the same
confirm flow as any other; no destructive fast-path is introduced. (FR-008, ADR-5, SC-004)

**Independent Test**: create an empty folder, select a branch, delete it: the confirmation
shows a count of 0 units, confirmation still occurs, and the folder is removed. A
not-confirmed delete removes nothing and the dialog dismisses.

- [ ] T012 [US3] In `src/apps/build-studio/index.tsx`, confirm the folder flow from US2
  **always** opens the confirm dialog before deleting — even when `describe-dir-deletion`
  returns 0 units — and never adds a bypass that commits without confirmation (ADR-5).
  Add an explicit code comment marking the destructive fast-path as intentionally absent.
- [ ] T013 [US3] Confirm the server guard from T002/T004 holds for the 0-count case:
  `delete-dir` without `confirm` is refused (409 + count, even when count is 0) — no
  special-casing of zero that skips the confirm requirement (FR-008).

**Checkpoint**: 0-count and N-count deletions are behaviourally identical except the quoted
number; both require confirmation (Story 3 scenarios 1–2).

---

## Phase 5: Polish & Cross-Cutting Concerns

- [ ] T014 [P] Cross-check the whole flow against `mockup.html` (binding UI contract):
  dialog copy, disabled-with-hint styling, count presentation, and the one-line banner on
  failure all match the mockup; fix any drift in `src/apps/build-studio/index.tsx`.
- [ ] T015 [P] Verify FR-011 across both file and folder paths: deleting a folder that
  contains the currently open file (centre viewer) clears `activePath` / surfaces the
  not-found state rather than a stale artifact (SC-005, design.md R2).
- [ ] T016 Run the full verification pass from `spec.md` → Acceptance Scenarios (US1–US3)
  and Success Criteria (SC-001…SC-005) against the running instance, and record any
  drift in `/Specs/user-specs/discrepancies.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Foundational (Phase 1)**: no dependencies — start here. **BLOCKS US2 and US3** (both
  wire to `describe-dir-deletion` / `delete-dir`).
- **US1 (Phase 2, P1)**: depends only on Foundational being *understood*, not on its code —
  it touches the client file-delete path and can proceed in parallel with Phase 1.
- **US2 (Phase 3, P1)**: depends on Phase 1 (server ops) **and** T005 (symmetric handler the
  folder flow reuses).
- **US3 (Phase 4, P2)**: depends on Phase 1 (409 guard) and US2 (the confirm flow it
  asserts).
- **Polish (Phase 5)**: depends on US1+US2+US3 complete.

### User Story Dependencies

- **US1 (P1)**: independent of US2/US3 code; can start immediately.
- **US2 (P1)**: needs Phase 1 + T005.
- **US3 (P2)**: needs Phase 1 + US2.

### Within Each Story

- Server ops before client wiring (Phase 1 → US2).
- Symmetric delete handler (T005) before folder wiring (T009) that reuses it.
- Menu gate (T007) before menu contents (T008) and wiring (T009).

### Parallel Opportunities

- T001 ∥ T002 ∥ T004 (all Phase 1; different functions/tests, no cross-dependency until T003
  dispatches them).
- US1 (T005, T006) ∥ Phase 1 — different files (`index.tsx` file-delete path vs
  `lifecycle.ts`/`route.ts`).
- T014 ∥ T015 (Polish, both read-only verification over the same built state).

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Foundational (server ops) — required context, even though US1 is client-only.
2. Complete Phase 2: US1.
3. **STOP and VALIDATE**: the reported infinite-confirm loop is gone (SC-001, SC-002).
4. Demo: failed delete closes the dialog and banners the reason once.

### Incremental Delivery

1. Phase 1 → Foundation ready (folder ops exist).
2. US1 → validate the D1 trap is removed (MVP for the reported bug).
3. US2 → folders become deletable with a count (the second reported defect).
4. US3 → confirm the 0-count safety invariant.
5. Polish → mockup conformance + FR-011 + full acceptance pass.

### Notes

- [P] = different files, no dependency on an incomplete task.
- Each story is independently completable and testable per its Independent Test.
- Commit after each task or logical group.
- Avoid: a destructive fast-path (US3), reusing bare `DELETE /api/specs` for folders (ADR-2),
  addressing folder ops by project id (ADR-3).
