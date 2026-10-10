# Feature Specification: Fix Spec-Tree Deletion in Build Studio

**Feature Branch**: `bos/052-spec-tree-delete`

**Created**: 2026-09-29

**Status**: Draft

**App Target**: `bos-core`

**Input**: User description: "Right-click a spec item in the Build Studio (BS) left treeview → Delete → a warning dialog appears → click Ok. The deletion does not complete and the same warning box is shown again (infinite confirm loop). Also, right-clicking the folders must offer a working Delete." (GitLab issue #53, reported by @alex.)

## Background

Build Studio's left tree lists spec stores, the Projects/folders inside them, and the
files under each. Two deletion paths are broken:

1. **File delete loops on failure.** The confirm dialog only closes on a *successful*
   delete. When the underlying `DELETE /api/specs` is refused server-side, the dialog
   stays open, appends the server's error text, and re-fires the identical failing
   request on every Ok — which reads to the user as "the same warning box appears again".
2. **Folders cannot be deleted at all.** A container row's context menu is gated to the
   `project` node type. Plain `dir` and `feature` folders get no right-click menu, so
   "right-click a folder → Delete" is unreachable for any folder that is not a Project.

This feature fixes both, and makes a failed deletion of *any* kind surface its reason
once, out of a dialog the user can get out of.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — A file delete never traps the user in a re-appearing dialog (Priority: P1)

A user right-clicks a file in the Build Studio tree, chooses Delete, and confirms. If the
deletion succeeds, the file disappears and the tree updates. If it is refused for any
reason (read-only store, no feature branch selected, frozen section, store not mounted on
the active branch), the confirmation dialog **closes**, the file remains, and the reason is
shown **once** in the app's persistent error banner. The user is never asked to confirm the
same failing operation again.

**Why this priority**: this is the defect the reporter hit first and is the one that reads
as "the app is broken / stuck". Removing the trap is the highest-value fix; it is correct
regardless of the exact server-side reason the delete is refused.

**Independent Test**: with a file in a writable store and a feature branch selected, delete
it (success path). Then, with a branch *not* selected (or on a frozen section), attempt the
delete and observe: dialog closes, file remains, a one-line banner names the reason, and
re-attempting re-prompts rather than silently re-firing.

**Acceptance Scenarios**:

1. **Given** a file in a writable store and a feature branch selected, **When** the user
   right-clicks it → Delete → confirms, **Then** the file is removed, the dialog closes, and
   the tree no longer shows it.
2. **Given** a file whose delete is refused server-side (e.g. no feature branch, or a
   frozen section), **When** the user confirms the delete, **Then** the dialog closes, the
   file remains, and a one-time banner shows the server's refusal reason — and **not** a
   re-appearing confirmation box.
3. **Given** the user is in scenario 2's state, **When** they re-attempt the delete,
   **Then** a fresh confirmation is presented (they must consciously re-confirm) rather than
   the previous failing request being re-fired.

---

### User Story 2 — Any folder can be right-clicked and deleted (Priority: P1)

A user right-clicks a folder in the Build Studio tree — a plain `dir`, a `feature` folder,
or a `project` — in a writable store, and the context menu offers **Delete…** (and, where it
already did, Rename). Confirming runs a folder deletion that removes the folder and
everything inside it, after showing a count of what will be removed.

**Why this priority**: the second sentence of the report is an explicit, currently
unmet requirement ("It must be possible to right-click on the folders as well and choose
delete"). It is P1 because without it folders are simply undeletable.

**Independent Test**: create a folder containing a few files, select a feature branch,
right-click the folder (not a Project row) → the menu now shows Delete… → confirm → the
folder and its contents disappear, and the count quoted in the confirmation matched what
was removed.

**Acceptance Scenarios**:

1. **Given** a plain `dir` folder in a writable store with a feature branch selected,
   **When** the user right-clicks it, **Then** a context menu appears offering Delete…
   (and Rename… where applicable).
2. **Given** a `feature` folder containing several spec files, **When** the user chooses
   Delete… and confirms, **Then** the dialog first shows a count of what will be removed and,
   on confirmation, the feature folder and all its files are gone from the tree.
3. **Given** a folder in a read-only store, **When** the user right-clicks it, **Then** no
   Delete… is offered (matching the existing behaviour for other node types in that store).

---

### User Story 3 — A deletion that would remove nothing dangerous still asks once (Priority: P2)

Deleting an empty folder, or a folder whose only contents would be trivial, behaves the
same as any other folder delete: it is confirmed, the count (possibly 0) is shown, and it is
committed on the active feature branch. There is no fast-path that bypasses confirmation
for destructive operations.

**Why this priority**: consistency and safety over convenience — destructive ops are always
confirmed, even when the count is zero, so the behaviour is predictable.

**Independent Test**: create an empty folder, delete it with a branch selected; the count
reads 0 units, confirmation still occurs, and the folder is removed.

**Acceptance Scenarios**:

1. **Given** an empty folder, **When** the user deletes it and confirms, **Then** the
   confirmation reported a count of 0 units and the folder is removed.
2. **Given** any folder deletion (0 or N units), **When** the user does not confirm,
   **Then** nothing is deleted and the dialog can be dismissed.

---

### Edge Cases

- **Read-only store** (e.g. `bos-system-specs`): no Delete… is offered for files or folders;
  right-click does nothing, matching the existing file behaviour.
- **No feature branch selected** for a writable store: the action first elicits a feature
  branch (the existing "name this piece of work" flow) before the delete proceeds, so a
  branchless delete is refused with a human-facing message rather than an agent-facing one.
- **Frozen / terminal section**: a folder inside a frozen section is refused by the server;
  the UI shows the reason once (per Story 1) rather than looping.
- **Store not mounted on the active branch**: the server refuses; the UI shows the reason
  once and closes the dialog (per Story 1). This is the likely true cause of the original
  loop, and the client fix makes it non-entangling even though the underlying refusal
  stands.
- **Item-owned stores**: deleting a marketplace item is *not* a folder delete (it is an
  uninstall) and remains out of scope, as it is today; the item row is labelled by the app
  and is refused, not treated as a `dir`.
- **Deleting the currently-open file's ancestor**: if the user deletes the folder that
  contains the file open in the centre viewer, the viewer is left in a sane state (the open
  path is cleared or the viewer shows the not-found state), not a stale artifact.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: When a deletion (file or folder) is refused or fails, the system MUST close the
  confirmation dialog and surface the reason **once** in the app's persistent error banner.
  The failed dialog MUST NOT remain open, and a re-appearing confirmation box MUST NOT be
  shown.
- **FR-002**: After a failed delete, re-attempting the operation MUST present a fresh
  confirmation; the previous (failed) request MUST NOT be silently re-fired by the dialog
  machinery.
- **FR-003**: On a successful delete, the system MUST close the dialog, remove the node, and
  refresh the tree so the node is no longer shown.
- **FR-004**: The system MUST offer a right-click context menu on **every writable
  container node** — `project`, `dir`, and `feature` — in a writable store, not only on
  `project` rows.
- **FR-005**: The container context menu MUST offer **Delete…** for any writable container,
  and the existing **Rename…** affordance where it already applied.
- **FR-006**: Before a folder deletion is performed, the system MUST show a count of what
  would be removed (units and/or files) in the confirmation dialog, mirroring the existing
  `describe-project-deletion` pattern.
- **FR-007**: The system MUST provide a recursive folder-delete operation that resolves the
  folder within its store, enforces the same branch/writability/frozen gates as any other
  write, removes the directory and its contents, and commits on the active feature branch.
- **FR-008**: A folder deletion without an explicit confirmation MUST be refused by the
  server, consistent with the existing `delete-project` confirm-required behaviour.
- **FR-009**: No Delete… MUST be offered for nodes in a read-only store, nor for a
  marketplace item row (deleting an item is an uninstall, not a folder delete).
- **FR-010**: A delete that targets a node with no feature branch selected MUST route through
  the existing branch-elicitation flow before the operation proceeds.
- **FR-011**: If a deleted folder contained the file currently open in the centre viewer,
  the system MUST not leave a stale artifact open (it MUST clear the active path or surface
  the not-found state).

### Key Entities

- **Container node**: a non-file row in the spec tree — `project`, `dir`, or `feature` —
  identified by its store-prefixed path; deletable when its store is writable.
- **Folder deletion (describe/perform)**: the pair of operations that (a) report how much a
  folder delete would remove and (b) perform it, both keyed by store id + folder path and
  both riding the active feature branch.
- **Delete confirmation**: a single-use confirmation whose success and failure outcomes are
  now symmetric: both close the dialog; failure additionally posts its reason to the banner.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can delete a file in a writable store with a branch selected and see it
  gone in the tree, in a single confirm (no second dialog, no re-appearing box).
- **SC-002**: In 100% of failed-delete cases, the confirmation dialog closes and exactly one
  banner line names the reason; the number of times the user must click the confirm button
  to be told "no" is at most one per attempt.
- **SC-003**: A user can delete any of the following from a writable store with a branch
  selected, and see it removed: a plain `dir`, a `feature` folder, and a `project`. (0 of the
  three worked before; 3 of 3 must work after.)
- **SC-004**: The folder-delete confirmation quotes a unit/file count that matches what is
  actually removed, so "delete the empty folder" and "delete 30 specs" are visually distinct.
- **SC-005**: Deleting a folder that contains the currently open file does not leave a stale
  artifact displayed in the centre viewer.

## Assumptions

- The fix is scoped to Build Studio's spec tree and the spec APIs it calls; no new protocol,
  daemon, or marketplace-item facet is introduced. The server change extends the existing
  `POST /api/specs` op dispatch and `DELETE /api/specs`-style recursive removal, and the
  client change is confined to the tree's context-menu wiring and the confirm dialog's
  failure handling.
- "Folder" in the report means any non-file container node (`project` | `dir` | `feature`),
  since the reporter contrasted "folders" with the spec *item* they right-clicked.
- Deleting a marketplace item (an uninstall) is explicitly out of scope, as it already is.
- The exact server-side reason for the original loop (branch/worktree routing vs.
  commit-on-save) is not required to fix the client symptom; the client is made correct
  either way, and the server refusal — if it is the routing message — is surfaced verbatim.
- Confirmation is always required for folder deletion, even when the count is zero; there is
  no destructive fast-path.
