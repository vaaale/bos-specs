# Design: Fix Spec-Tree Deletion in Build Studio

**Feature**: `052-spec-tree-delete`
**Branch**: `bos/052-spec-tree-delete`
**Spec**: `spec.md` (same directory) · **Mockup**: `mockup.html` (same directory)
**App Target**: `bos-core`

---

## 1. Classification

A **bug fix with a real UI surface**. It touches two subsystems that already share one
implementation — the Build Studio tree client and the spec-lifecycle server — so the design
constrains the fix to extend those rather than fork them. The two defects are independent;
each is designed on its own, and they happen to land on the same client file and the same
server op dispatch.

| Defect | Where | Nature |
|---|---|---|
| **D1** — failed delete traps the user in a re-appearing confirm dialog | client (and surfaced server refusals) | Asymmetric dialog outcome: closes only on success |
| **D2** — folders can't be right-clicked to delete | client (menu gate) + server (no folder describe/perform op) | Container context menu limited to `project` rows; no recursive folder-delete op with a count |

---

## 2. Context (container level)

```
 Build Studio window (src/apps/build-studio/)
 ┌────────────────────────────────────────────────────────────────────┐
 │  Left tree (index.tsx: renderNode)          Centre viewer          │
 │  - store groups → projects → dirs →        - artifact markdown/html │
 │    features → files                              + error banner      │
 │  - context menus: openStoreMenu /            + persistent `error`    │
 │    openProjectMenu / openFileMenu            state, shown in the     │
 │  - ConfirmDialog (shared Dialogs.tsx)        viewer footer area      │
 │                                                                          │
 │        POST /api/specs { op }        DELETE /api/specs?path[&branch]     │
 └──────────────┬───────────────────────────────┬─────────────────────────┘
                │                               │
        src/app/api/specs/route.ts
                │                               │
        src/lib/specs/lifecycle.ts      src/lib/dev/spec-fs.ts
        (project ops, dispatched by    (specfs.remove — ALREADY recursive
         store kind; resolveOpStore,    fs.rm + prepareWrite + commitOnSave;
         assertHasProjects)             resolveInStore, prepareWrite,
                                        assertEditablePath in pipeline.ts)
```

Key facts the design relies on (all confirmed in source):

- `specfs.remove(path, { branch })` does `fs.rm(abs, { recursive: true, force: false })`
  **then** `commitOnSave`. It already removes a directory recursively. It runs
  `prepareWrite` (branch gate), but the **frozen-section gate** (`assertEditablePath`) is
  applied by the *route* (`DELETE` handler), not inside `specfs.remove`.
- The POST route dispatches `create-project` / `rename-project` / `delete-project` /
  `describe-project-deletion` to `lifecycle.ts`. `delete-project` refuses unless
  `confirm: true` (else 409 + the count) — the discipline we extend.
- The client's `deleteFileAction.onConfirm` closes the dialog **only after the `r.ok`
  check passes**; on failure it `setDialogError(msg)` and leaves the dialog open.
  `ConfirmDialog` renders `message + "\n\n" + dialogError` and re-enables Delete once
  `dialogBusy` clears → the loop.
- `renderNode` wires the container row's menu as
  `onContextMenu={node.type === "project" ? (e) => openProjectMenu(e, node) : undefined}`
  → plain `dir` and `feature` rows have no menu.
- The tree already computes, per store group, `group.liveBranch` (the real writable branch
  this group is on) and renders a "not on this branch" badge when it is empty — the data
  the D1 pre-refusal needs is already present.

---

## 3. Component design

### 3.1 D1 — make delete outcomes symmetric (client)

**Change**: in `BuildStudioApp.deleteFileAction.onConfirm`, treat failure exactly like
`lifecycleOp` already does for project ops — dismiss the dialog **and** surface the reason
in the persistent `error` banner. Do not `setDialogError` and leave the box up.

```
onConfirm:
  setDialogBusy(true); setDialogError("");
  try {
    r = DELETE /api/specs?path[&branch]
    if (!r.ok) throw Error(res.error)
    setConfirmDialog(null)                 // success → close
    if (activePath === node.path) setActivePath("")   // FR-011
    loadTree()
  } catch (e) {
    setConfirmDialog(null)                 // FAILURE → close (was: leave open)
    setError(e.message)                    // FAILURE → banner once (was: dialogError only)
  } finally { setDialogBusy(false) }
```

