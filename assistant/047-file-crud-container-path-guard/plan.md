# Implementation Plan: Wire rejectContainerPath into the VFS CRUD file tools

**Branch**: `bos/self-heal-0053` | **Date**: 2026-09-29 | **Spec**: `/Specs/user-specs/assistant/047-file-crud-container-path-guard/spec.md`

**Input**: Feature specification from the spec above (self-heal case 0053).

## Summary

The VFS file tools in `src/lib/assistant/tools/server/files.ts` already carry a shared guard, `rejectContainerPath(path)`, whose docstring promises it is the "shared guard for every VFS file tool." It is wired into the five search/edit tools (`file_edit`, `file_patch`, `file_search`, `file_grep`, `file_glob`) but is absent from the six CRUD tools (`file_read`, `file_list`, `file_write`, `file_mkdir`, `file_delete`, `file_rename`). As a result, a real on-disk container path handed to a CRUD tool fails with a VFS-shaped error (a bare `ENOENT`, an empty listing, a no-op "Created folder …") that reads as a typo, so a capable agent retries the identical wrong call — the exact 2× loop observed in case 0053 on `file_read`.

The fix is to wire the existing guard into the six CRUD handlers, before the `vfs.*` call, exactly as the search/edit tools already do. No new mechanism, no new file type, no changes to the guard/predicate/diagnostic themselves. The change is confined to one source file plus its regression test.

## Technical Context

**Language/Version**: TypeScript (BOS's Next.js App Router codebase, Node server runtime).

**Primary Dependencies**: none new — reuses the module's own `rejectContainerPath`, `isRealContainerPath`, `containerPathDiagnostic`, the `inScope`/`withFeatureScope` wrapper, and the `@/os/vfs` module. No new imports.

**Storage**: N/A (VFS file tools; no new persistence).

**Testing**: The repo's existing test framework (Node/TypeScript unit tests under `src/lib/assistant/tools/server/__tests__/`). The guard functions `isRealContainerPath` and `containerPathDiagnostic` are already `export`ed from `files.ts`, so a test can import them directly and assert on the exact diagnostic string.

**Target Platform**: Node server (the file tools are `serverTool`s; the `server-only` import at the top of `files.ts` enforces this).

**Project Type**: single-project web service (Next.js). This is a server-side tool handler change.

**Performance Goals**: none — a single synchronous predicate check at the top of each handler (string split + `Set.has`), negligible.

**Constraints**:
- The guard MUST run inside each tool's existing feature-scope wrapper (`inScope`), before the `vfs.*` call (FR-007).
- A clean VFS path MUST be unaffected — identical observable behavior to today (FR-008).
- `rejectContainerPath` / `isRealContainerPath` / `containerPathDiagnostic` MUST NOT be modified (FR-010).
- `file_rename` checks `path` first, then `to`; first container path wins (FR-006).

**Scale/Scope**: 6 CRUD tool handlers in 1 file + 1 new test file.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **I. Spec-Driven** — PASS. `spec.md` exists and is the source of truth; this plan references it.
- **II. Server Authority & SSR Boundary** — PASS. The change is server-side (the file tools are `serverTool`s behind the `server-only` import); no client/secret exposure.
- **III. Always Delegate; Claude Codes** — PASS. Implementation is delegated to the Developer (Claude) sub-agent via `dev_delegate`; Build Studio authors only spec/plan/tasks.
- **IV. Minimize Blast Radius** — PASS. The change lands on the existing feature branch `bos/self-heal-0053` in an isolated worktree; it touches one source file + one test file. Promote is the user's explicit action (FR-024).
- **V. The VFS Is Not the Source** — PASS. This feature edits `src/` (BOS source) via the developer sub-agent — the correct mechanism; it does not route BOS-source edits through VFS file tools.
- **VI. Specs & Docs Stay in Sync** — PASS (conditioned). `spec.md`/`plan.md`/`tasks.md` are authored here. The Developer MUST search `docs/dev/` and `docs/usage/` for any page describing the current per-tool container-path behavior and update it in the same change (FR-011); if none exists, verify-by-search satisfies it.
- **VII. Respect Boundaries** — PASS. No secrets, `package.json`, lockfile, or build-config change. The change is to a server tool handler, not to `tools/supervisor/**` or build config (FR-023) — self-heal is permitted to touch this surface.

**No violations. No Complexity Tracking entry required.**

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/assistant/047-file-crud-container-path-guard/
├── spec.md          # Feature specification (authored)
├── plan.md          # This file
└── tasks.md         # Task list (authored in the tasks step)
```

### Source Code (repository root)

The fix modifies exactly one existing source file and adds one test file:

```text
src/lib/assistant/tools/server/
├── files.ts                        # MODIFY — add rejectContainerPath(path) to the six CRUD handlers
└── __tests__/
    └── files.test.ts               # ADD — regression test: container path → diagnostic; clean path → no-op, for each CRUD tool
```

**Structure Decision**: Single-project layout (BOS is a single Next.js repo). No new directory, no new module. The test file lands in the module's existing `__tests__/` directory, consistent with the sibling `gitlab-feature-branch.test.ts` that already lives there.

### Hard scope constraint (FR-014)

The plan's file list — the ONLY files the Developer may modify — is:

1. `src/lib/assistant/tools/server/files.ts`
2. `src/lib/assistant/tools/server/__tests__/files.test.ts` (new)
3. *(conditional, Constitution VI)* any existing page under `docs/dev/` or `docs/usage/` that describes the current per-tool container-path behavior, identified by the Developer's `docs/` search. If the search finds none, this item contributes no file and the Developer records "no docs page to update" in the report.

If the fix requires modifying ANY file outside this list, the Developer MUST stop and report the justification for each additional file rather than modifying it.
