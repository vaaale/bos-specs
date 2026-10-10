# Implementation Plan: Fix Spec-Tree Deletion in Build Studio

**Branch**: `bos/052-spec-tree-delete` | **Date**: 2026-09-29 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/Specs/user-specs/build-studio/052-spec-tree-delete/spec.md`

## Summary

Fix two Build Studio spec-tree deletion defects in one change:

1. **D1** — a failed file delete loops: the `ConfirmDialog` only closes on success, so a
   server-side refusal re-fires the identical request on every Ok (the reported infinite
   confirm loop). The client's delete outcome handling is made **symmetric** — success and
   failure both close the dialog; failure additionally posts its reason **once** to the
   persistent error banner (FR-001/FR-002/FR-003).
2. **D2** — folders can't be right-clicked to delete: the container context menu is gated to
   `project` rows only. The gate is generalized to every writable container
   (`project | dir | feature`), and a new **folder-delete op pair**
   (`describe-dir-deletion` / `delete-dir`) is added to the spec-lifecycle server, mirroring
   the existing `describe-project-deletion` / `delete-project` confirm-and-count discipline
   (FR-004…FR-009).

Technical approach: extend, don't fork. The design (`design.md`, sections 3.1–3.5) rides the
existing `lifecycleOp` client pattern, the existing `ConfirmDialog`/`ContextMenu` components,
and `specfs.remove` — which is *already* recursive and already commits on save. No new
protocol, daemon, or component file is introduced.

## Technical Context

**Language/Version**: TypeScript (BOS repository, Next.js 15 / React 18)

**Primary Dependencies**: none new — existing `fs` (node), Next.js App Router route handlers,
BOS spec-lifecycle library

**Storage**: git-backed spec stores (VFS-mounted at `/Specs/*`); deletion is
`fs.rm(abs, { recursive: true, force: false })` + `commitOnSave` (existing
`specfs.remove` in `src/lib/dev/spec-fs.ts`)

**Testing**: Vitest unit (existing test co-located pattern for `lifecycle.ts` / route handler);
manual verification of the client flow against the mockup as binding UI contract

**Target Platform**: BrowserOS desktop shell (React client) + Next.js server routes

**Project Type**: web application (existing — in-repo fix, no new project)

**Performance Goals**: count walk is a directory listing only (bounded; spec-tree scale, R3)

**Constraints**:
- All destructive spec-tree ops stay on the one lifecycle code path with one confirm/count
  discipline (ADR-2)
- Folder ops are addressed by **store-prefixed path**, not project id (ADR-3)
- No Delete… offered in read-only stores or for marketplace-item rows (FR-009)
- Confirmation always required, even for a 0-count folder (ADR-5)

**Scale/Scope**: 3 files modified, 2 new server ops, 0 new files (code)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`/Specs/bos-system-specs/.specify/memory/constitution.md` is **not readable on the active
branch** — the `bos-system-specs` store mount is empty in this worktree. No constitution
gates could therefore be evaluated; this is recorded here rather than asserted as passing.
No gate failures can be justified because none were found.

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/build-studio/052-spec-tree-delete/
├── spec.md              # Feature specification (source of truth)
├── design.md            # Architect design (classification, ADRs, file plan)
├── plan.md              # This file
├── mockup.html          # Binding UI contract for the tree + dialogs
└── tasks.md             # Phase 2 output (__SPECKIT_COMMAND_TASKS__ — NOT created here)
```

No `research.md` / `data-model.md` / `contracts/` companions are warranted: the unknowns
were resolved during the design step (all source facts confirmed in `design.md` §2), there
are no new entities (only two new op payloads on an existing dispatch), and the contracts
are internal API ops whose shape is pinned by the design (§3.4) and the mockup.

### Source Code (BOS repository root)

```text
src/
├── apps/build-studio/
│   └── index.tsx            # MODIFIED — D1 + D2 client (see design.md §4)
├── lib/specs/
│   └── lifecycle.ts         # MODIFIED — describeDirDeletion, deleteDirIn (D2)
└── app/api/specs/
    └── route.ts             # MODIFIED — POST dispatch for describe-dir-deletion / delete-dir (D2)
```

**Structure Decision**: in-repo modification of the three files named in `design.md` §4.
No new component file, no new API surface beyond the two ops, no new store or branch
topology.

## Design Notes (folded from `design.md`)

- **Classification**: bug fix with a real UI surface; two independent defects (D1 client
  dialog contract, D2 menu gate + missing server ops) landing on the same client file and
  the same server op dispatch.
- **Component design**:
  - *3.1* `deleteFileAction.onConfirm` → symmetric: both success and failure call
    `setConfirmDialog(null)`; failure adds `setError(e.message)` once (mirrors
    `lifecycleOp`'s existing banner discipline).
  - *3.2* Pre-refusal: when a node's store is not on the active branch
    (`group.liveBranch` empty — data the tree already renders), Delete… is rendered
    **disabled with a hint** instead of sending a guaranteed refusal (ADR-4).
  - *3.3* `renderNode` gate `node.type === "project"` → `isWritableContainer(node)`
    (`project | dir | feature` in a writable store); `openProjectMenu` →
    `openContainerMenu`. Rename stays project-only; Delete… extends to all three.
  - *3.4* Server: `describe-dir-deletion { path }` → count of files/feature units
    (read-only); `delete-dir { path, confirm }` → 409 + count unless `confirm: true`,
    then `assertEditablePath(path)` + `specfs.remove(path, { branch })`. Both resolve via
    `resolveOpStore`/`resolveInStore`, inheriting marketplace synthesis and routing.
  - *3.5* Client wiring: folder Delete… opens the same confirm-with-count UX as Project
    Delete… ("Checking what this would remove…" → `describe-dir-deletion` fills the count →
    confirm → `delete-dir { confirm: true }`), success/failure through 3.1.
- **ADRs** (all in `design.md` §6): ADR-1 fix D1 in the client; ADR-2 new op pair rather
  than bare `DELETE /api/specs`; ADR-3 store-prefixed-path addressing; ADR-4
  client-side pre-refusal for branch-uncoupled stores; ADR-5 confirmation always required.
- **Risks** (design.md §7): R1 exact original server string unconfirmed (mitigated by 3.1+3.2);
  R2 stale centre-viewer artifact when the open file's ancestor is deleted (FR-011: clear
  `activePath` on the deleted path *or any ancestor of it*); R3 count-walk cost (bounded
  listing). Open question left for the human: refusing a `feature` delete while its MR/
  promotion is in flight — out of scope, flagged not assumed.

## Complexity Tracking

Not applicable — no constitution violations to justify.
