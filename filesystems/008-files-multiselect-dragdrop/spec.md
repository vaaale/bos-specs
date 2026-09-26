# Feature Specification: Files — Multi-Selection & Drag-and-Drop

**Feature Branch**: `008-files-multiselect-dragdrop`

**Created**: 2026-09-26

**Status**: Draft

**App Target**: `builtin-app` — the Files app at `src/apps/files/`. A first-class window app compiled into BOS; a change to existing BOS source, not a marketplace item. Rationale: the Files app needs direct VFS state and internal BOS APIs, is a thin wrapper around the VFS subsystem, and ships as part of BOS rather than being independently installable/removable.

**Input**: User description: "Improvements to the Files app — (1) select multiple items via shift-click or by dragging a rectangle (marquee); (2) right-click context-menu actions must apply to all selected items; (3) move/copy items — including multiple selected items — via drag and drop."

**Clarifications (2026-09-26)**: (a) Ctrl/Cmd+click toggling of a single item is **in scope** for multi-select; (b) drag-and-drop must **also** work between two separately-open Files app instances (cross-window), not only within a single window.

**Design decisions (confirmed at the design step, 2026-09-26)**:
- **View model (Q1=B)**: the file list switches from the current icon-tile **grid** to a **row list** as the Files app's primary view. Multi-select, Shift-click range, marquee, and drag-drop are all implemented over the row list.
- **Context menu (Q2=A)**: right-click opens a **full context menu** (Open / Rename / Copy / Move / Download / Delete), replacing the current Download-only menu. Single-only actions (Open, Rename) grey out for a multi-item selection; batch actions (Copy, Move, Delete, Download) apply to the whole selection.

## User Scenarios & Testing

### User Story 1 — Select multiple items (Priority: P1)

A user browsing a folder selects a group of items in one continuous gesture: they click the first item, hold **Shift** and click a later item to include everything in between; they press and drag an empty area of the file list to draw a rectangle that captures every item it touches; or they **Ctrl/Cmd+click** individual items to add to or remove from the current selection one at a time.

**Why this priority**: Multi-selection is the foundation for both later stories. Without a way to select more than one item at a time, "apply an action to all selected items" and "drag multiple items" have nothing to act on.

**Independent Test**: Open the Files app on a folder with ≥ 10 visible items. Shift-click two items with items between them and confirm every item from the first to the second is selected (highlighted). Then click-drag a rectangle over a subset and confirm exactly those items are selected. Each can be demonstrated with no other change in place.

**Acceptance Scenarios**:

1. **Given** a single item is selected, **When** the user Shift-clicks another item, **Then** every item from the originally selected item to the newly clicked item (inclusive, in list order) becomes selected.
2. **Given** no items are selected, **When** the user clicks and drags from an empty area of the list to form a rectangle, **Then** on mouse-up every item whose row is intersected by the rectangle is selected, and the rectangle indicator is removed.
3. **Given** several items are selected, **When** the user clicks a single item without any modifier, **Then** only that item is selected and all previously selected items are deselected.
4. **Given** items are selected, **When** the user presses **Ctrl/Cmd + A** (or clicks a "Select all" affordance), **Then** every item in the current folder is selected.
5. **Given** a set of items is selected, **When** the user Ctrl/Cmd+clicks an item that is *not* selected, **Then** it is added to the selection; **When** they Ctrl/Cmd+click an item that *is* selected, **Then** it is removed from the selection, with all other selections unchanged.

---

### User Story 2 — Context-menu actions apply to the whole selection (Priority: P2)

With multiple items selected, the user right-clicks and picks a single action from the context menu (e.g. Rename, Copy, Move, Delete, Download, Open). The action is applied to every item in the selection in one operation, not just the item that was clicked.

**Why this priority**: This is the payoff of multi-selection — batch operations. It depends on a selection existing (US1) but delivers most of the productivity value of the feature.

**Independent Test**: Select 3 files. Right-click and choose "Copy"; confirm a single confirmation/feedback indicates 3 items were copied. Select 2 folders and choose "Delete"; confirm both are removed (or, for the test, both show the delete confirmation).

**Acceptance Scenarios**:

1. **Given** 3 items are selected, **When** the user right-clicks on one of them and chooses "Copy", **Then** all 3 items are copied to the clipboard and the UI reflects a 3-item copy (count shown in the feedback/clipboard state).
2. **Given** multiple items are selected, **When** the user right-clicks, **Then** the context menu shows actions appropriate to the *set* (e.g. "Move" is offered; "Rename" is disabled or relabelled for >1 item since one name cannot fit several distinct items).
3. **Given** 2 items are selected, **When** the user right-clicks and chooses "Delete", **Then** a confirmation names both items (or the count) and, on confirm, both are deleted.
4. **Given** exactly one item is selected, **When** the user right-clicks it, **Then** the full single-item menu appears and behaves exactly as it does today (no regression).

---

### User Story 3 — Drag-and-drop to move or copy (Priority: P3)

The user drags one item or a group of selected items and drops them onto a destination folder. A drop into a *different* folder **moves** the items; a drop back into the *same* folder **copies** them. A drop onto the empty area of the current folder is a same-folder drop and therefore a **copy** (same semantics as dropping onto a folder in the same location); an *invalid* target — a file row, the dragged item itself, a descendant of a dragged folder, or a destination that changed mid-drag — is a no-op (the items stay put). The same gesture MUST also work **across two separately-open Files app windows**: dragging an item (or the current selection) out of one instance and dropping it into another performs the move/copy in the shared VFS and updates both views.

**Why this priority**: Drag-and-drop is the most expressive but least essential of the three capabilities; move/copy are also reachable via the context menu (US2). It is last because it is the most complex interaction to get right and depends on both selection (US1) and the copy/move operations (US2).

**Independent Test**: With 2 items selected, drag them onto a sibling folder and confirm both now appear in that folder and no longer in the source. Then drag a single item back into its own folder and confirm a duplicate with a non-colliding name is created.

**Acceptance Scenarios**:

1. **Given** 2 items are selected, **When** the user drags them (the selection follows the cursor) and drops on a different folder, **Then** both items are moved into that folder and are no longer in the source folder.
2. **Given** a single item is in a folder, **When** the user drags it and drops it in the *same* folder, **Then** a copy is created with a non-colliding name (e.g. `file (1).txt`) and the original is unchanged.
3. **Given** several items are selected, **When** the user starts a drag, **Then** the drag preview reflects the whole selection (count badge, e.g. "3 items"), not just the first item.
4. **Given** items are being dragged, **When** the user drops onto an *invalid* target (a file row, the dragged item itself, a descendant of a dragged folder, a destination that changed mid-drag, or outside the list), **Then** the items are returned to their original location with no error state. *(Dropping onto the empty area of the current folder is NOT invalid — it is a same-folder copy, per scenario 2.)*
5. **Given** two Files app instances are open, showing different folders, **When** the user drags a selection out of instance A and drops it into a folder in instance B, **Then** the items appear in that folder in instance B, are removed from instance A's view, and both instances reflect the shared VFS change.
6. **Given** two Files app instances are open showing the *same* folder, **When** the user drags an item from one instance and drops it into the other, **Then** a correctly-named copy is created (same-folder semantics) and the original in both views is unchanged.

---

### Edge Cases

- **Empty folder**: Marquee drag and Shift-click with no items present select nothing and cause no error.
- **Single-item folder**: Shift-click and marquee degrade gracefully to single-item selection.
- **Marquee starting on an item vs. empty space**: A drag that begins on an existing item initiates a *move/copy drag* (US3); a drag that begins on empty list space initiates a *selection marquee* (US1). The two gestures must not conflict.
- **Name collision on copy**: When a copy's target name already exists, the app appends/adjusts a numeric suffix so no existing file is overwritten.
- **Mixed types in a selection**: A selection containing both files and folders can be moved, copied, and deleted as a group; per-item actions that are type-specific (e.g. "Open", "Rename") reflect set-appropriate availability in the menu.
- **Selection across folder navigation**: Selecting items, then navigating into a subfolder, clears the selection (selection is scoped to the current folder).
- **Large selection**: Selecting and then dragging/deleting a large number of items (tens) completes without the UI freezing perceptibly (see SC-003).
- **Cross-instance, same folder**: Two open instances pointing at the same folder behave as a same-folder copy on a drop between them (a new named copy, original intact in both views) — see US3 scenario 6.
- **Cross-instance, destination navigated mid-drag**: If the destination instance's folder changes between drag start and drop, the drop is a no-op (items return to origin); the user must re-initiate the drag.

## Requirements

### Functional Requirements