This is the unconditional half of the fix. It does not depend on *why* the server refused:
whichever refusal fires (branch-not-mounted, frozen section, read-only), the box closes and
the reason is shown once, out of a dialog the user can get out of. FR-001/FR-002/FR-011.

### 3.2 D1 — refuse before the round-trip for nodes the branch doesn't cover (client)

**Rationale**: the most likely *server* cause of the original loop is the branch/coupling
refusal (`"Store … is not mounted on branch …"` / `BranchRequiredError`). The tree already
knows, per store, whether it is on the active branch (`group.liveBranch`). A request we
already know will be refused is wasted and, before 3.1, was the thing that trapped the user.
**Change**: when building the file/folder menu, if the node's store is not on the active
branch, render *Delete…* **disabled with a hint** (matching the existing disabled-action
pattern) instead of sending a guaranteed refusal. If a branch *is* selected and the store is
on it, enable as usual (3.1 still covers the residual failure cases). FR-010 is unchanged —
no branch at all still routes through the existing branch-elicitation flow.

> **Honesty note**: the exact server string behind the original loop was not pasted by the
> reporter. 3.2 is a *client-side* pre-refusal driven by data the tree already holds, so it
> does not depend on that string; it removes the guaranteed-to-fail request for the
> branch-uncoupled case. The residual, string-specific server fix (e.g. a clearer message
> for a frozen section) is a small follow-on, not a blocker — 3.1 already makes any such
> refusal non-entangling.

### 3.3 D2 — offer a context menu on every writable container (client)

**Change**: generalize the container-row menu gate in `renderNode` from
`node.type === "project"` to **writable containers**: `project | dir | feature`. Read-only
stores (`group.writable` falsy) and item-owned stores keep no Delete (an item is an
uninstall, FR-009). Rename stays available only where it already applied (project); the new
requirement is **Delete…** on all three writable container kinds (FR-004/FR-005).

```
onContextMenu={isWritableContainer(node) ? (e) => openContainerMenu(e, node) : undefined}
```

`openContainerMenu` is `openProjectMenu` generalized: it resolves the node's store
(`storeIdOf`), bails on non-writable, and builds the menu. For a `project` it is exactly
today's menu (Rename… + Delete… via the existing `delete-project` op). For a `dir` or
`feature` it offers **Delete…** wired to the *folder* ops (3.4), plus View history (a read,
already meaningful for any path).

### 3.4 D2 — a recursive folder-delete op with a confirm count (server)

**Change**: add two ops to the POST `/api/specs` dispatch and `lifecycle.ts`, mirroring the
existing `delete-project` / `describe-project-deletion` pair, addressed by **store-prefixed
path** (not project id, since a plain `dir` is not a project):

- `describe-dir-deletion` `{ path }` → `{ deletion: { label, files, units } }`. Walks the
  store subtree under `path` (via `specfs.listDir` recursion or a bounded walk) and counts
  files and feature units. Read-only; used to fill the confirm dialog (FR-006).
- `delete-dir` `{ path, confirm }` → performs the recursive remove. Refuses unless
  `confirm: true` (else 409 + the count), applies `assertEditablePath(path)` (frozen gate)
  and the branch gate, then `specfs.remove(path, { branch })` (already recursive + commit)
  (FR-007/FR-008).

`lifecycle.ts` gains `describeDirDeletion(storePrefixedPath)` and
`deleteDirIn(storePrefixedPath, ctx)`. They resolve through `resolveOpStore`/
`resolveInStore` so they inherit marketplace synthesis and the same routing as every other
op — keeping the module's stated principle: **one implementation, the agent tools and the
Build Studio menus are two paths to the same outcome.**

