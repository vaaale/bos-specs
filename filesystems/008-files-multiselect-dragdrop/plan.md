# Implementation Plan: Files — Multi-Selection & Drag-and-Drop

**Branch**: `008-files-multiselect-dragdrop` | **Date**: 2026-09-26 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/filesystems/008-files-multiselect-dragdrop/spec.md` (FR-001…FR-016, US1–US3, SC-001…SC-006).

**Note**: Technical context and structural design are in [design.md](design.md) (architect, verified by architect-reviewer: *Ready for plan*). This plan folds in the design's decisions, adds the Constitution Check, the concrete file layout, the project structure, and — critically — the **baked-in should-fix decisions (SF-1, SF-2, SF-3)** from the design review so they are explicit plan-level commitments, not left to the implementer to rediscover.

---

## Summary

Add multi-selection (Shift-click range, Ctrl/Cmd+click toggle, marquee rectangle, Select-all) and drag-and-drop move/copy — including cross-instance between two open Files windows — to the built-in Files app, with the right-click context menu applying to the whole selection. The UI surface is a **row list** (replacing the current icon-tile grid) and a **full right-click context menu** (replacing the Download-only menu), per confirmed design decisions Q1=B / Q2=A.

Technical approach (from [design.md](design.md)): extract a pure selection model, a row-list component, and a generalized context menu under `src/apps/files/`; add two collision-safe server ops (`copy`, `move`) to `vfs.ts` + `/api/fs` + `fsClient`; and exploit the fact that built-in apps render **inline in one document** to get cross-instance DnD (shared `dataTransfer`) and both-views-update (same-document `CustomEvent`) with **zero** changes to `Window.tsx`, `os-store.ts`, or `manifest.ts`.

---

## Technical Context

**Language/Version**: TypeScript (Next.js App Router + React 18; the project's existing TS)

**Primary Dependencies**: none new. Reuses `fsClient` (`src/lib/os-client.ts`), `vfs.*` (`src/os/vfs.ts`), `useOSStore` (`src/store/os-provider.tsx`), `fileHandlersClient` (`src/os/file-handlers.ts`), React state/hooks.

**Storage**: VFS (the user's sandbox, `data/vfs`) — read via existing `fsClient`/server ops. No new storage.

**Testing**: Playwright e2e (the project's `e2e/` convention) + pure-function unit tests for `selection.ts` and `dnd.ts` (Vitest/Jest per the project's existing unit runner).

**Target Platform**: the browser (BOS desktop shell).

**Project Type**: web (single-page Next.js app; this is a builtin-app change to an existing window app).

**Performance Goals**: SC-003 — move/copy a selection of up to 50 items without a perceptible UI freeze (ops are `fs.rename`/`fs.cp`; v1 fires per-item concurrently).

**Constraints**: 
- No new `package.json` deps (Constitution VII). 
- Keep all existing `data-testid` values so existing tests pass (R3). 
- The external-file **upload** drag path must remain byte-for-byte unchanged (FR-013).

**Scale/Scope**: one app restructure (`src/apps/files/`), 5 new modules, 4 modified shared files, 2 new server ops.

---

## Constitution Check

*GATE: Must pass before Phase 0. Re-checked after design — all still pass.*

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. Spec-Driven** | ✅ PASS | spec.md + design.md exist and agree; FR-013/SC-005 wording was corrected to state Copy/Move are new (done in `specify`/`design`); plan references the agreed spec as source of truth. |
| **II. Server Authority & SSR Boundary** | ✅ PASS | The new `copy`/`move` ops run server-side (`src/os/vfs.ts` + `src/app/api/fs/route.ts`); the client only sends paths over `fetch`. No secret, Node API, or fs access leaks to the client. Collision-safe naming lives on the server (ADR-2) — the single source of truth. |
| **III. Always Delegate** | ✅ PASS | Implementation is delegated to the Developer (Claude) sub-agent via `dev_delegate`; this plan is the contract it builds against. No code is written by the authoring agent. |
| **IV. Minimize Blast Radius** | ✅ PASS | Work happens on feature branch `bos/008-files-multiselect-dragdrop`; the Developer edits an isolated worktree; the change is a first-class builtin-app modification, not a runtime-state hack. |
| **V. The VFS Is Not the Source** | ✅ PASS | BOS source is changed only under `src/` by the Developer. VFS content (user files) is the *subject* of the feature, never the medium of the change. |
| **VI. Specs & Docs Stay in Sync** | ⚠️ ACTION (in `implement`) | `docs/usage` (Files app user doc) and `docs/dev` (developer doc) MUST be updated in the same change as the code — new multi-select + DnD + context-menu behaviors, and the two new `fsClient.copy/move` ops. Track as an explicit task (T-docs). |
| **VII. Respect Boundaries** | ✅ PASS | No changes to `package.json`, lockfiles, or build config (no new deps). Quality gates `npx tsc --noEmit` + `npm run lint` must pass for changed code; `npm run build` not run while `next dev` is up. |

**No violations.** No Complexity Tracking entry required.

---

## Project Structure

### Documentation (this feature)

```text
specs/filesystems/008-files-multiselect-dragdrop/
├── spec.md              # specify output (FR-001…FR-016)
├── mockup.html          # design — binding UI contract (row-list + full context menu)
├── design.md            # design — architect output (verified Ready for plan)
├── plan.md              # THIS file
├── checklists/
│   └── requirements.md  # specify quality checklist
└── tasks.md             # tasks output (next step — NOT created now)
```

### Source Code (repository root) — concrete layout, from design.md §3

```text
src/apps/files/
├── manifest.ts          # UNCHANGED (multi-instance already works; inline already default)
├── index.tsx            # MODIFIED — state owner/orchestrator: selection/fresh/uploadOver/vfsOver,
│                        #   cwd-clear effect, bos:files:changed subscription, batch handlers,
│                        #   re-dispatched drag handlers; renders <FileList> + <ContextMenu>
├── selection.ts         # NEW — Selection type + pure ops (plainClick, shiftRange, toggle,
│                        #   selectAll, marqueeSelect, prune) + uniqueSiblingName helper
├── dnd.ts               # NEW — VFS_DND_MIME, encode/decodeDragPayload, isVfsDrag/isUploadDrag,
│                        #   resolveDest (+ validity: file-row/self/descendant no-op, mid-drag guard),
│                        #   move-vs-copy op decision, buildDragImage
├── vfs-sync.ts          # NEW — window CustomEvent channel: emitVfsChanged + useVfsChanged hook
├── FileList.tsx         # NEW — row list (replaces the grid): rows, per-row gestures, marquee
└── ContextMenu.tsx      # NEW — full Q2=A menu (generalized CursorMenu): per-selection enabling

