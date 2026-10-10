# Implementation Plan: Settings cleanup (041-settings-cleanup)

**Branch**: `bos/041-settings-cleanup` | **Date**: 2026-10-08 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/Specs/user-specs/core-platform/041-settings-cleanup/spec.md`

## Summary

Three Settings cleanups in one pass, driven by GitLab issue #58 ("Remove unused page in Settings"):

- **(A) US1 — P1:** Remove the dead "System Tools" Settings page and its orphaned API route + server lib. The page's toggle gates a legacy unsandboxed `bash -lc` endpoint that nothing calls; real command execution is the separate Command Execution page (`run_command`), which is untouched.
- **(B) US2 — P2:** Remove the "Knowledge bases" tab from the per-agent Settings editor (`AgentDetails.tsx`). The Knowledge Base *item* and its `kb_search`/`kb_retrieve` MCP tools and the per-agent `kbs` allowlist data model all remain — only the Settings UI affordance is deleted.
- **(C) US3 — P3:** Sort the Settings sidebar "modules" A→Z by display title (case-insensitive, namespace tie-break) instead of the numeric `order` field. Applies to built-in registrations and plugin-contributed panels together, with no client-side change.

**Technical approach:** A deletions-plus-one-re-sort change. (A) and (B) delete code whose sole consumer is being removed — safe iff the orphan set is closed, which `design.md` verifies against live source. (C) changes the comparator in one server function, `listConfigSchemas()` (`src/lib/config/registry.ts`), which is the single point where built-in and plugin panels are merged and ordered and which the Settings sidebar consumes as-is. No new files, no new data model, no new API route, no daemon, no protocol.

## Technical Context

**Language/Version**: TypeScript (strict), as the existing BOS codebase.

**Primary Dependencies**: Next.js (App Router) + React + Zustand — existing; no new dependencies introduced.

**Storage**: N/A. The persisted `data/config/system-tools.json` is deliberately left in place and inert (FR-004 / ADR-3) — no migration, no cleanup, no new file.

**Testing**: `npx tsc --noEmit` (typecheck) + `npm run lint` (quality gates per Constitution VII). Verification is a repo-wide `grep` over `src/` for the removed identifiers (FR-015 / SC-003) plus manual confirmation of the Settings sidebar in a running build (SC-001). No new automated test files are specified.

**Target Platform**: The existing BrowserOS Node/Next.js server + browser client.

**Project Type**: web-service + SPA (existing); this change only edits/deletes files within it.

**Performance Goals**: N/A — the comparator change affects ~21 modules sorted once per `GET /api/config`; no measurable impact.

**Constraints**: Build and typecheck must be clean with no dangling references (FR-016 / SC-004). `run_command`, the Command Execution page, and the Knowledge Base item must behave exactly as before (SC-004).

**Scale/Scope**: 6 source files touched — 2 modified in place, 1 modified (the barrel), 3 deleted. No new files.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked post-design — no new concerns introduced.*

| Principle | Status | Note |
|---|---|---|
| I. Spec-Driven | ✅ | Spec approved; plan references `design.md`; work follows the pipeline. |
| II. Server Authority & SSR Boundary | ✅ | Changes are in server lib (`src/lib/config/registry.ts`, deleting `src/lib/system/bash.ts`), a server API route (deleting `src/app/api/system/bash/route.ts`), and Settings UI components. No secret/Node-API surface added. |
| III. Always Delegate; Claude Codes | ✅ | `implement` will delegate to the Developer sub-agent; Build Studio writes only this plan. |
| IV. Minimize Blast Radius | ✅ | All changes on feature branch `bos/041-settings-cleanup`; isolated worktree; deletions only shrink the surface. |
| V. The VFS Is Not the Source | ✅ | Source changes go through the Developer on `src/`, never VFS file tools. |
| VI. Specs & Docs Stay in Sync | ✅ | No end-user or dev doc describes the removed System Tools page or KB tab as a supported feature to preserve — confirm at `converge`; record any drift in `discrepancies.md`. |
| VII. Respect Boundaries | ✅ | No `package.json`, lockfile, or build-config change. Typecheck + lint gates apply to changed code. |

**Gates: PASS.** No violations to justify.

## Project Structure

### Documentation (this feature)

```text
/Specs/user-specs/core-platform/041-settings-cleanup/
├── spec.md              # Feature specification (user stories + FRs)
├── design.md            # Architectural design + verified orphan set + ADRs
├── plan.md              # This file
├── mockup.html          # UI mockup (Settings sidebar A→Z, no System Tools / KB tab)
└── tasks.md             # Task breakdown (next step)
```

No `research.md`, `data-model.md`, `contracts/`, or `quickstart.md` — none are warranted. The design step already resolved every open question and verified the orphan set against live source; there is no new data model or external interface, and the success criteria in `spec.md` double as the validation checklist.

### Source Code (repository root) — only what this feature touches

```text
src/
├── lib/
│   ├── config/
│   │   └── registry.ts                 # MODIFY: delete system-tools registration (A);
│   │                                   #         change listConfigSchemas() comparator (C)
│   └── system/
│       └── bash.ts                     # DELETE (A) — orphaned by the route deletion
├── app/
│   └── api/
│       └── system/
│           └── bash/
│               └── route.ts            # DELETE (A) — legacy unsandboxed bash endpoint
└── components/
    └── apps/
        └── settings/
            └── assistant/
                ├── AgentDetails.tsx    # MODIFY: remove "kb" tab + KnowledgeBasesGrid usage (B)
                ├── KnowledgeBasesGrid.tsx  # DELETE (B) — orphaned by the tab removal
                └── index.ts            # MODIFY: remove the two KnowledgeBasesGrid re-exports (B)

# Left in place, untouched (intentionally inert / out of scope):
data/config/system-tools.json           # inert after the registration is removed (A / ADR-3)
src/lib/agent/kb-catalog.ts             # Knowledge Base item runtime (B / ADR-2) — unchanged
src/lib/agent/instructions.ts           # buildKbIndexBlock — unchanged
```

**Structure Decision**: No new directories or files. All changes are in-place edits to two existing server components + one barrel, plus three file deletions, concentrated in the three areas the spec scopes. `design.md` §"Component / change plan" lists the exact edit per file.

## Design Notes

Full architecture, the verified orphan set, and the three ADRs live in [design.md](./design.md) and are referenced, not repeated here. The load-bearing points for implementation:

- **ADR-1 (the only behavioural change)**: `listConfigSchemas()` replaces its `(a.order ?? 100) - (b.order ?? 100)` comparator with `a.title.localeCompare(b.title, undefined, { sensitivity: "base" }) || a.namespace.localeCompare(b.namespace, undefined, { sensitivity: "base" })`. The `order` field stays on each `ConfigSchema` as inert data. This is the single edit that makes the sidebar A→Z for built-ins and plugin panels together with no client change (FR-011–FR-014 / SC-005).
- **Orphan closure (A, B)**: before deleting, the Developer confirms the `grep` targets in FR-015 return no hits — `system-tools`, `@/lib/system/bash`, `runBash`, `BashResult`, `api/system/bash`, `KnowledgeBasesGrid` — so a deletion does not strand a live import (FR-016).
- **Preservation boundaries (B / ADR-2)**: the `kbs` allowlist model, `kb-catalog.ts`, `buildKbIndexBlock`, and the agent API's `catalog.kbs` payload are explicitly untouched (FR-009/FR-010). Only the Settings tab and its component are removed.
- **No migration (A / ADR-3)**: `data/config/system-tools.json` is left in place; nothing reads it after the registration is removed.

## Complexity Tracking

No Constitution violations — nothing to justify.
