# Implementation Plan: File-Type Handler Registry & "Open With"

**Branch**: `036-file-type-handlers` | **Date**: 2026-08-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/Specs/user-specs/core-platform/036-file-type-handlers/spec.md`

## Summary

Introduce a platform-level **file-type-handler registry** so apps declare (in their manifest) which MIME types they can render and/or edit. The Files app becomes the first consumer: double-click opens a file with the selected (default) handler for its type; the right-click context menu lists every registered handler as "Open with \<App\>". The built-in `html-viewer` (web_view) is registered as the default `text/html` renderer via its manifest declaration.

**Architecture**: see [design.md](./design.md) — 8 ADRs, component/container/context diagrams, the open-file launch contract (§6), and the concrete file/module plan (§4).

## Technical Context

**Language/Version**: TypeScript (strict), React 19, Next.js 15 (App Router)

**Primary Dependencies**: none new — uses existing `os-store.launch`, `listInstalledManifests`, `writeFileAtomic`, `dataDir`, the existing MIME map, and the existing `IframeApp`/`Window` delivery split.

**Storage**: `data/system/file-handlers.json` (new, atomic JSON writes via `writeFileAtomic`). Branch-isolated via `dataDir()` / `BOS_DATA_DIR`.

**Testing**: No explicit test framework requirement in the spec. Verification is via the acceptance scenarios in spec.md (manual / Playwright e2e at `implement` time). `npx tsc --noEmit` + `npm run lint` are the quality gates (Constitution VII).

**Target Platform**: BrowserOS desktop (single Next.js process, no worker threads, no service facet)

**Project Type**: Web application (Next.js) — modifications to existing built-in apps + new server modules + new API route

**Performance Goals**: Handler list resolved in <100ms per context-menu open (a few manifest reads; immaterial at current app count)

**Constraints**:
- No per-app special cases in core code (FR-013, SC-004)
- No-handler path must be byte-for-byte the existing behavior (SC-003)
- Client code cannot import `server-only` modules (Constitution II; ADR-7)
- No new external dependencies

**Scale/Scope**: 11 files (8 new/modified in `src/`, 1 new API route, 1 new data file, docs). 1 new manifest field. 1 new API endpoint (2 methods).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status |
|---|---|
| **I. Spec-Driven** | ✅ spec.md → design.md → plan.md → tasks.md pipeline. |
| **II. Server Authority & SSR Boundary** | ✅ Server: registry + selection + API route (`server-only`). Client: pure param assembly in `src/os/file-handlers.ts` (framework-free) + `launch` store call. No secrets. (ADR-7, design.md §2) |
| **III. Always Delegate** | ✅ Implementation via `dev_delegate` on feature branch. |
| **IV. Minimize Blast Radius** | ✅ Feature branch. Registry is stateless/derived (ADR-1). Files-app change is strictly additive. No new mutable global. |
| **V. VFS Is Not the Source** | ✅ Selection state in `data/system/`, not VFS, not `src/`. |
| **VI. Specs & Docs Stay in Sync** | ✅ Polish phase updates `docs/dev/` (design.md §2 flags this). |
| **VII. Respect Boundaries** | ✅ No `package.json`/lockfile/build-config changes. `tsc --noEmit` + `lint` gates. |

**No conflicts.** No Complexity Tracking entry needed.

## Project Structure

### Documentation (this feature)

```text
/Specs/user-specs/core-platform/036-file-type-handlers/
├── spec.md          # Feature specification
├── design.md        # Architecture (ADRs, component plan, launch contract)
├── plan.md          # This file
├── mockup.html      # UI mockup (Files app context menu, two states)
└── tasks.md         # Implementation tasks
```

### Source Code (repository root)

```text
src/
├── os/
│   ├── types.ts                    # MOD: + AppFileHandlerDeclaration, AppManifest.fileHandlers?
│   └── file-handlers.ts            # NEW: framework-free, client-safe (baseMime, mimeForPath, fileBaseMime, rawUrlFor, buildLaunchParams)
├── lib/
│   ├── mime.ts                     # MOD: re-export from src/os/file-handlers.ts (3 existing importers untouched)
│   ├── os-client.ts                # MOD: + fileHandlersClient (list, setSelected)
│   ├── apps/
│   │   └── store.ts                # MOD: mirror fileHandlers through InstalledApp (type, readApp, toManifest)
│   └── file-handlers/
│       ├── registry.ts             # NEW: server-only (matchDeclared, handlersFor, effectiveSelected)
│       └── selection.ts            # NEW: server-only (readSelection, writeSelection → data/system/file-handlers.json)
├── app/
│   └── api/
│       └── file-handlers/
│           └── route.ts            # NEW: GET ?mime= → handler list; POST → set/clear selection
├── components/
│   └── apps/
│       └── IframeApp.tsx           # MOD: + withFileParams (sibling to withEventParams)
└── apps/
    ├── html-viewer/
    │   └── manifest.ts             # MOD: + fileHandlers declaration (text/html, render, default, label "Web View")
    └── files/
        └── index.tsx               # MOD: double-click → selected handler; right-click → "Open with" group

data/
└── system/
    └── file-handlers.json          # NEW (runtime): per-type user selection { "<baseMime>": "<appId>" }
```

**Structure Decision**: Follows the existing BOS layout exactly. New server code under `src/lib/file-handlers/` (mirrors `src/lib/events/` for spec 034). New API route at `src/app/api/file-handlers/` (standard App Router). Shared client-safe helpers at `src/os/file-handlers.ts` (framework-free convention, same as `src/os/types.ts`). No new project/package boundaries.

### Docs to update (Polish phase)

```text
docs/dev/
├── apps/
│   ├── built-in-apps.md            # MOD: document the fileHandlers manifest field
│   └── file-handlers.md            # NEW: the open-file launch contract (for handler app authors)
└── extending-bos.md                # MOD: "Add a file handler" recipe
```

## Complexity Tracking

No Constitution violations. No entry needed.

## Design Notes

All architectural decisions are in [design.md](./design.md). Key references for implementers:

- **§4** — concrete file/module plan (the table above is extracted from it)
- **§5** — integration points (existing BOS mechanisms called but not modified)
- **§6** — the open-file launch contract (FR-014): the contract, the `paramShape` mapping, and the two delivery paths
- **§7** — ADRs 1–8 (registry derivation, delivery split, paramShape vocabulary, MIME normalization, selection persistence, edit-only semantics, SSR boundary, authoritative MIME map)
- **§8** — risks (R-2 withFileParams clobber, R-3 launch null, R-4 hidden apps) and open questions (O-1 edit-only selection, O-2 follow-on item contract)
- **§9** — UI mockup reference (two frames mapping to the context menu design)
