# Implementation Plan: Workflow Manager Service Tools (039-Compliant)

**Branch**: `bos/001-workflow-manager-service-tools` (per-project under Workflow Manager) | **Date**: 2026-08-20 | **Spec**: `user-specs/workflow-manager/001-workflow-manager-service-tools/spec.md`

**Input**: Feature specification + `design.md` (full structural design, verified by architect-reviewer MF-1..MF-3 / SI-1..SI-3, revised once). This plan **references** `design.md` rather than restating it — the design is the authoritative structural source. ADRs below are folded in from `design.md` §6.

---

## Summary

Rewrite the Workflow Manager marketplace item (app id `workflows`) so its workflow tools are exposed as **service-declared native tools** per spec 039 (`deploymentMode: "tools"`, `tool_declare` at startup, worker-IPC `tool_call` dispatch), and workflows persist to the **real VFS `/Workflows/`** via the loopback `/api/fs` bridge — fixing the recurring invisible-content bug. The workflow execution engine stays in BOS source (`src/lib/workflows/runner.ts`); the service orchestrates/gates execution and owns the tool surface.

**Primary deliverables**:
1. **Marketplace-item rewrite** (App Target `marketplace-item`) — service facet (tool-declaring worker) + app facet (list/detail/run UI, mockup-driven) + config.
2. **Scoped `bos-core` delegation** (ADR-7, user-approved Option A) — retire the built-in `workflowTools()` server tools that would otherwise shadow 7 of the 10 service tools.

---

## Technical Context

**Language/Version**: Node.js worker-thread service (plain JS, unbundled — outside the `@/` graph); app facet TypeScript/TSX (React).

**Primary Dependencies**: BOS 039 service-tool contract (`serviceToolBridge`, `workerIpc`), loopback `/api/fs` + `/api/workflows/*` HTTP routes, `window.__bos` broker (app facet). No BOS-source imports from the worker.

**Storage**: Real VFS `/Workflows/` (JSON files `<id>-workflow.json`), reached exclusively via loopback `/api/fs` — never a host path under `dataDir()/system/` (NFR-004).

**Testing**: Unit tests for the service modules (tool declarations, handlers, vfs bridge, migration, runner gating) + Playwright e2e for the app (list/detail/run views, empty state, service-stopped banner). Per user instruction, tests (unit + e2e) are in scope.

**Target Platform**: BrowserOS marketplace item (app + background-service facets).

**Project Type**: marketplace-item (app facet + service facet together) + one scoped bos-core change.

**Performance Goals**: Every `tool_call` bounded well under the 30s kernel timeout (`TOOL_CALL_TIMEOUT_MS = 30_000`); `workflow_run` is **fire-and-poll** (returns `runId` in <1s; caller polls `workflow_status`) — ADR-8.

**Constraints**: Service worker cannot import BOS source; all storage via loopback `/api/fs`; all execution/validation/status via loopback `/api/workflows/*`; cancellation only via `workflow_cancel` → `POST /api/workflows/cancel` (never fetch-abort) — ADR-2/ADR-4/ADR-8.

**Scale/Scope**: One marketplace item (10 tools, 3 app views) + retirement of one BOS-source tool module.

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after design (done — design.md §2).*

| Principle | Compliance |
|---|---|
| **II. Server Authority & SSR Boundary** | Compiles. Service talks to BOS only over loopback HTTP (`/api/fs`, `/api/workflows/*`) + worker IPC; no direct Node/filesystem access to the VFS from the worker. |
| **IV. Minimize Blast Radius** | Compiles with a **scoped exception** (ADR-7): retiring `workflowTools()` is a deliberate bos-core change on a feature branch, isolated to `src/lib/assistant/tools/server/workflows.ts` (delete) + `src/lib/assistant/registry.ts` (remove registration). Execution engine untouched. |
| **V. The VFS Is Not the Source** | Compiles — the core correctness fix. Workflows persist to real VFS `/Workflows/` via loopback `/api/fs`; never a host path under `dataDir()/system/`. |
| **VI. Specs & Docs Stay in Sync** | Compiles. Spec + design + plan written together; docs `services.md` §15 already documents the `deploymentMode: "tools"` contract. |
| **VII. Respect Boundaries** | Compiles with a scoped exception (ADR-7). Item files stay under `data/user-apps/items/workflows/`, never touching `package.json`/lockfiles. Single exception: the bos-core retirement. |

**Tension acknowledged (user-approved)**: spec Assumption "this spec only changes the marketplace item, not BOS source" is **relaxed** — the built-in `workflowTools()` retirement is a required bos-core change (user selected **Option A**). Execution engine stays untouched. Recorded in spec.md Assumptions (last bullet) and design.md §2/§6.

No gate violations require Complexity Tracking — the single bos-core exception is minimal, fully scoped, and user-approved.

---

## Project Structure

### Documentation (this feature)

```text
specs/user-specs/workflow-manager/001-workflow-manager-service-tools/
├── spec.md               # feature spec (Assumptions updated: bos-core exception)
├── design.md             # structural design (architect, verified by architect-reviewer)
├── mockup.html           # UI mockup — binding UI contract (list/detail/run + stopped banner)
└── plan.md               # this file
```