> Why not just reuse `DELETE /api/specs?path=<folder>` (it's already recursive)? Because it
> has no describe/count round-trip and no confirm-required 409 guard, and it bypasses the
> lifecycle module the destructive project ops go through. Folding folder-delete into the
> same op pair as project-delete keeps destructive ops on one code path and one
> confirmation discipline — which is the exact divergence this subsystem keeps producing.

### 3.5 D2 — wire the menu item to the folder flow (client)

**Change**: *Delete…* on a `dir`/`feature` opens the same confirm-with-count UX that
*Delete…* on a Project uses — dialog opens immediately ("Checking what this would
remove…"), `describe-dir-deletion` fills in the count (`dialogBusy` keeps Confirm disabled
until it arrives), confirm calls `delete-dir { confirm: true }`. Success closes and
`loadTree()`; failure routes through 3.1's symmetric handling. FR-006/FR-011.

---

## 4. File / module plan (what this feature creates or modifies)

| File | Change | Defect |
|---|---|---|
| `src/apps/build-studio/index.tsx` | `deleteFileAction.onConfirm` → symmetric success/failure (close on failure + banner); `renderNode` container menu gate → `isWritableContainer`; `openProjectMenu` → `openContainerMenu` handling `project \| dir \| feature`; Delete… pre-disabled when store not on active branch; wire folder Delete to describe/perform flow | D1 + D2 |
| `src/lib/specs/lifecycle.ts` | `describeDirDeletion(path)`, `deleteDirIn(path, ctx)` (store-prefixed-path addressed; count; confirm-gated; reuses `specfs.remove`) | D2 |
| `src/app/api/specs/route.ts` | POST dispatch for `describe-dir-deletion` and `delete-dir` (mirrors `delete-project`/`describe-project-deletion`, including the 409-unconfirmed guard and `assertEditablePath`) | D2 |

**Created**: none beyond the spec-kit artifacts in this directory. No new component file,
no new API surface beyond the two ops above.

---

## 5. Integration points (existing mechanisms this merely calls into)

- `specfs.remove` / `specfs.listDir` / `resolveInStore` / `prepareWrite` — `src/lib/dev/spec-fs.ts`
- `assertEditablePath` (frozen/terminal gate) — `src/lib/specs/pipeline.ts`
- `resolveOpStore` (marketplace synthesis) — `src/lib/specs/lifecycle.ts`
- `ConfirmDialog`, `PromptDialog`, `ContextMenu` — `src/apps/build-studio/Dialogs.tsx`
- `withFeatureBranch` / `suggestFeatureBranchName` (branch elicitation) — already on the menu path
- `lifecycleOp`'s busy/error/banner discipline — the client pattern 3.1 copies

---

## 6. Decisions (ADRs)

- **ADR-1 — Fix D1 in the client, not by guessing the server string.** The trap is a
  client-side dialog contract ("close only on success"). Making success/failure symmetric
  removes the loop for *every* server refusal and is testable without reproducing the
  exact server error. Chosen over "patch the server message first", which would leave the
  client able to trap the user on any other refusal.
- **ADR-2 — Folder delete is a new op pair, not a reuse of `DELETE /api/specs`.** Keeps all
  destructive spec-tree ops on the one lifecycle code path with one confirm/count
  discipline. Reusing the bare DELETE verb would silently drop the count and the
  unconfirmed-409 guard that `delete-project` has.
- **ADR-3 — Address folder ops by store-prefixed path, not project id.** A plain `dir` is
  not a project; the only thing the UI has is the node's path. Project ops keep their id
  addressing; folder ops are path-addressed. The two coexist in one dispatch.
- **ADR-4 — Pre-disable Delete when the store isn't on the active branch (client).** Uses
  data the tree already renders (`group.liveBranch`) to avoid a guaranteed-to-fail
  round-trip, matching the existing disabled-action pattern. Not a blocker on the missing
  server string.
- **ADR-5 — Confirmation always required, even for a 0-count folder.** Destructive ops are
  uniformly confirmed; no fast-path (SC-004 / Story 3).

---

## 7. Risks & open questions

- **R1 — exact server string for the original loop is unconfirmed.** Mitigated: 3.1 makes
  any refusal non-entangling, and 3.2 pre-refuses the branch-uncoupled case from local data.
  A clearer message for a specific residual refusal is a small follow-on.
- **R2 — deleting the folder that holds the file open in the centre viewer.** FR-011
  requires the viewer not show a stale artifact: `deleteFileAction`/folder flow must clear
  `activePath` when the removed path is the open one (or an ancestor of it — the design
  clears on the deleted path *or* any ancestor containing it).
- **R3 — count walk cost on very large folders.** Bounded: it's a directory listing of the
  spec subtree, not a content read; acceptable at spec-tree scale. If a folder is huge, the
  count still completes well within the dialog's "checking…" window.
- **Open (for the human, optional)**: should a `feature` folder's delete also refuse when
  the feature has an in-flight MR/promotion? Out of scope here (spec says deletion is a
  spec-store op, branch-coupled); flagged so it isn't silently assumed away.