- **FR-001**: The Files app MUST support multi-item selection in the file list.
- **FR-002**: Shift-click MUST select the contiguous range of items from the current selection anchor to the clicked item, inclusive.
- **FR-003**: A click-drag begun on empty list space MUST draw a visible rectangle (marquee) and, on release, select every item whose row it intersects.
- **FR-004**: A plain click MUST clear any prior selection and select only the clicked item.
- **FR-005**: "Select all" (Ctrl/Cmd+A) MUST select every item in the current folder.
- **FR-006**: The context menu MUST apply the chosen action to *all* currently selected items, not only the item under the cursor.
- **FR-007**: When more than one item is selected, the context menu MUST present set-appropriate actions (e.g. batch move/copy/delete; single-only actions such as Rename disabled or clearly indicated).
- **FR-008**: Dragging one item or the current selection MUST allow dropping onto a destination folder; a drop into a *different* folder MUST move the items, and a drop into the *same* folder MUST copy them.
- **FR-009**: A drag involving multiple items MUST reflect the whole selection in its preview (item count) and move/copy all of them on drop.
- **FR-010**: Copies MUST never overwrite an existing item; on name collision the app MUST generate a non-colliding name.
- **FR-011**: Selection MUST be scoped to the current folder and cleared on folder navigation.
- **FR-012**: A drop onto an invalid or empty target MUST be a no-op that leaves all dragged items in their original location.
- **FR-013**: All *existing* single-item behaviors — open, rename, delete, download, the open-with handler list, and the external-file upload drop path — MUST remain unchanged (no regression). Copy and Move are **new** capabilities introduced by this feature; they are also available in the one-item case but are not part of the pre-existing surface that FR-013 protects.
- **FR-014**: Ctrl/Cmd+click MUST toggle a single item in or out of the current selection without changing any other selected item.
- **FR-015**: Drag-and-drop of one item or a selection MUST work between two separately-open Files app instances (cross-window), not only within a single instance.
- **FR-016**: After a cross-instance move or copy, BOTH the source and destination instances MUST reflect the resulting VFS change (the source view drops moved items; the destination view shows the moved/copied items).

### Key Entities

- **Item**: A row in the file list — a file or a folder, with a name, type, and VFS path.
- **Selection**: The set of currently-selected items in the active folder, plus a selection *anchor* (the item a Shift-range is measured from).
- **Marquee**: The transient rectangle drawn while a selection drag is in progress.
- **Destination**: A folder that is the target of a move or copy operation.
- **Files instance**: A separately-open window of the Files app; each observes the shared VFS, so a change made by any instance is visible in all.
- **Shared drag payload**: The data carried by an in-progress drag (the VFS paths of the dragged items) that any Files instance can read to complete a cross-window move/copy.

## Success Criteria

### Measurable Outcomes

- **SC-001**: A user can select 10+ contiguous items in a single Shift-click gesture, with 100% of the intended items selected and no unintended items included.
- **SC-002**: A user can issue one context-menu action (copy/move/delete) and have it applied to every selected item in a single operation, with the feedback naming the correct item count.
- **SC-003**: Moving or copying a selection of up to 50 items via drag-and-drop completes without a perceptible UI freeze (interaction remains responsive throughout).
- **SC-004**: A drag into the same folder always produces a correctly-named copy and never overwrites an existing file.
- **SC-005**: No regression in single-item operations: every pre-existing one-item context-menu action behaves identically after the change.
- **SC-006**: A drag from one Files instance to another completes the move/copy in the shared VFS, and both instances reflect the result without requiring a manual refresh of either.

## Assumptions

- The Files app's file list is a vertical, scrollable row-based list (this feature switches the app from its current icon-tile grid to the row list — see Design decisions, Q1=B); "the rectangle touches a row" means horizontal overlap with any part of the row.
- Marquee selection and drag-move are distinguished by the *origin* of the pointer-down (empty space = select, item = move/copy).
- Copy is the same-folder drop semantics; move is the different-folder drop semantics. Explicit Copy/Move are also available from the context menu (US2) independent of drag-drop.
- Non-colliding copy naming follows the conventional `name (n).ext` suffix scheme.
- Selection is not persisted across navigation within a session; it is ephemeral to the current folder view.
- Keyboard modifiers: Shift for range; Ctrl/Cmd+A for select-all; Ctrl/Cmd+click toggles a single item in/out of the selection.
- Two open Files app instances are independent windows but both observe the **same shared VFS**; a move/copy made by either is reflected in both. The cross-window drag payload carries VFS paths that any Files instance can interpret to complete the operation (the mechanism by which one window hands a drag to another is an implementation detail left to `design`).
