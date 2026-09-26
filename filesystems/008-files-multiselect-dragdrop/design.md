# Design — Files: Multi-Selection & Drag-and-Drop

**Feature**: `008-files-multiselect-dragdrop` · **Role**: architect · **Date**: 2026-09-26
**Spec**: `spec.md` (FR-001…FR-017) · **UI contract**: `mockup.html` (row-list + full context menu are the confirmed surface, Q1=B / Q2=A / Q3=B)

This design is grounded in the actual source. Where docs and source disagreed, source won. Real names used throughout: `fsClient` (`src/lib/os-client.ts`), `vfs.*` (`src/os/vfs.ts`), `/api/fs` (`src/app/api/fs/route.ts`), `useOSStore` (`src/store/os-provider.tsx`), `Window.tsx` (`src/components/desktop/Window.tsx`), `VfsEntry` (`src/os/types.ts`).

---

## 1. Classification

**Target: `builtin-app` — the Files app at `src/apps/files/`.**

This matches `spec.md`'s App Target verbatim. The Files app:
- ships as part of BOS (manifest `builtin: true`, `src/apps/files/manifest.ts`), is not independently installable/removable;
- needs direct VFS state and internal BOS APIs (`fsClient`, `useOSStore`, `fileHandlersClient`);
- is a thin client over the VFS subsystem.

**Scope of the change is a first-class modification to existing BOS source** (a builtin app plus a small, additive set of shared VFS/server/client APIs), not a marketplace item and not a core-OS behavioral change. `manifest.ts`, `Window.tsx`, and `os-store.ts` are **deliberately not modified** (see §3 — they already provide what we need; touching them is the thing we explicitly avoid in ADR-1).

**Scope note (FR-013 / SC-005).** The no-regression guarantee applies to the genuinely-existing single-item surface — today: Open (double-click), Rename (hover `Pencil`), Delete (hover `Trash2`), Download (right-click `CursorMenu`), the async "Open with" handler list, and the external-file upload-drop path. **Copy and Move are new** capabilities introduced by this feature (per FR-013); they are added to the one-item menu without altering any of those existing actions. This is exactly what FR-013's acceptance (US2 scenario 4: "the full single-item menu appears and behaves exactly as it does today") requires.

---

## 2. Context / container / component design

### 2.1 The container facts that drive everything

Two source facts make the whole design possible and are the reason the alternatives in the ADRs collapse:

1. **Multiple Files instances are already allowed.** `os-store.ts` `launch()` only enforces a single instance when `app.singleton` is set. `src/apps/files/manifest.ts` sets no `singleton`, so the user can already open two (or N) Files windows, each with its own `WindowInstance.id` and its own `params.path`. Nothing to change here.

2. **Builtin apps render inline in one document.** In `Window.tsx`, a non-iframe app renders as
   `createElement(AppComponent, { windowId, appId, params })` inside the window body `<div className="relative min-h-0 flex-1 …">`. Only `manifest?.kind === "iframe"` goes through `IframeApp`. The Files manifest has no `kind`, so **every Files instance is a React subtree in the same `document` / same `window`**. Consequences:
   - HTML5 `dataTransfer` is document-global → a drag started on instance A's row can be dropped on instance B's row with **no bridge code** (the browser hit-tests and routes the DOM drop).
   - A same-document `window` `CustomEvent` dispatched by one instance is received by every other instance → the "both views update" channel needs **no os-store change and no server event bus**.
   - **Caveat (risk):** window chrome is CSS-`transform`-positioned (`translate3d`), which makes the window the containing block for any `position: fixed` descendant (this is exactly why the existing `CursorMenu` portals to `document.body`). New fixed-position chrome (context menu) must portal to `document.body`; the marquee must be `position: absolute` inside the (scrollable) list, not `fixed`.