### Source Code — marketplace item (item root `data/user-apps/items/workflows/`)

```text
workflows/                                  # item root
├── services/
│   ├── service.json                        # REWRITE — deploymentMode:"tools", entry, configSchema
│   ├── index.js                            # REWRITE — worker entry: lifecycle + tool_declare + IPC
│   ├── tools.js                            # NEW — 10 workflow tool declarations + JSON schemas
│   ├── handlers.js                         # NEW — tool handlers (list/create/read/modify/run/status/cancel/delete/export/validate)
│   ├── vfs.js                              # NEW — loopback /api/fs bridge helpers
│   ├── migration.js                        # NEW — additive legacy-workflow migration
│   └── runner.js                           # NEW — run orchestration: state gate + fire-and-poll runId + cancel
├── app/
│   └── src/main.tsx                        # REWRITE — list/detail/run views (mockup-driven)
│       └── (components)                    # list cards, detail editor, run view + event stream
├── config/
│   └── workflows.json                      # REWRITE — { port: 0, host: "127.0.0.1" }
└── spec/                                   # optional — item's own spec
```

### Source Code — bos-core delegation (feature branch, ADR-7)

```text
src/
└── lib/assistant/
    ├── tools/server/workflows.ts           # RETIRE (delete) — workflowTools() set
    └── registry.ts                         # MODIFY — remove workflowTools import + spread
```

### Test structure (per user instruction: unit + e2e in scope)

```text
# unit — service modules
<item>/services/__tests__/tools.test.js
<item>/services/__tests__/handlers.test.js
<item>/services/__tests__/vfs.test.js
<item>/services/__tests__/migration.test.js
<item>/services/__tests__/runner.test.js

# e2e — app views (Build Studio e2e/<feature-id>.spec.ts)
specs/user-specs/workflow-manager/001-workflow-manager-service-tools/e2e/001-workflow-manager-service-tools.spec.ts
```

**Structure Decision**: Item layout mirrors the marketplace-item target shape (app/ + services/ + config/) per `target-marketplace-item.md`; the single bos-core change is scoped to the two files above. This mirrors existing item conventions rather than inventing new structure.

---

## Design Notes (folded from design.md)

### Classification
`marketplace-item` — an item with **both** an app facet (Workflow Manager UI, id `workflows`) and a service facet (tool-declaring worker), **plus** one scoped `bos-core` delegation to retire `workflowTools()` (ADR-7). Two target shapes in one feature — the item rewrite is the primary deliverable; the bos-core change exists solely to un-shadow the service tools.

### Key ADRs (authoritative detail in design.md §6)
- **ADR-1**: Execution engine stays in BOS source; service orchestrates via loopback `/api/workflows/*`, does not reimplement.
- **ADR-2**: Persist workflows to real VFS via loopback `/api/fs` — never a host path (fixes invisible-content bug).
- **ADR-3**: Service binds a real port defaulting to `0` (OS-assigned) even though tool dispatch is worker-IPC.
- **ADR-4**: Execution/validation/status/generate via loopback `/api/workflows/*`; pure CRUD via `/api/fs`.
- **ADR-5**: Additive, idempotent migration — copy + archive legacy, never delete.
- **ADR-6**: Resolve BOS's HTTP origin from `NEXT_PUBLIC_APP_ORIGIN`/`APP_ORIGIN` with `http://localhost:3000` fallback — **no `BOS_PORT` exists** (MF-1 fix).
- **ADR-7**: **Retire built-in `workflowTools()`** (delete module + registry registration) — the required bos-core change (MF-3 fix, user-approved Option A).
- **ADR-8**: `workflow_run` is **fire-and-poll** (return `runId`, caller polls `workflow_status`) — not synchronous (MF-2 fix; sync would blow the 30s kernel timeout).

### Open items carried into tasks
- **workflow_list/read/delete** don't exist in the current 7-tool server set — FR-002 requires 10; the three are new declarations sourced from the app's existing capabilities. Confirm exact names/schemas (design §7 #2).
- NDJSON streaming contract for the app path confirmed (design §7 #3).
- App capability grants (`services:read`, `fs:read`) are user-granted post-install (design §7 #5).
- Migration mutex is effectively moot (serial worker) — keep as cheap defensive documentation only (design §5 SI-3).

---

## Optional companions

`research.md` / `data-model.md` / `contracts/` are **not** separately warranted here — `design.md` already resolves the technical unknowns (origin resolution, NDJSON contract, cancellation semantics, tool surface), and the data model is small (Workflow + WorkflowTool + ExecutionEvent, defined in spec Key Entities). The tool declarations in `tools.js` (names/schemas) serve as the interface contract and live in design.md §3.3/§4.

---

## Completion Summary (plan step)

- Branch: `bos/001-workflow-manager-service-tools` (marketplace-item — no dev_branch_request for the item; the bos-core retirement will use the active feature branch `bos/042-workflow-manager-service` at implement).
- plan.md written at `specs/user-specs/workflow-manager/001-workflow-manager-service-tools/plan.md`.
- No extension hooks present.
- Next step: `tasks` (break plan into tasks, with unit + e2e tests in scope).