src/os/vfs.ts            # MODIFIED — add server-only copy(from,to) + move(from,to)
src/app/api/fs/route.ts  # MODIFIED — add case "copy" / case "move" to the POST switch
src/lib/os-client.ts     # MODIFIED — add fsClient.copy + fsClient.move (mirror rename shape)

# DELIBERATELY UNCHANGED (recorded so reviewers don't expect them):
#   src/apps/files/manifest.ts, src/components/desktop/Window.tsx, src/store/os-store.ts
```

**Structure Decision**: builtin-app restructure under the existing `src/apps/files/` folder + a small additive set to the shared VFS/server/client trio. No new app, no new folder tree, no marketplace item. The design's ADR-1 (native HTML5 DnD, same-document) and ADR-2 (first-class copy/move ops) are the two load-bearing structural choices.

---

## Design Notes

The full structural design lives in [design.md](design.md) (read it; this plan does not repeat it). The decisions the implementer must honor are summarized here, with the **should-fix items baked in as plan-level decisions**:

### D1 — Selection model (US1, FR-001…005, FR-014)
Pure `selection.ts` ops over an **anchor + Set** model, keyed by absolute `VfsEntry.path`, scoped to `cwd`, cleared on navigation (FR-011). Plain-click re-anchors; Shift-range measures from the *current* anchor; Ctrl/Cmd toggle leaves other paths untouched.

### D2 — Row list + gesture split (US1/US3, R2, SF-3)
`FileList.tsx` replaces the grid. **Pointer-down target is the disambiguator**: a row-origin gesture = move/copy (native `draggable`) or select (click/shift/ctrl/dblclick/contextmenu); an empty-list-space-origin gesture = marquee (pointer-captured, `position:absolute` overlay inside the scrollable list — **not** `fixed`, because window chrome is `transform`-positioned). A row is not the background, so the two paths cannot overlap.

### D3 — Context menu (US2, FR-006/007, FR-013)
`ContextMenu.tsx` generalizes `CursorMenu` (portal to `document.body` + viewport clamp preserved, `data-testid="files-context-menu"` kept). Open/Rename are single-only (greyed for >1); Copy/Move/Delete/Download are batch. Right-click on an unselected row promotes it to the selection; on a selected row it keeps the whole selection. One-item case = length-1 of the batch handlers → structurally no regression.

### D4 — Upload vs move/copy (FR-013, R3)
**Mutually exclusive by construction**: an OS file drag sets `dataTransfer.types ∋ "Files"` → the *unchanged* `uploadFiles` path; an in-app row drag sets `VFS_DND_MIME` and **not** `"Files"` → move/copy. `isVfsDrag` is checked **first** in `onDragOver`/`onDrop`.

### D5 — Move/copy ops (US3, FR-008/010/012, ADR-2)
New server ops: `copy` = collision-safe recursive `fs.cp`; `move` = collision-safe `fs.rename`. Both return the resolved destination path. Collision-safe `name (n).ext` naming is server-side (single source of truth); a retry-on-EEXIST loop guards the TOCTOU window (R4). Cross-mount is refused consistently with `rename` (R5).

### D6 — Cross-instance DnD + both-views-update (FR-015/016, SC-006, ADR-1, R1)
Native HTML5 DnD carries the payload across the two inline instances (same `dataTransfer`, browser does the cross-subtree hit-test). Post-drop, the operating instance dispatches a same-document `window` CustomEvent (`bos:files:changed`); every instance re-fetches and prunes. **No** os-store, Window.tsx, or server-bus changes.

---

## Plan-level decisions (baked-in should-fix from design review)

These are the non-blocking findings the architect-reviewer said must be baked into the plan. They are **explicit commitments here**, not left for the implementer to rediscover.

### PD-1 ← SF-1: `onDragOver` must `preventDefault()` for the VFS drag, or `drop` never fires
The existing `onDragOver` returns early without `preventDefault()` for any drag not containing `"Files"`. A VFS drag carries the custom MIME, not `"Files"`, so under the current guard it is rejected as a drop target and `drop` never fires.
**Decision**: in the re-dispatched `onDragOver`, the `isVfsDrag` branch MUST call `e.preventDefault()` (and set `e.dataTransfer.dropEffect = "move"`/`"copy"`) before returning; the `isUploadDrag` branch keeps its existing `preventDefault()`. **Test required**: an e2e asserting `drop` actually fires on **both** a same-instance target and a cross-instance target (not just that the drag ghost appears).

### PD-2 ← SF-2: Drag-start auto-raises the source window, which fights cross-window drop routing
`Window.tsx` fires `onPointerDown={() => focus(win.id)}` → `os-store.focus` bumps `zIndex`. Starting a drag in instance A lifts A to the front, so a drop lands on B only where B is *exposed*. Default geometry (`launchSize` 80% viewport, 28px cascade) makes two consecutive Files windows overlap heavily.
**Decision**: for v1, the **reliable cross-instance case is side-by-side (non-overlapping) windows** — exactly the mockup's depiction — and the e2e places the two windows so neither covers the other. The drag-start-raises-source behavior in `Window.tsx` is **not modified** (it is shared chrome; changing it is out of blast radius). A non-overlapping-window e2e is required to prove the cross-instance move/copy (SC-006). *(If a future requirement demands dragging while windows overlap, the fix is to suppress the source raise on in-app drag-start — recorded as a known limitation, not v1 scope.)*

### PD-3 ← SF-3: The descendant/self-drop rule is checked **before** the op-decision
Design §2.7's `op = (parentOf(p) === destFolder) ? "copy" : "move"` would misclassify "drag folder `A`, drop onto its descendant `A/sub`" as a *move* and attempt `fs.rename(A → A/sub/A)`, which fails.
**Decision**: `resolveDest` evaluates validity **first** and returns a clean no-op (FR-012) when the destination is (a) a file row, (b) the dragged item itself, or (c) a **descendant of any dragged folder** — *before* any move/copy op is computed. The op-decision only runs on a valid dest. **Test required**: drag a folder onto its own subfolder → no-op, folder unchanged; drag a folder onto itself → no-op.

### PD-4 ← Nits (fold in where cheap)
- **N-2** — the `setDragImage` ghost element MUST be in the rendered DOM (`position:fixed; top:-9999px`), **not** `display:none`, or the drag image renders blank in some browsers.
- **N-3** — `fresh` carries a **reason** (`"moved"` | `"created"`) so the row chip matches the mockup's MOVED vs NEW/COPY distinction, rather than lumping both into "NEW".
- **N-4** — the `bos:files:changed` CustomEvent is the **single** refresh trigger for *all* instances including the operating (destination) one; the destination does **not** also self-refresh separately (avoids a double fetch).

---

## Deliverables & Quality Gates

From the Constitution (a change is complete only when):
1. Code implements FR-001…FR-016 per design.md + this plan (D1–D6, PD-1…PD-4).
2. `docs/usage` (Files app) and `docs/dev` updated in the same change (Principle VI — task T-docs).
3. `npx tsc --noEmit` passes.
4. `npm run lint` passes.
5. e2e tests pass: selection gestures (shift/ctrl/marquee/select-all/clear), context-menu set-enabling, in-window move/copy (empty-area drop in the current folder = same-folder **copy**; invalid targets = no-op), cross-instance move/copy (non-overlapping windows, PD-2), collision-safe naming, descendant/self-drop no-op (PD-3), a **50-item** drag-drop completing within the responsiveness budget (SC-003), upload-path no-regression (D4), and the `drop`-fires-on-both-targets assertion (PD-1).
6. Existing Files `data-testid` contract intact (R3) — existing tests still pass.
