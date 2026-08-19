# Workflow Manager — Service-Tool Exposure (Spec 039-Compliant) Design

**Spec**: `user-specs/workflow-manager/001-workflow-manager-service-tools/`
**App Target (spec.md)**: `marketplace-item` — **agreed**
**Date**: 2026-08-18 (revised — architect-reviewer MF-1..MF-3, SI-1..SI-3)

---

## 1. Classification

**App Target: `marketplace-item`** — an item with **both** an app facet (the
Workflow Manager UI, id `workflows`) and a service facet (a background
worker-thread daemon), rewritten so the workflow tools are exposed as
native service-declared tools via the 039 `tool_declare`/worker-IPC contract.

This **agrees** with spec.md's `App Target` field (`marketplace-item`) for the
feature as a whole — the migration + tool surface + service facet live entirely
inside the item. However, the review resolved the previously-deferred name-
collision (MF-3): retiring the built-in `workflowTools()` server tools is
**required** for US1, and that is a `bos-core` change (removing
`src/lib/assistant/tools/server/workflows.ts` and its registration in
`src/lib/assistant/registry.ts`). This design therefore spans **two** target
shapes: the marketplace-item rewrite (this spec's primary deliverable) **plus**
a small, explicitly-scoped `bos-core` delegation to retire `workflowTools()`
(§4 file plan, ADR-7). This **tensions the spec Assumption** "this spec only
changes the marketplace item, not BOS source" — acknowledged explicitly in
§2/§6; the execution engine itself still stays in BOS source untouched.

### Facets

| Facet | Path (item root `data/user-apps/items/workflows/`) | Role |
|---|---|---|
| **app** | `app/` | The Workflow Manager window UI — lists/creates/edits/runs workflows, streams step events (mockup-driven). |
| **service** | `services/service.json` + `services/` entry & modules | `deploymentMode: "tools"` worker — declares the workflow tools, owns the tool surface, gates execution, persists workflows to real VFS. |
| **config** | `config/workflows.json` | Service config (port defaults `0`, host). |

The service does **not** reimplement the workflow execution engine. It
orchestrates/gates execution and owns the tool surface; workflow *runs*
remain on the main thread in BOS source (`runner.ts`), reached through the
existing `/api/workflows/*` routes (see §6).

---

## 2. Constitution Check

| Principle | Compliance |
|---|---|
| **II. Server Authority & SSR Boundary** | Complies. The service talks to BOS only over loopback HTTP (`/api/fs`, `/api/workflows/*`) and worker IPC — no direct filesystem or Node API access to the VFS. |
| **IV. Minimize Blast Radius** | Complies with a scoped exception. The item rewrite itself is BOS-source-free; but retiring `workflowTools()` (§4/ADR-7) is a **deliberate `bos-core` change** on a feature branch, isolated to removing one server-tool module + its registry registration. The execution engine is untouched. |
| **V. The VFS Is Not the Source** | Complies, and is the core correctness fix. Workflows are persisted to the *real* VFS `/Workflows/` via the loopback `/api/fs` bridge — never a host path under `dataDir()/system/` derived from the service's own config dir. This closes the recurring invisible-content bug. |
| **VI. Specs & Docs Stay in Sync** | Complies. This `design.md` is written alongside `spec.md`; the rewrite is tracked by the feature spec. (BOS docs `services.md` §15 already documents the `deploymentMode: "tools"` contract this item opts into.) |
| **VII. Respect Boundaries** | Complies with a scoped exception. The design's *item* files stay under `data/user-apps/items/workflows/` and never touch `package.json`/lockfiles. The single exception is the `bos-core` retirement of `workflowTools()`, scoped to `src/lib/assistant/tools/server/workflows.ts` + `src/lib/assistant/registry.ts` (ADR-7). |

No conflicts flagged, but one **tension acknowledged** (see §6/ADR-7): the
spec Assumption "this spec only changes the marketplace item, not BOS source"
is **relaxed** — retiring the shadowing built-in `workflowTools()` is a
required `bos-core` change. This does not contradict the constitution; it is a
deliberate, minimal, feature-branch-scoped exception flagged for Build Studio
to reconcile before `plan`.

---

## 3. Architecture

### 3.1 Context (C1)

The Workflow Manager is a user-facing tool for authoring and running multi-step
workflows (DAGs of delegate/tool/ag-ui steps). The assistant can drive it
directly through native tools; the user can drive it through the app UI. Both
read and write the same workflows, persisted in the user's real VFS
`/Workflows/` — visible in Files and to the workflow engine.

```
┌────────────┐   calls service-declared tools   ┌───────────────────────────┐
│  Assistant  │ ───────────────────────────────▶ │  Workflow Manager service │
│  (BOS)      │                                  │  deploymentMode:"tools"   │
└────────────┘   tool_call / tool_result         └─────────────┬─────────────┘
                                                               │
┌────────────┐                                    ┌────────────┴────────────┐
│  User / app │  lists/runs/edits                 │  BOS workflow engine     │
│  (iframe)   │ ───────────────────────────────▶  │  src/lib/workflows/runner │
└────────────┘                                    └──────────────────────────┘
                                                               │
                                                               ▼
                                                    real VFS /Workflows/
                                                    (via loopback /api/fs)
```

### 3.2 Container (C2)

BOS's real containers involved:

1. **The Next.js app process** — hosts the workflow execution engine
   (`src/lib/workflows/runner.ts`), the `/api/workflows/*` routes that drive
   it, the `/api/fs` VFS bridge, and the service-tool bridge
   (`serviceToolBridge`) that registers/dispatches service-declared tools.
2. **The Workflow Manager worker-thread service** — a plain Node worker
   (unbundled, outside the `@/` graph), bound to its own configurable port
   (default `0`). It declares the workflow tools, handles `tool_call`
   dispatches, persists workflows to the real VFS via loopback `/api/fs`, and
   gates execution by checking service state.
3. **The VFS store** (`src/os/vfs.ts`, backed by `data/vfs/`) — the real
   user-sandbox filesystem where workflows live at `/Workflows/`.
4. **The marketplace item's own git repo** (`data/user-apps/items/workflows/`)
   — the item source (app + service + config), installed behind one `system/workflows`
   symlink.
5. **The Supervisor** (deployment-fronted scenarios) — proxies to the
   service's bound port via `httpPath`/`wsPath` when BOS is behind a reverse
   proxy. Not needed for worker-IPC tool dispatch (which is in-process), but
   relevant if the app or any client ever reaches the service over HTTP.

### 3.3 Component (C3) — item-level modules

The item is a self-contained marketplace item; all files below are **new or
rewritten** under `data/user-apps/items/workflows/`.

```
workflows/                                  (item root)
├── services/
│   ├── service.json                        # REWRITE — deploymentMode:"tools", entry, configSchema
│   ├── index.js                            # REWRITE — worker entry: lifecycle + tool_declare + IPC
│   ├── tools.js                            # NEW — workflow tool declarations + schemas (FR-002)
│   ├── handlers.js                         # NEW — tool handlers: list/create/read/modify/run/status/
│   │                                       #         cancel/delete/export/validate
│   ├── vfs.js                              # NEW — loopback /api/fs bridge helpers (FR-004)
│   ├── migration.js                        # NEW — additive legacy-workflow migration (FR-005)
│   └── runner.js                           # NEW — run orchestration: state gate + step-event streaming (FR-007/008)
├── app/
│   └── src/main.tsx                        # REWRITE — app UI (mockup-driven; list/detail/run views)
│       └── (components)                    # list cards, detail editor, run view + event stream
├── config/
│   └── workflows.json                      # REWRITE — { port: 0, host: "127.0.0.1" }
└── spec/                                   # optional — item's own spec (existing)
```

#### `services/service.json`

```json
{
  "id": "workflows",
  "name": "Workflow Manager Service",
  "version": "2.0.0",
  "description": "Exposes workflow tools as native service-declared tools; persists workflows to the real VFS /Workflows/.",
  "entry": "index.js",
  "deploymentMode": "tools",
  "configSchema": {
    "type": "object",
    "properties": {
      "port": { "type": "number", "default": 0, "description": "Service port — 0 lets the OS assign a free port" },
      "host": { "type": "string", "default": "127.0.0.1", "description": "Bind host" }
    }
  },
  "dependencies": [],
  "settingsRegistration": {
    "label": "Workflow Manager",
    "icon": "workflow",
    "order": 100
  }
}
```

- `deploymentMode: "tools"` opts the service into native tool exposure
  (validated by `manifestValidator.ts`; FR-001, FR-010).
- `port` defaults to `0` (OS-assigned, non-colliding) per §3/NFR-003 — never a
  fixed number; `ServiceManager` skips its port check for `0` and records the
  actually-bound port in `runtime.json`.

#### `services/index.js` — worker entry

Guarded `parentPort` (CH-011 — the entry is also `import()`ed from the main
thread for `validateManifestAtStart`):

- On `initialize` (`msg.configDirPath`), read config, bind the HTTP/health
  server (port from config or OS-assigned), post `{ type: "initialized" }`
  and `{ type: "bound", port, host }`.
- After `initialized`, post one `tool_declare` per workflow tool (FR-001):
  `workflow_list`, `workflow_create`, `workflow_read`, `workflow_modify`,
  `workflow_run`, `workflow_status`, `workflow_cancel`, `workflow_delete`,
  `workflow_export`, `workflow_validate` — each with `name`, `description`,
  `inputSchema`. Declarations are static per process (039 v1 assumption).
- Run the one-time additive migration (`migration.js`) after bind (FR-005),
  logging progress.
- On `tool_call` (`{ callId, name, args }`), dispatch to `handlers.js`,
  post `{ type: "tool_result", payload: { callId, result } }` or
  `{ type: "tool_error", payload: { callId, error } }` (FR-003).
- On `dispose`, close the server and post `{ type: "disposed" }`.

#### `services/tools.js` — tool declarations & schemas

The ten tool declarations, sourced from the existing `workflowTools()`
definitions in `src/lib/assistant/tools/server/workflows.ts` (same names,
descriptions, schemas — behaviorally identical surface; spec Assumption). Each
declaration's `inputSchema` is the JSON Schema form of the current `schema()`
definitions. Example:

```js
{
  name: "workflow_list",
  description: "List all workflows stored in the real VFS /Workflows/, with id, name, version, step/agent counts.",
  inputSchema: { type: "object", properties: {}, required: [] }
},
{
  name: "workflow_create",
  description: "Generate and persist a new workflow from a natural-language task description.",
  inputSchema: { type: "object", properties: { taskDescription: { type: "string" } }, required: ["taskDescription"] }
}
```

#### `services/handlers.js` — tool handlers

Each handler implements the workflow operation against the real VFS through
`vfs.js`, or through the existing BOS workflow routes for execution:

| Tool | Implementation |
|---|---|
| `workflow_list` | `vfs.list('/Workflows')` → filter `*-workflow.json` → parse each → return summaries. |
| `workflow_create` | Call `POST /api/workflows/generate` (BOS generate route) then `POST /api/workflows` to persist; or call generate via loopback and save via `/api/fs`. |
| `workflow_read` | `vfs.read('/Workflows/<id>-workflow.json')`. |
| `workflow_modify` | Read workflow, deep-merge patch, write back via `/api/fs`, then validate. |
| `workflow_run` | Gate on service running state (`runner.js`), then `POST /api/workflows/run` to start the run and **return a `runId` immediately** (fire-and-poll — ADR-8); caller polls `workflow_status` for progress/final state. (FR-007/008). |
| `workflow_status` | `GET /api/workflows/status?id=<id>` (engine's runtime status). |
| `workflow_cancel` | `POST /api/workflows/cancel?id=<id>` (engine's `cancelWorkflow`). |
| `workflow_delete` | `DELETE` the workflow + execution-log JSON via `/api/fs`. |
| `workflow_export` | `vfs.read` the workflow JSON, return full JSON string. |
| `workflow_validate` | Read workflow, `POST /api/workflows/validate` (engine's `validateWorkflow`). |

Handlers never import BOS source. All storage via loopback `/api/fs`
(NFR-004); execution/validation via loopback `/api/workflows/*`.

#### `services/vfs.js` — loopback VFS bridge

Helpers wrapping the real VFS HTTP API over loopback
(`fetch('http://127.0.0.1:<port>/api/fs...')`), the sanctioned unbundled-
worker-thread path (`target-marketplace-item.md`, `services.md` §6):

```js
// BOS's own HTTP origin. Resolved from the same env var the rest of BOS uses
// for its public origin (see src/lib/integrations/oauth/origin.ts,
// src/lib/integrations/webhooks/manager.ts). No `BOS_PORT` exists anywhere in
// src/ — only BOS_PORT_BASE/BOS_PORT_POOL_SIZE in PortChecker.ts (the service
// -port pool), so it is never injected by the worker manager. The loopback
// host is always 127.0.0.1; only the port varies.
const BOS_ORIGIN = () => {
  const origin = process.env.NEXT_PUBLIC_APP_ORIGIN ?? process.env.APP_ORIGIN;
  if (origin) return origin.replace(/\/$/, "");
  return "http://localhost:3000"; // fallback for plain `npm run dev` (Next.js default port)
};

async function vfsList(p)   { /* GET /api/fs?op=list&path= */ }
async function vfsRead(p)   { /* GET /api/fs?op=read&path= */ }
async function vfsWrite(p, c){ /* POST /api/fs { op:'write', path, content } */ }
async function vfsMkdir(p)  { /* POST /api/fs { op:'mkdir', path } */ }
async function vfsDelete(p) { /* POST /api/fs { op:'delete', path } */ }
async function vfsRename(p,to){ /* POST /api/fs { op:'rename', path, to } */ }
```

The service resolves BOS's HTTP origin from `NEXT_PUBLIC_APP_ORIGIN`/`APP_ORIGIN`
with a `http://localhost:3000` fallback (see §11 ADR-6) — **not** from any
injected `BOS_PORT`, which does not exist in src/ (verified: `ServiceManager`
passes only `{ configDirPath, logsPath, serviceId }` to the worker, and
`runtime.json` holds the service's *own* bound port, not BOS's). The loopback
host is always `127.0.0.1`. Small JSON files (workflows, execution logs) use
the JSON `/api/fs` route — no `/api/fs/raw` needed (these are trivially small).
No special auth headers for plain VFS content under `/Workflows` (unmounted
path → no feature scope; per-user container isolation applies in multi-user
deployments).

#### `services/migration.js` — additive legacy migration

Runs once after bind (idempotent, FR-005):

1. For each workflow JSON found in a **legacy non-VFS location** (e.g.
   `dataDir()/system/...` — the buggy host path the old item wrote to), read
   it.
2. If the workflow id does not already exist under real VFS `/Workflows/`, copy
   it there via `vfsWrite` (additive-only).
3. Archive the legacy location (rename to a `.legacy-archived` suffix) — never
   delete.
4. Log each migrated/archived/skipped workflow via the log channel (NFR-005).

Migration is guarded so it cannot run concurrently with a tool call that
mutates `/Workflows` (a simple in-worker mutex around the write path).

#### `services/runner.js` — run orchestration & gating

- **State gate (FR-007)**: `workflow_run` first checks that the service is in
  a healthy running state. Because the worker *is* the service, the natural
  "running" signal is "the worker is alive and handling `tool_call`" — a call
  can only reach the worker if the service is running. An explicit
  `isWorkflowsServiceRunning()`-equivalent is enforced kernel-side by 039:
  `ServiceToolBridge.unregisterServiceTools(serviceId)` removes all tools on
  `stop`/crash/uninstall (FR-006/011), so a run against a stopped service is
  refused before dispatch (schema/registry-level). The worker additionally
  re-checks a `healthy` flag it sets after successful bind + migration, and
  returns `tool_error` "Workflows service is not ready" if unset.
- **Streaming (FR-008) + timeout (ADR-8)**: `workflow_run` is **fire-and-poll**,
  not synchronous — a multi-step run exceeds the 30s kernel tool-call timeout
  (`TOOL_CALL_TIMEOUT_MS = 30_000` in `ServiceManager.ts`; the run route's
  `maxDuration = 600` exists precisely because runs are long). It therefore
  (1) `POST /api/workflows/run` once to *start* the engine's NDJSON stream,
  (2) immediately return a `runId` + `workflowId` `tool_result` (bounded, <1s,
  well within 30s), and (3) let the caller poll `workflow_status` for
  progress/final state. The engine's step events are still streamed for the
  **app** (which consumes the NDJSON stream directly, no IPC timeout
  involved) and are also persisted to the execution log, so `workflow_status`
  reflects live step statuses (`src/lib/workflows/runner.ts` mirrors every
  event into the runtime status map). See ADR-8 for why this is chosen over
  raising the per-service timeout.
- **Cancellation (FR-009, SI-1)**: cancellation is **never** via aborting the
  loopback fetch — `runWorkflowStream`'s `finally { await driver }` keeps the
  run alive even if the consumer disconnects. The only way to halt a run is
  `cancelWorkflow(id)`, reached exclusively through the `workflow_cancel` tool
  → `POST /api/workflows/cancel` (`src/app/api/workflows/cancel/route.ts`,
  which aborts the engine's per-run `AbortController`). Both the app's Cancel
  button and the `workflow_cancel` tool therefore route through this single
  handler.

#### `app/src/main.tsx` — app UI

The Workflow Manager app (mockup-driven). Views:

- **List view** — card grid of workflows read from the service's real VFS
  (`/Workflows/`), each card with name, id·version, step/agent counts, and a
  status pill (Ready/Running/Failed) derived from `workflow_status`. Empty
  state when `/Workflows/` has no workflows. A service-stopped banner when the
  service is not running.
- **Detail view** — steps list, dependencies, config tabs; Run / Edit /
  Export / Delete actions.
- **Run view** — progress bar, per-step statuses (completed/running/retry/
  pending/cancelled), and a live event stream; Cancel run button.

The app talks to the service/BOS through `window.__bos` broker with granted
capabilities (`services:read`, `workflows:...`, `fs:read`) once installed as a
real app — not a direct `fetch()` under the opaque `marketplace`-origin
sandbox. It lists workflows through the service (per FR-006) so app, engine,
Files, and assistant all read the same real-VFS location.

---

## 4. Concrete File/Module Plan

Only files this feature **creates or modifies** — the item files under the item
root `data/user-apps/items/workflows/`, plus the single scoped `bos-core`
delegation below (ADR-7). No other BOS-source files are modified.

| Path (item root) | Action | Purpose |
|---|---|---|
| `services/service.json` | **Modify** | `deploymentMode: "tools"`, configSchema (port `0`), settingsRegistration. |
| `services/index.js` | **Rewrite** | Worker entry — lifecycle IPC, `tool_declare` at startup, `tool_call` dispatch, migration trigger. |
| `services/tools.js` | **Create** | The 10 workflow tool declarations + JSON schemas (FR-002). |
| `services/handlers.js` | **Create** | Tool handlers for list/create/read/modify/run/status/cancel/delete/export/validate. |
| `services/vfs.js` | **Create** | Loopback `/api/fs` bridge helpers (FR-004, NFR-004). |
| `services/migration.js` | **Create** | One-time additive legacy-workflow migration (FR-005). |
| `services/runner.js` | **Create** | Run orchestration — state gate, NDJSON step-event streaming, cancellation (FR-007/008/009). |
| `app/src/main.tsx` | **Rewrite** | App UI — list/detail/run views (mockup-driven), reads `/Workflows` through the service. |
| `app/` (supporting components/CSS) | **Modify** | Components for list cards, detail editor, run view + event stream. |
| `config/workflows.json` | **Modify** | `{ "port": 0, "host": "127.0.0.1" }`. |

### `bos-core` delegation (ADR-7 — required for US1, scoped exception)

| Path (BOS source, feature branch) | Action | Purpose |
|---|---|---|
| `src/lib/assistant/tools/server/workflows.ts` | **Retire (delete)** | Remove the `workflowTools()` set that shadows the service's 7 overlapping tools (registry.ts spreads service tools first, built-ins win every collision). |
| `src/lib/assistant/registry.ts` | **Modify** | Remove the `workflowTools` import + `...workflowTools()` spread. |

This is the **only** BOS-source change in the design, and it exists *solely* to
un-shadow the service-declared tools — the execution engine
(`src/lib/workflows/*`) stays untouched. It tensions the spec Assumption (see
§6/ADR-7).

No entries for the `builtin-app` target shape (does not apply — omitted).

---

## 5. Integration Points

Existing BOS mechanisms this design **calls into** but does **not** create or
modify — dependencies, not deliverables:

| Mechanism | Real route/file | Used for |
|---|---|---|
| **Service tool bridge** (`deploymentMode:"tools"`, `tool_declare`, gating, lifecycle cleanup) | `src/lib/agent/service-tool-bridge.ts`, `src/core/service/ServiceManager.ts`, `src/core/service/workerIpc.ts` | Declaring the 10 tools, schema-validation before dispatch, allowlist/deferred gating, removal on stop/crash/uninstall (039 FR-004/005/006). |
| **Worker IPC protocol** (`tool_call`/`tool_result`/`tool_error`, `bound`) | `src/core/service/workerIpc.ts`, `serviceToolTypes.ts` | Dispatch and result/error return for each tool call. |
| **Workflow execution engine** | `src/lib/workflows/runner.ts` (`runWorkflowStream`, `cancelWorkflow`, `isWorkflowsServiceRunning`), `src/lib/workflows/store.ts`, `validate.ts`, `generate.ts` | **Stays in BOS source** (spec Assumption). The service orchestrates/gates; the engine runs on the main thread. Reached via existing routes. |
| **Workflow API routes** | `src/app/api/workflows/route.ts` (CRUD), `.../run/route.ts`, `.../status`, `.../cancel`, `.../validate`, `.../generate` | The service's execution/validation/status/generate calls (loopback). |
| **Real VFS bridge** | `src/app/api/fs/route.ts` (`op=list|read|write|mkdir|delete|rename`), `src/os/vfs.ts` | All workflow persistence under `/Workflows/` — the sanctioned loopback path, never a host path under `dataDir()/system/`. |
| **Assistant tool registry / gating** | `src/lib/assistant/registry.ts` (`assistantTools()`), `src/lib/assistant/gate.ts`, `src/lib/agent/tool-gate.ts`, `capabilities-registry.ts` | Surfacing service tools and applying the same allowlist/deferred gates as built-ins. |
| **`workflowTools()` server tool set (superseded → retired)** | `src/lib/assistant/tools/server/workflows.ts` | **Retired** (ADR-7). Its definitions remain the source of truth for the service's declarations (spec Assumption), but the module + registration are removed via the `bos-core` delegation — otherwise it shadows all 7 overlapping service tools (registry.ts spreads service tools first, built-ins win). |
| **`window.__bos` broker + app capabilities** | `src/os/...`, `src/lib/apps/store.ts` (`setAppCapabilities`) | The app facet reaches BOS APIs (services, fs) through the SDK broker with granted capabilities, not direct fetch (opaque-origin sandbox). |
| **Settings → Plugins → Services** | `src/components/apps/settings/ServicesTab.tsx`, `ServiceCard.tsx`, `ServiceConfigPanel.tsx` | User manages start/stop/restart/logs/config of the service; `runtime.json` holds the actual bound port. |

---

## 6. ADRs

### ADR-1: Keep the execution engine in BOS source (no reimplementation in the service)

- **Context**: Workflow *runs* require the full assistant stack on the main
  thread (`runSubAgent` needs the agent runtime, sub-agent store, feature-branch
  resolution). A worker-thread service cannot host that.
- **Options**: (a) Service reimplements execution; (b) service orchestrates via
  loopback `/api/workflows/*`, engine stays in `src/lib/workflows/runner.ts`.
- **Decision**: (b). The service owns the tool surface + gating; the engine
  stays where it is (spec Assumption, `architecture-overview.md` §14).
- **Consequences**: The service is a thin orchestrator over existing routes —
  less code, no duplicated engine logic, and the engine's cancellation/status/
  streaming semantics are reused verbatim. The service's "running" gate remains
  meaningful because 039 removes the tools on stop, so a run against a stopped
  service is refused before dispatch.

### ADR-2: Persist workflows to the real VFS via loopback `/api/fs`, never a host path

- **Context**: The recurring invisible-content bug — workflows landing under
  `dataDir()/system/...` (a host path derived from the service's config dir)
  are invisible to Files/engine/app. A worker thread can't import `@/os/vfs`.
- **Options**: (a) Loopback `/api/fs` bridge (sanctioned worker-thread path);
  (b) reimplement VFS logic in the worker; (c) write to a host path.
- **Decision**: (a). The worker calls BOS's own `/api/fs` over loopback — path
  traversal protection, mount-table routing, atomic writes all stay in BOS
  (`target-marketplace-item.md` "Reaching the VFS from a service").
- **Consequences**: Workflows at real VFS `/Workflows/` are visible to Files,
  the engine's `listWorkflows()`, the app, and the assistant — no invisible
  content (SC-003). Small JSON files → JSON `/api/fs` route; no `/api/fs/raw`
  needed.

### ADR-3: The service binds a real port defaulting to `0` (even though tools are worker-IPC)

- **Context**: 039 tool dispatch is worker IPC (in-process), so the workflow
  tools technically don't need a network port. But the service still binds one:
  (a) every worker-thread service binds a real port — that's what the
  Supervisor's proxy forwards to (`services.md` §6/§11); (b) the reserved
  `loopback-http` tool transport value — a type-level reservation only, with
  **no route wired in v1** (services.md §15; FR-008 is the `tool_call`/
  `tool_result`/`tool_error` IPC contract, not a shipped HTTP path) — and the
  app's config reads (`GET /api/services/<id>/config` → `runtime.port`) rely
  on it.
- **Options**: (a) No port (incorrect — no such mechanism); (b) fixed port
  (collision risk); (c) port defaulting to `0`.
- **Decision**: (c). `configSchema.port` defaults `0` (OS-assigned, never
  collides with BOS reserved ports); `ServiceManager` records the bound port in
  `runtime.json`.
- **Consequences**: Satisfies NFR-003; no EADDRINUSE risk; the app reads the
  real bound port/`httpPath` from config when it needs to reach the service
  over HTTP (deployment scenarios in §11 of `services.md` — Bastion /
  Supervisor-only / standalone).

### ADR-4: Execution/validation/status/generate via loopback `/api/workflows/*`, not direct worker imports

- **Context**: The worker cannot import `src/lib/workflows/*` (unbundled, no
  `@/` graph). Reimplementing `validateWorkflow`/`generateWorkflowFromTask` in
  the worker would duplicate BOS logic and drift.
- **Decision**: The service calls the existing BOS workflow routes over
  loopback (`/api/workflows/*`) for anything that touches the engine —
  execution, validation, status, cancel, generate. Pure CRUD (list/read/delete/
  modify/export) reads/writes JSON directly via `/api/fs`.
- **Consequences**: No duplicated logic; the worker stays thin; the engine
  remains the single authority for validation and execution.

### ADR-5: Additive, idempotent migration — copy + archive, never delete

- **Context**: Stranded legacy workflows live in a non-VFS host path. Deleting
  the legacy copy risks data loss; copying blindly risks overwriting newer
  real-VFS content.
- **Decision**: Additive-only, idempotent: skip workflows whose id already
  exists under real VFS; archive (rename) the legacy location, never delete
  (FR-005, spec Assumption).
- **Consequences**: No data loss, no overwrite; migration can be re-run safely;
  the legacy archive remains recoverable.

> **SI-3 note for `implement`**: the migration mutex is effectively moot — a
> worker thread's `parentPort` message handler is serial, and migration runs to
> completion before any `tool_call` is dispatched (declarations are posted after
> `initialized`, migration before the tool surface is live). It is harmless but
> adds no real concurrency protection; keep it only as cheap defensive
> documentation, not as a synchronization mechanism.

### ADR-6: Resolve BOS's HTTP origin from `NEXT_PUBLIC_APP_ORIGIN` (MF-1)

- **Context**: `services/vfs.js` needs BOS's own HTTP origin for loopback
  `/api/fs` and `/api/workflows/*` calls. The original design claimed the worker
  manager "injects `BOS_PORT`" — **false**; no `BOS_PORT` exists anywhere in
  src/ (only `BOS_PORT_BASE`/`BOS_PORT_POOL_SIZE`, the service-port pool in
  `PortChecker.ts`). `ServiceManager` passes only `{ configDirPath, logsPath,
  serviceId }` (verified `src/core/service/ServiceManager.ts`), and
  `runtime.json` holds the service's own port, not BOS's.
- **Options**: (a) Read `process.env.NEXT_PUBLIC_APP_ORIGIN`/`APP_ORIGIN` with a
  `http://localhost:3000` fallback — the exact pattern BOS itself uses for
  public-origin resolution (`src/lib/integrations/oauth/origin.ts`,
  `src/lib/integrations/webhooks/manager.ts`); (b) a BOS-source change to pass
  BOS's origin into worker `initialize`/`workerData`.
- **Decision**: (a). It uses a mechanism that demonstrably exists, requires **no
  BOS-source change**, and matches how the rest of BOS resolves its own origin.
- **Consequences**: No extra worker-manager plumbing; in a reverse-proxied
  deployment the loopback call may traverse the public origin rather than
  staying on 127.0.0.1 — acceptable for auth-gated routes, flagged in §7. If a
  future design needs a guaranteed in-container loopback origin, that would be a
  `bos-core` change to pass BOS's origin explicitly.

### ADR-7: Retire the built-in `workflowTools()` server tools (MF-3)

- **Context**: US1 requires the workflow tools to appear as service-declared
  native tools. But `registry.ts`'s `assistantTools()` spreads service tools
  **first** and built-ins **after**, so a built-in wins every name collision —
  the existing `workflowTools()` (7 tools) shadows 7 of the 10 service tools,
  and the rewrite would be moot. The design previously left this as an "open
  question" with a contradictory "shadowed-or-removed" position.
- **Options**: (a) Retire `workflowTools()` (delete module + registry
  registration) — a `bos-core` change; (b) gate the server set behind
  "service-not-running" (keep both, but hide built-ins while the service runs);
  (c) leave both registered (shadowed — rewrite moot).
- **Decision**: (a) **retire**. Option (b) is rejected: it leaves dead
  duplicate code, requires runtime gating logic that 039 does not provide for
  built-in-vs-service collisions, and still leaves the shadowing ambiguity; the
  service is the single authority for the workflow tool surface. The service
  tools are behaviorally identical (spec Assumption — sourced from
  `workflowTools()`'s own definitions), so the assistant loses no capability.
- **Consequences**: This is the **one** `bos-core` change in the design. It
  **tensions the spec Assumption** "this spec only changes the marketplace item,
  not BOS source" — acknowledged explicitly (see §2, §6). It must run on a
  feature branch, isolated to the two files; the execution engine
  (`src/lib/workflows/*`) is untouched. Flagged for Build Studio to reconcile
  the assumption before `plan`; if the assumption is non-negotiable, US1 cannot
  be met and the alternative (b) would need a 039 extension to prefer
  service-declared tools over built-ins on collision — a larger `bos-core`
  change.

### ADR-8: `workflow_run` is fire-and-poll, not synchronous (MF-2)

- **Context**: A multi-step workflow run exceeds the kernel's 30s tool-call
  timeout (`TOOL_CALL_TIMEOUT_MS = 30_000` in `ServiceManager.ts`; the run
  route's `maxDuration = 600` exists precisely because runs are long). Buffering
  NDJSON events and returning one `tool_result` at run end breaks SC-002/SC-006
  for any non-trivial run.
- **Options**: (a) Return a `runId` immediately; caller polls `workflow_status`;
  (b) raise the per-service timeout via `serviceToolBridge().setToolCallTimeoutMs()`.
- **Decision**: (a). Option (b) is rejected **because it is not effective
  today**: verified in source — the bridge's `setToolCallTimeoutMs()` writes
  `this.toolCallTimeoutMs`, but `ServiceManager`'s dispatcher calls
  `waitForToolResult(worker, callId, TOOL_CALL_TIMEOUT_MS, signal)` with the
  **hardcoded module const**, ignoring the bridge field. Making the timeout
  actually configurable would itself be a `bos-core` change, and even then a
  synchronous multi-minute tool call is fragile (holds the run's tool slot,
  single failure point). Fire-and-poll keeps each `tool_call` bounded (<1s),
  streams progress via `workflow_status` (which mirrors every engine event into
  the runtime status map), and lets cancellation stay on the `workflow_cancel`
  path.
- **Consequences**: `workflow_run` returns `runId` + `workflowId` and the caller
  (assistant or app) polls `workflow_status`; step events still stream to the
  **app** via the NDJSON route directly (no IPC timeout involved). Trade-off:
  the assistant must issue follow-up `workflow_status` calls — acceptable, and
  consistent with the existing `workflow_status` tool.

---

## 7. Risks / Open Questions

1. **Server-tool retirement — RESOLVED (MF-3/ADR-7)**: The existing BOS-source
   `workflowTools()` (`workflow_create/modify/run/status/cancel/export/validate`)
   collides by name with 7 of the 10 service-declared tools. Verified in
   `registry.ts`: service tools are spread **first**, built-ins **after**, so a
   built-in wins every collision — if `workflowTools()` stayed registered, the
   service's 7 overlapping tools would be shadowed and the rewrite moot. **Decision:
   retire `workflowTools()`** — delete the module + remove its registry
   registration, as a scoped `bos-core` delegation (see §4). The gating-
   behind-service-not-running alternative was rejected (ADR-7).
2. **The `workflow_list`/`workflow_read`/`workflow_delete` tools don't exist in
   the current `workflowTools()` set** — FR-002 requires 10 tools but the
   current server set has 7. The design adds list/read/delete as new
   declarations (sourced from the app's existing capabilities). Confirm exact
   names/schemas for these three.
3. **`/api/workflows/run` NDJSON streaming contract**: The service consumes the
   run stream over loopback (for the app path). Confirmed shape: `POST
   /api/workflows/run` returns `application/x-ndjson` (buffered NDJSON via
   `ReadableStream`, `maxDuration = 600`). **Cancellation is NOT via
   fetch-abort** — `runWorkflowStream`'s `finally { await driver }` keeps the
   run going on consumer disconnect; cancellation is exclusively via
   `cancelWorkflow(id)` through `POST /api/workflows/cancel` (`src/app/api/
   workflows/cancel/route.ts`). This is why `workflow_cancel` is the only
   cancellation path (SI-1).
4. **Loopback origin resolution — RESOLVED (MF-1/ADR-6)**: There is **no**
   `BOS_PORT` in src/ and `ServiceManager` passes only
   `{ configDirPath, logsPath, serviceId }` to the worker; `runtime.json` holds
   the service's *own* port, not BOS's. The origin is resolved from
   `NEXT_PUBLIC_APP_ORIGIN`/`APP_ORIGIN` (the same env var the rest of BOS uses)
   with a `http://localhost:3000` fallback — grounded in `oauth/origin.ts`,
   `webhooks/manager.ts`. No BOS-source change needed for this. Residual caveat:
   in a reverse-proxied deployment the worker's loopback calls to
   `NEXT_PUBLIC_APP_ORIGIN` may traverse the public origin rather than staying
   on 127.0.0.1 — acceptable (the VFS/workflow routes are auth-gated), flagged
   for `implement` to confirm under Bastion.
5. **App capability grants**: The app needs `services:read` (and possibly
   `fs:read`) granted in Settings → Apps after install — never auto-granted.
   The mockup's "Start service" button must route through the granted
   capability or the generic service-management UI.

---

## 8. UI Mockup Reference

**Path**: `mockup.html` (same spec directory).

The mockup defines three views that map directly onto the Component design:

| Mockup view | Maps to |
|---|---|
| **List view** (`#view-list`) | `app/src/main.tsx` list view; cards read real VFS `/Workflows/` via the service (FR-006). Status pills (Ready/Running/Failed) from `workflow_status`. Empty state when `/Workflows/` is empty. |
| **Detail/editor view** (`#view-detail`) | Detail view; steps list, dependencies, config tabs; Run / Edit / Export / Delete actions → `workflow_run`/`workflow_modify`/`workflow_export`/`workflow_delete` (through the service or its tools). |
| **Run view** (`#view-run`) | Run view; progress bar, per-step statuses, live event stream (FR-008), Cancel run (FR-009). |

The mockup's **service-stopped banner** (`#stopped-banner`) and **service
status pill** (`#service-pill`) reflect the service state — when stopped, tools
are removed (FR-011) and the app shows the amber banner with a "Start service"
action. The **empty state** copy ("the engine, Files, and the app all read the
same location") is the UX statement of ADR-2's invisible-content fix.