If Files were ever re-rendered as an iframe, both the DnD and the refresh channel would break (cross-iframe HTML5 DnD is sandbox-isolated; `window` events don't cross frames). It is not — that is recorded as Risk R1.

### 2.2 Component restructure of `src/apps/files/index.tsx`

Today `index.tsx` is one component that inlines: toolbar, the **icon-tile grid**, the right-click `CursorMenu`, the in-app text/image viewer, and the upload drag handlers. The restructure extracts the two interaction-heavy surfaces into their own components and keeps `index.tsx` as the state owner + orchestrator.

**`FileBrowser` (index.tsx) — state & orchestration.** New/changed state:
- `cwd: string` (existing) — the current folder.
- `entries: VfsEntry[]` (existing) — list for `cwd`.
- **`selection: { paths: Set<string>; anchor: string | null }`** (new) — the multi-select model, keyed by `VfsEntry.path` (stable, absolute). Scoped to `cwd`.
- **`fresh: Set<string>`** (new) — paths just created by a copy/move, for the brief "NEW" row highlight (mockup `row.new` + chip). Self-clears after a short timeout.
- `menu: { x, y } | null` (changed) — the context menu now anchors to the **selection**, not a single entry; `handlers` fetch is keyed to the single-item case (open-with is single-file only — directories have no handlers, per the existing 036 contract).
- `dragActive` (existing, **re-purposed**) — today means "external upload drag over the window." It now means "a drag is over the window that this instance will handle," and we split it into `uploadOver` (external `Files` drag) vs `vfsOver` (in-app VFS drag) so the overlay copy is correct ("Drop files to upload" vs "Drop to move/copy").

**Behavioral invariants preserved (FR-013):**
- `refresh(cwd)` is unchanged (`fsClient.list`).
- `openEntry` / `openInApp` / `launchHandler` / `pickHandler` (open-with, file-handlers integration) unchanged.
- `renameEntry` / `removeEntry` / `newFolder` / `newFile` / `saveText` / `setAsWallpaper` unchanged in behavior, but **`removeEntry` and batch ops are generalized** to take a path set (see 2.5); the single-item path is a length-1 call, so behavior is identical.
- `onDragOver/onDragEnter/onDragLeave/onDrop` are **kept** but re-dispatched through a small branch (see 2.6): if the drag is an in-app VFS drag → move/copy; else if it's an external `Files` drag → the **existing** `uploadFiles` path, byte-for-byte.

### 2.3 Selection model (`src/apps/files/selection.ts`)

Pure, testable logic. The model is **anchor + Set**, per the spec's Key Entities.

```ts
// illustrative only — not implementation
export interface Selection {
  paths: Set<string>;          // VfsEntry.path of every selected item
  anchor: string | null;       // the path a Shift-range is measured from
}
```

Row order is whatever `fsClient.list` returns (already sorted: dirs-first, then `localeCompare` — see `vfs.list`). Range math indexes into that array.

Operations (each returns a new `Selection`):
- **`plainClick(sel, path)`** → `{ paths: {path}, anchor: path }` — clears prior (FR-004, US1.3).
- **`shiftRange(sel, order, clickPath)`** → from `sel.anchor` to `clickPath`, inclusive, in `order` index terms; unioned with nothing (replaces). If `anchor` is null (no prior plain click), treat `anchor = clickPath` → single select (degrades gracefully, edge case "single-item folder"). **Anchor is left at its current value** so a subsequent Shift-click still measures from the original anchor (the literal reading of FR-002 "the current selection anchor"). A plain click re-anchors.
- **`toggle(sel, path)`** (Ctrl/Cmd+click, FR-014, US1.5) → add `path` if absent, remove if present; leave every other path untouched; set `anchor = path`.
- **`selectAll(order)`** (FR-005, US1.4) → `paths` = every path in `order`, `anchor` = first.
- **`marqueeSelect(order, rowRects, rect, { additive })`** (FR-003, US1.2) → every row whose bounding rect **intersects** `rect` (horizontal + vertical overlap — spec Assumption "the rectangle touches a row means horizontal overlap with any part of the row"). `additive` when Ctrl/Cmd held (union), else replaces. Empty intersection → empty/no-op (edge case "empty folder").
- **`prune(sel, validPaths)`** → after a refresh, drop selected paths that no longer exist (used by the receiving/source instance so a stale selection never references a moved/removed path).

**Scope & clearing (FR-011):** a `useEffect` on `cwd` resets `selection` (and `fresh`, and any in-progress drag/marquee) so selection never leaks across folder navigation. Selecting then navigating clears, as the edge case requires.

Keyboard: a single `keydown` listener on the list (focused) handles **Ctrl/Cmd+A** → `selectAll` (guarded with `preventDefault`), **Escape** → clear selection / close menu.

### 2.4 Row-list view + marquee + item drag (`src/apps/files/FileList.tsx`)

Replaces the current `<div className="grid … grid-cols-[repeat(auto-fill,minmax(96px,1fr))]">` grid with a **vertical row list** (the confirmed Q1=B surface; mockup `.row` layout: icon + name, 37px rows, `sel` highlight, `drop` highlight, `new` chip).

**Row width cap (FR-017, Q3=B — added post-implementation).** The rows are **width-capped and left-aligned** (≈65% of the list width; the name truncates within the cap) so that a **persistent empty region** — the space to the right of the rows *and* the area below the last row (the list container keeps its full height) — is **always present**, regardless of item names. This is what makes the empty-space-origin marquee reliably reachable: with full-width rows (the as-shipped defect) there is no reachable empty space, so every click-and-drag landed on a row and became a file drag instead of a marquee. The width cap guarantees the two-pointer-down-target split below always has two distinct, reachable targets. The list container stays the full window width (the cap is on each row, not the container), so the marquee overlay can still span the full list width.

Per-row interactions, **disambiguated by pointer-down target** (the spec's own assumption and Edge Case "Marquee starting on an item vs empty space"):

- **Pointer-down / click on a ROW** → selection gesture. Because rows are `draggable`, the browser's native drag (movement past threshold) becomes a **move/copy drag** (§2.6); a click that doesn't move is a **plain-click select**; Ctrl/Cmd+click is **toggle**; Shift+click is **range**; double-click is **open** (existing `openEntry`); right-click is **context menu** (promote-to-selection, §2.5). The native drag only starts after a real movement, so clicks/dblclicks/contextmenu are not eaten.
- **Pointer-down on EMPTY list space** (the list background, not a row — detected by `e.target` having no closest `[data-rowsel]` row) → **marquee**. Captures the pointer, tracks the rectangle in the list's **content** coordinates (offset by `scrollTop`/`scrollLeft`), renders an `position: absolute` overlay (mockup `.marquee`), and on pointer-up commits `marqueeSelect`. Because it originates on the background, it cannot conflict with a row's native drag — the two gestures start on different elements.

This is the crux of the gesture split (Risk R2): **row-origin = move/copy, empty-space-origin = marquee**, separated at the `pointerdown` target test, so neither path needs to detect "the other one."

The drag **ghost** for move/copy (FR-009 "preview reflects the whole selection — item count") is produced with `dataTransfer.setDragImage(ghostEl, x, y)` from an offscreen element rendering the mockup `.ghost` (stacked tiles + "N items" badge + MOVE/COPY tag). We do not hand-render a live cursor-following element (that would fight the window transform); the browser composites the drag image.

### 2.5 Context menu (`src/apps/files/ContextMenu.tsx`)

Generalizes the existing `CursorMenu` (kept: portal to `document.body`, `useLayoutEffect` viewport clamping, `contentKey` re-measure, `data-testid="files-context-menu"`). It now presents the **full Q2=A set** and enables per the selection (FR-006/FR-007):

| Action | Enabled when | Notes |
|---|---|---|
| **Open** | `selection.paths.size === 1` | Single-only (FR-007); for a dir navigates, for a file uses open-with/in-app (unchanged). Disabled for >1, relabeled "single item only" (mockup). |
| **Rename…** | `size === 1` | Single-only; existing `renameEntry`. |
| **Copy** | `size >= 1` | Batch; §2.7. |
| **Move…** | `size >= 1` | Batch; opens a destination picker (a lightweight folder chooser reusing the existing breadcrumb/folder model, or `window.prompt`-free dropdown). |
| **Download** | `size >= 1` | Batch; per-path `fsClient.downloadEntry` (dir → "Download as zip", matching the existing single-item label). |
| **Delete** | `size >= 1` | Batch; confirmation names the set (count, or both names for 2) per US2.3. |

Header names the set: `"1 item"` / `"N items selected"` (mockup `.mhead`). The **"Open with" handler section is preserved** for the single-file case (still keyed via `fileHandlersClient.list(fileBaseMime(path))` — directories have no handlers, unchanged from the 036 contract).

**Right-click promotion:** right-clicking a row that is *not* in the current selection sets `selection = { that row }` (so the menu acts on it, matching desktop convention) — but right-clicking a row that *is* in the selection keeps the whole selection. Right-click on empty space acts on the current selection (or opens nothing if empty).

**One-item case (FR-013 / US2.4):** with `size === 1`, every row is enabled and behaves exactly as the existing single-item behaviors (Open/Rename/Delete/Download/open-with unchanged); Copy/Move are newly present but inert w.r.t. regression.

### 2.6 The two drag paths, disambiguated (the upload-vs-move/copy boundary)

This is the single most regression-sensitive seam. The existing upload handlers must keep working **unchanged**; the new move/copy path is added as a *preceding* branch keyed on the drag's origin type.

- **External OS file drag → UPLOAD (unchanged).** The browser marks it with `dataTransfer.types` including `"Files"`. `isUploadDrag(e) = e.dataTransfer.types.includes("Files")`. When true: run the **existing** `onDragOver/onDragEnter/onDragLeave/onDrop` → `uploadFiles` → `fsClient.upload(cwd, files)` path, including the `files-drop-overlay` "Drop files to upload" and the `files-upload-progress` bar. **No behavior change.**
- **In-app VFS drag → MOVE/COPY (new).** Rows set a custom MIME on `dragstart`:
  `e.dataTransfer.setData(VFS_DND_MIME, encodeDragPayload({ paths: selection.paths (or the single row if it wasn't selected), sourceCwd: cwd, op: null }))` and also a `text/plain` fallback. `isVfsDrag(e) = e.dataTransfer.types.includes(VFS_DND_MIME)`. The window container's `onDragOver/onDrop` checks `isVfsDrag` **first**; if true it does **not** treat it as upload (and `isUploadDrag` is false for it, because we never set `"Files"`), so the two paths are mutually exclusive by construction.

The `VFS_DND_MIME = "application/x-bos-vfs-paths"` and all encode/decode + drop-target-resolution logic live in `src/apps/files/dnd.ts` (§3). Because both instances are in one document, `dataTransfer` carries the payload from A to B with no extra channel (ADR-1).

**Drop-target resolution & validity (FR-008/FR-012/edge cases):**
- Drop on a **folder row** (type `dir`, not one of the dragged items) → dest = that folder's `path`.
- Drop on the **list background** → dest = the instance's current `cwd`.
- Drop on a **file row** → invalid (can't move into a file) → **no-op** (FR-012).
- Drop where dest === the dragged item's own parent, or dest is a descendant of a dragged folder → **no-op** / or same-folder **copy** per §2.7.
- **"Destination navigated mid-drag"** (edge case): the destination instance records `cwdAtDragEnter` on the first `dragover` of the session; on `drop`, if its live `cwd !== cwdAtDragEnter`, **no-op**. This precisely implements "items return to origin; user re-initiates."
- On a **no-op**, selection is preserved (mockup P12: items "returned to origin, still selected," no error).

### 2.7 Move vs copy semantics + collision-safe naming (FR-008/FR-010)

The **operating instance** (the one receiving the drop — always the destination) decides the op from the paths, since it knows both the source parents (in the payload) and its own `cwd`:

```
for each dragged path p:
  destFolder = (drop on a folder row) ? that folder.path : thisCwd
  op = (parentOf(p) === destFolder) ? "copy" : "move"
```

All dragged items share the same source parent (the source's `cwd`), so the op is uniform across the selection. This yields the spec's rules exactly:
- different folder → **move** (US3.1, FR-015 cross-instance move).
- same folder → **copy** (US3.2), including **cross-instance same folder** (edge case US3.6: A.cwd === B.cwd ⇒ copy, original intact in both).

Both ops are **collision-safe**: the server computes a non-colliding destination name (`name (n).ext`, starting `n=1`, skipping any existing sibling) so a copy never overwrites (FR-010) and a move never clobbers either. The server returns the **actually-created path(s)** so the client can (a) refresh, (b) mark them in `fresh` (NEW chip), and (c) build the "Copied/Moved N items" toast (SC-002). Naming algorithm and rationale in ADR-2.

**Batching / SC-003 (50 items, no freeze):** v1 issues one `move`/`copy` server op per item, fired concurrently from the client (`Promise.all` over the selection) with a result aggregate, then a single refresh. Each op is a fast `fs.rename` (move) or `fs.cp` (copy). If profiling shows 50 concurrent requests jank, the follow-up is a batched server op (out of scope; see Risks).

### 2.8 "Both views update without manual refresh" (FR-016 / SC-006) — the channel

The operating (destination) instance, after its VFS mutation succeeds:
1. refreshes its own `cwd` (`fsClient.list`) and sets `fresh` for created paths;
2. **broadcasts** the change to every other inline instance via a same-document `window` `CustomEvent`:
   `window.dispatchEvent(new CustomEvent("bos:files:changed", { detail: { affected: Set<string>, destCwd } }))` (helper in `src/apps/files/vfs-sync.ts`).

Every Files instance subscribes in a `useEffect` (`bos:files:changed`); on receipt it re-fetches its own `fsClient.list(cwd)` and **prunes** its selection to still-existing paths. The **source** instance thus drops the moved items; the **destination** already shows them. No manual refresh, no os-store mutation, no server event bus — purely the same-document fact from §2.1. (This is a *different* channel from the DnD `dataTransfer`, which carries the payload at drop time; the CustomEvent carries the "VFS changed, refresh" signal afterward.)

The same broadcast is (optionally, low-cost) fired by other mutating app actions (delete/rename/upload/new) so any instance stays honest; the feature *requires* it for move/copy only.

---

## 3. File / module plan — ONLY what this feature creates or modifies

### Modified (existing files)

| Path | What changes |
|---|---|
| `src/apps/files/index.tsx` | Restructure: drop the inline grid → render `<FileList>`; add `selection`/`fresh`/`uploadOver`/`vfsOver` state + `cwd`-clear effect + `bos:files:changed` subscription; generalize `removeEntry` to a path-set batch; add batch Copy/Move/Delete/Download handlers; re-dispatch `onDragOver/Enter/Leave/Drop` through the upload-vs-vfs branch; move menu ownership to `<ContextMenu>`; keep open-with/viewer/upload logic. |
| `src/os/vfs.ts` | Add two **server-only** core functions: `copy(from, to): Promise<string>` and `move(from, to): Promise<string>` (each returns the resolved destination path). `move` = collision-safe `fs.rename`; `copy` = collision-safe recursive `fs.cp`. Reuse existing `list`, `normalizeVfsPath`, `resolveSafe`, and the mount-boundary refusal pattern from `rename`. |
| `src/app/api/fs/route.ts` | Add `case "copy"` and `case "move"` to the POST `switch`, each calling the new `vfs.*` op and returning `{ ok: true, to }` (the resolved path) so the client can refresh + mark NEW. |
| `src/lib/os-client.ts` | Add `copy: (from, to) => …` and `move: (from, to) => …` to the `fsOps()` factory (so both `fsClient` and `fsClient.scoped` get them), mirroring the existing `rename` shape but returning the resolved `to`. |

### Created (new files)

| Path | Responsibility |
|---|---|
| `src/apps/files/selection.ts` | The `Selection` type + pure ops: `plainClick`, `shiftRange`, `toggle`, `selectAll`, `marqueeSelect` (rect∩row), `prune`. Plus the `name (n).ext` helper `uniqueSiblingName(base, existingNames)` (used for client-side preview; the server remains authoritative). |
| `src/apps/files/dnd.ts` | `VFS_DND_MIME`; `encodeDragPayload`/`decodeDragPayload`; `isVfsDrag(e)` / `isUploadDrag(e)`; drop-target resolution (`resolveDest(dropTarget, draggedPaths)`); validity rules (file-row invalid, self/descendant invalid, `cwdAtDragEnter` mid-drag guard); move-vs-copy op decision; `buildDragImage(paths, op)` for `setDragImage`. |
| `src/apps/files/vfs-sync.ts` | The `window` `CustomEvent` channel: `emitVfsChanged({ affected, destCwd })` + `useVfsChanged(handler)` hook (subscribe/cleanup). The FR-016 both-views-update mechanism. |
| `src/apps/files/FileList.tsx` | The row list (replaces the grid): renders rows from `entries` + `selection` + `fresh` + `dropHighlight`; owns per-row click/shift/ctrl/dblclick/contextmenu, `draggable` + `dragstart`, and the empty-space **marquee** (pointer-captured, absolute overlay). |
| `src/apps/files/ContextMenu.tsx` | The full Q2=A context menu (generalized `CursorMenu`): portal + viewport clamp preserved; Open/Rename/Copy/Move/Download/Delete with per-selection enabling + set-count header; the preserved "Open with" single-file section. |

### Deliberately NOT modified (recorded so reviewers don't expect them)

- `src/apps/files/manifest.ts` — multi-instance already works (no `singleton`); inline rendering already the default (no `kind`).
- `src/components/desktop/Window.tsx` — relied on (inline, same document); not touched.
- `src/store/os-store.ts` — we intentionally do **not** add drag or VFS-change state here (ADR-1).

---

## 4. Integration points — existing mechanisms merely called into (not owned)

- **`fsClient` existing ops** (`src/lib/os-client.ts`): `list` (refresh), `read` (viewer), `write`/`mkdir` (new file/folder), `remove` (delete, generalized to a loop over the selection set), `rename` (single rename), `downloadEntry` (download), `rawUrl` (image viewer), `upload` (the preserved upload path). We call these; we add only `copy`/`move` (§3).
- **`fileHandlersClient.list/setSelected`** + `buildLaunchParams`/`fileBaseMime` (`src/os/file-handlers.ts`): the open-with registry — preserved unchanged for single-file open.
- **`useOSStore`** (`src/store/os-provider.tsx`): `setTitle` (window title), `launch` (openEntry hand-off to a handler app), `applySettings` (set-as-wallpaper in the viewer). Read-only to us; **no new store actions added.**
- **Existing upload drag handlers** in `index.tsx` (`onDragOver/Enter/Leave/Drop` + `uploadFiles` + `files-drop-overlay`/`files-upload-progress`): called into, not restructured — the new path is a preceding branch only.
- **Inline rendering in `Window.tsx`** + the **shared single-document** fact: the substrate for both DnD (`dataTransfer`) and refresh (`window` CustomEvent). Relied on, not modified.
- **VFS core primitives** (`src/os/vfs.ts` `list`/`rename`/`resolveSafe`/`normalizeVfsPath`/mount handling): the new `copy`/`move` build on these; we do not alter existing ones.

---

## 5. ADRs

### ADR-1 — Cross-instance drag-and-drop mechanism

**Decision: native HTML5 drag-and-drop carrying a shared custom MIME payload (`application/x-bos-vfs-paths`) of VFS paths. No custom event-bus handoff, no os-store-mediated drag state.**

Options evaluated against the real source:

- **A. Native HTML5 DnD + shared MIME (chosen).** Rows set a custom MIME on `dragstart`; the destination's existing `onDrop` slot reads it. Works **because Files instances render inline in one document** (§2.1): `dataTransfer` is document-global and the browser performs the cross-subtree hit-testing and drop routing for free. It reuses the *existing* `onDragOver/onDrop` handlers (added branch, not new infrastructure), gives us a real system drag ghost (we refine it via `setDragImage` for FR-009's count badge), and **disambiguates from upload cleanly**: an OS file drag carries `types ∋ "Files"` (→ upload, unchanged), an in-app row drag carries our custom MIME and **not** `"Files"` (→ move/copy). Mutually exclusive by construction — no state to reconcile.
- **B. Custom event-bus handoff (rejected).** An event bus carries *data*, not a *drop location*. To know "drop onto folder X," the source would have to track the global pointer and hit-test it against every other instance's folder-row rects — i.e., reimplement DnD. Heavier, and it needs each instance to publish row bounding boxes. No benefit over A given same-document rendering.
- **C. OS-store-mediated drag (rejected).** Would require adding live-drag state (paths + cursor + active-target) to `src/store/os-store.ts`, a **shared container this feature does not own** (and §3 lists it as deliberately untouched), plus hit-testing against per-instance rects, plus coordinating dragend across instances. More moving parts, more coupling, and it still needs the DOM to tell us where the drop landed. A gets the browser to do exactly that.

**Why A wins here specifically:** the inline-rendering fact means there is no iframe boundary to cross — the exact condition that makes native DnD trivial. Had Files been an `IframeApp` (`kind: "iframe"`), A would break (cross-iframe DnD is sandbox-isolated) and we'd be pushed to C; it is not.

**Companion channel (not an alternative):** the *post*-drop "both views update" signal is a same-document `window` CustomEvent (`vfs-sync.ts`, §2.8) — deliberately **not** the `/api/events/stream` bus (that's a server round-trip / cross-origin mechanism for cross-app events; overkill and wrong-ownership for same-document instances) and **not** an os-store mutation. DnD (dataTransfer) and refresh (CustomEvent) are two separate, both zero-new-infrastructure channels, both enabled by inline rendering.

### ADR-2 — Move/copy implementation: compose from existing ops vs. add client/server ops

**Decision: add first-class `move` and `copy` server ops + VFS core functions + `fsClient.move/copy`. Do NOT compose copy from `read`+`write` in the client.**

What `fsClient`/server actually expose today (`src/lib/os-client.ts`, `src/app/api/fs/route.ts`, `src/os/vfs.ts`): `list`, `read` (**utf8 only** — `op=read` → `vfs.readText`), `write`, `mkdir`, `delete`, `rename`, `upload`, `downloadEntry`. **There is no copy and no move op.** `vfs.rename` is an atomic `fs.rename` (O(1), files *and* dirs, refuses cross-mount) — that *is* a move primitive, but it **clobbers an existing target** (POSIX `fs.rename` replaces) and does no collision naming. There is **no recursive copy** primitive at all.

Why *not* compose (Alternative A):
- **Binary-unsafe:** `fsClient.read` is `readText`/utf8. Composing copy as `read(path)` + `write(dest, content)` corrupts images, zips, any non-text. The spec's own examples (`screenshot-…png`, `archive-2025.zip`) would break.
- **No directories:** `read` on a dir fails; there is no recursive walk to compose from. Copying a folder (FR "mixed types in a selection can be moved/copied") is impossible.
- **No collision safety / race:** collision-safe `name (n).ext` naming needs the destination listing; doing that client-side is unreliable for non-`cwd`/cross-instance destinations and races concurrent instances. Server-side naming is the single source of truth.
- **Round-trips:** N items ⇒ 2N sequential requests; SC-003 (50 items, no freeze) suffers.

Why add ops (chosen):
- **`copy` = collision-safe recursive `fs.cp(src, dst, { recursive: true })`** — Node's built-in, mirrors the existing `fs.rm(recursive)` idiom in `vfs.remove`, correct for bytes and dirs, one request per item.
- **`move` = collision-safe `fs.rename`** — reuses the existing atomic rename machinery (and its cross-mount refusal) but first picks a non-colliding destination name so a move never overwrites.
- Both **return the resolved destination path**, letting the client refresh, mark NEW rows, and build the count toast (SC-002).
- Naming algorithm (server, in `vfs.ts`): split `base`/final-extension; for `n = 1,2,…`, candidate = `n===1 ? base : "${base} (${n})"` re-joined with the extension; pick the first not in the destination `list`. A retry-on-EEXIST loop guards the TOCTOU window between the listing and the write (Risk R4).

**Note on move-vs-plain-rename:** we add a dedicated `move` op rather than having the client call the existing `rename` after computing a name, because (a) the client cannot reliably know a non-`cwd` destination's siblings, and (b) it collapses to 1 request instead of list+rename (2). The `name (n).ext` scheme then lives in exactly one place (the server), used by both copy and move.

---

## 6. Risks

**R1 (lead) — FR-016 / SC-006: "both views update without manual refresh."**
The guarantee rests on *both* Files instances living in the **same document** (inline rendering, §2.1). Mechanism: destination self-refreshes + dispatches a `window` `CustomEvent`; every other instance re-fetches its `cwd` and prunes stale selection.
- *Failure mode:* if Files were ever rendered as an `IframeApp` (or the two "instances" were separate OS windows/tabs rather than windows in one BOS session), `window` events and `dataTransfer` do **not** cross the boundary → the source view goes stale until a manual refresh, violating SC-006.
- *Mitigation:* (a) Files is inline today and the design depends on it — assert this in a unit/e2e test that two Files instances share a document; (b) the source prunes selection on refresh so a moved item can't linger as a dead selection; (c) "instances" are scoped to **one BOS session / one document** (the spec's Key Entities: "two separately-open Files app instances … both observe the same shared VFS"); distinct browser tabs are out of scope by definition.

**R2 — Marquee-vs-move/copy gesture disambiguation (pointer-down origin).**
Both gestures start with a `pointerdown`; a mis-classification either steals a marquee into a drag or a drag into a marquee. **The as-shipped defect proved the latent weakness of this split: with full-width rows the "empty space" target did not actually exist, so every drag originated on a row.**
- *Mitigation:* separate by **target element at pointerdown** — rows are `draggable` (native drag on movement), the list *background* starts the marquee via pointer capture. A row is not the background, so the paths cannot overlap; the native drag only begins after a movement threshold, so plain-click / dblclick / contextmenu selection are preserved. **The background target is guaranteed to exist and be reachable by FR-017/Q3 (width-capped rows ⇒ a persistent empty region to the right and below the rows).** Cover with a test: drag-from-row ⇒ move/copy; drag-from-empty-region ⇒ marquee (including in a folder whose rows carry long names — SC-007); click-on-row ⇒ select; none of these should trigger the other.

**R3 — Grid→row-list is a regression-sensitive change to an existing app (FR-013 / SC-005), including the upload-drop path.**
We are replacing the primary view of a shipping app. The existing upload drag-and-drop, open (dblclick), rename, delete, download, open-with handlers, in-app viewer, and set-as-wallpaper must all keep working.
- *Mitigation:* (a) the upload path is preserved as a *preceding-branch-free* sibling — `isUploadDrag` (`"Files"`) still routes to the untouched `uploadFiles`/`fsClient.upload`; the new path only engages on our custom MIME, which an OS drag never sets; (b) keep every existing `data-testid` (`files-entry`, `files-context-menu`, `files-drop-zone`, `files-drop-overlay`, `files-upload-progress`, `files-open-with*`) so existing tests pass; (c) `CursorMenu`'s portal + viewport-clamp logic moves verbatim into `ContextMenu`; (d) single-item menu is a length-1 case of the batch handlers, so one-item behavior is structurally identical.

**R4 — Collision-naming TOCTOU under concurrent instances.**
Two copies/moves into the same folder at once can compute the same `name (1).ext`. *Mitigation:* server retry-on-EEXIST loop in `vfs.copy/move`; worst case a higher suffix, never an overwrite (FR-010 holds).

**R5 — Cross-mount move/copy.**
`vfs.rename` already refuses cross-backend renames; `copy`/`move` adopt the same refusal (consistent, avoids corrupting a mount's own bookkeeping). Same-mount (the common case: within `/Documents`, across sibling folders) is fully supported. Cross-mount (e.g., `/Documents` → `/Specs`) is a documented limitation, not a crash.

**R6 — SC-003 (50 items, no perceptible freeze).**
v1 fires per-item server ops concurrently (`Promise.all`) then one refresh. If 50 concurrent requests prove janky, add a batched `move`/`copy` server op (accepts `from[]`, returns `to[]`) — a small, isolated follow-up. The ops themselves are `fs.rename`/`fs.cp`, both fast; the risk is request fan-out, not per-op cost.

---

## 7. Design review — verdict & folded findings (architect-reviewer, 2026-09-26)

**Verdict: Ready for plan.** All seven load-bearing source claims independently verified true (no `copy`/`move` op today; `read` is utf8-only; `/api/fs` POST switch is the right additive seam; `vfs.rename` atomic + cross-mount-refusing + clobbering; `fs.cp` the natural recursive primitive; builtin apps render inline in one document; window is CSS-transform-positioned hence the portal; Files is non-singleton so multi-instance already works). No blockers; every FR/SC/edge case is covered. The findings below are **non-blocking** and must be **baked into plan.md / tasks.md**.

**Should-fix (load-bearing for the plan):**

- **SF-1 — `onDragOver` must `preventDefault()` for the in-app VFS drag, or `drop` never fires.** The existing `onDragOver` returns early without `preventDefault()` for any drag not containing `"Files"` (`index.tsx`). A VFS drag carries the custom MIME, not `"Files"`, so under the current guard it is rejected as a drop target. The `isVfsDrag`-first ordering must be made explicit and load-bearing, and the plan must include a test asserting `drop` actually fires on **both** same-instance and cross-instance targets.
- **SF-2 — Drag-start auto-raises the source window, which fights cross-window drop routing under default geometry.** `Window.tsx` fires `onPointerDown={() => focus(win.id)}` → `os-store.focus` bumps `zIndex`; starting a drag in instance A lifts A to the front, so a drop lands on B only where B is *exposed* (not covered by the now-top A). With `launchSize` = 80% viewport and `nextLaunchOrigin` cascading by 28px, two consecutive Files windows overlap heavily. The plan must (a) include a **non-overlapping-window** e2e for the cross-instance case, and (b) record a decision: either drag-start suppresses the source raise, or the reliable cross-instance case is documented as side-by-side (non-overlapping) windows — the mockup's depiction.
- **SF-3 — Make the descendant/self-drop rule airtight and evaluate it *before* the op-decision.** §2.7's `op = (parentOf(p) === destFolder) ? "copy" : "move"` would classify "drag folder `A`, drop onto its descendant `A/sub`" as a **move** and attempt `fs.rename(A → A/sub/A)`, which fails. §2.6 already names this "no-op" but with a confusing "no-op / or same-folder copy" slash. Pin it: **drop-onto-a-dragged-folder-itself and drop-onto-a-descendant-of-a-dragged-folder are invalid → clean no-op (FR-012), checked in `resolveDest` before any op is computed.**

**Nits (fold in where cheap):**

- **N-1** — §1's "correction to the spec's wording" is a false premise: FR-013 already states Copy/Move are *new*. The conclusion (no-regression = open/rename/delete/download/open-with/upload) is correct and matches the spec; only the "correction" framing is wrong — drop the framing.
- **N-2** — The `setDragImage` ghost element must be in the rendered DOM (`position:fixed; top:-9999px`), not `display:none`, or the drag image renders blank in some browsers.
- **N-3** — Chip semantics: the mockup distinguishes **MOVED** (cross-instance) from **NEW/COPY** (created). `fresh` should carry a reason (moved vs created) so the chip matches the confirmed surface, rather than lumping both into "NEW".
- **N-4** — Destination double-refresh: the operating instance both self-refreshes (§2.8 step 1) and would receive its own `bos:files:changed`. Let the CustomEvent be the single refresh trigger for *all* instances (including the destination) to avoid a double fetch.
