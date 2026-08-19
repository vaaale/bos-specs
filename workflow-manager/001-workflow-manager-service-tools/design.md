# Workflow Manager — Service-Tool Exposure (Spec 039-Compliant) Design

**Spec**: `user-specs/workflow-manager/001-workflow-manager-service-tools/`
**App Target (spec.md)**: `marketplace-item` — **agreed**
**Date**: 2026-08-18

---

## 1. Classification

**App Target: `marketplace-item`** — an item with **both** an app facet (the
Workflow Manager UI, id `workflows`) and a service facet (a background
worker-thread daemon), rewritten so the workflow tools are exposed as
native service-declared tools via the 039 `tool_declare`/worker-IPC contract.

This **agrees** with spec.md's `App Target` field (`marketplace-item`). There is
no `bos-core` component to this change: the workflow execution engine stays in
BOS source (`src/lib/workflows/runner.ts`), the service-tool bridge and worker
IPC are already implemented BOS mechanisms (039, bos-core), and the migration +
tool surface live entirely inside the item. The only BOS-source file whose
*role* changes is `src/lib/assistant/tools/server/workflows.ts`
(`workflowTools()`), but this design **does not modify it** — see Integration
points; its tool set is simply superseded by the service-declared equivalents.

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
| **IV. Minimize Blast Radius** | Complies. This is a marketplace-item rewrite — no BOS-source change, no feature branch required. The item installs via `app_build` behind one symlink; the execution engine is untouched. |
| **V. The VFS Is Not the Source** | Complies, and is the core correctness fix. Workflows are persisted to the *real* VFS `/Workflows/` via the loopback `/api/fs` bridge — never a host path under `dataDir()/system/` derived from the service's own config dir. This closes the recurring invisible-content bug. |
| **VI. Specs & Docs Stay in Sync** | Complies. This `design.md` is written alongside `spec.md`; the rewrite is tracked by the feature spec. (BOS docs `services.md` §15 already documents the `deploymentMode: "tools"` contract this item opts into.) |
| **VII. Respect Boundaries** | Complies. The design modifies only the item's own files under `data/user-apps/items/workflows/`. It does not touch `package.json`, lockfiles, or BOS source. |

No conflicts flagged.

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
| `workflow_run` | Gate on service running state (`runner.js`), then `POST /api/workflows/run` and stream step events back (FR-007/008). |
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
const BOS_ORIGIN = () => `http://127.0.0.1:${process.env.BOS_PORT}`; // BOS's own port, injected by the worker manager

async function vfsList(p)   { /* GET /api/fs?op=list&path= */ }
async function vfsRead(p)   { /* GET /api/fs?op=read&path= */ }
async function vfsWrite(p, c){ /* POST /api/fs { op:'write', path, content } */ }
async function vfsMkdir(p)  { /* POST /api/fs { op:'mkdir', path } */ }
async function vfsDelete(p) { /* POST /api/fs { op:'delete', path } */ }
async function vfsRename(p,to){ /* POST /api/fs { op:'rename', path, to } */ }
```

The service reads BOS's own port from its `initialize` config/`runtime.json`
context (the worker manager supplies the BOS port alongside `serviceId`); the
loopback host is always `127.0.0.1`. Small JSON files (workflows, execution
logs) use the JSON `/api/fs` route — no `/api/fs/raw` needed (these are
trivially small). No special auth headers for plain VFS content under
`/Workflows` (unmounted path → no feature scope; per-user container
isolation applies in multi-user deployments).

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
- **Streaming (FR-008)**: `workflow_run` calls `POST /api/workflows/run` with
  an NDJSON body stream, reads each NDJSON event (step.start/complete/fail/
  retry/cancel, workflow.*), and emits each as an `ExecutionEvent` in the
  `tool_result` payload (same `encodeNested`-style tree the current
  `workflow_run` produces) — so the app and assistant observe progress and
  final state. The NDJSON stream is consumed with a timeout/cancellation
  wrapper (NFR-001).

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

Only files this feature **creates or modifies** (all under the item root
`data/user-apps/items/workflows/`). No BOS-source files are modified.

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

No entries for `bos-core` or `builtin-app` target shapes (they do not apply —
omitted, not marked N/A).

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
| **`workflowTools()` server tool set (superseded)** | `src/lib/assistant/tools/server/workflows.ts` | **Not modified.** Its definitions are the source of truth for the service's declarations (spec Assumption); once the service declares the same names, the server tools become redundant/duplicate and are shadowed-or-removed per 039 collision rules. Flag for `implement` to confirm whether they should be retired. |
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
  Supervisor's proxy forwards to (`services.md` §6/§11); (b) a future
  `loopback-http` tool transport (039 FR-008) and the app's config reads
  (`GET /api/services/<id>/config` → `runtime.port`) rely on it.
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

---

## 7. Risks / Open Questions

1. **Server-tool retirement (highest)**: The existing BOS-source
   `workflowTools()` (`workflow_create/modify/run/status/cancel/export/validate`)
   will collide by name with the service-declared tools. Per 039, a built-in
   wins a name collision — so if `workflowTools()` stays registered, the
   service's tools are shadowed and the rewrite is moot. **Open question for
   `implement`**: retire/remove `workflowTools()` from
   `src/lib/assistant/tools/server/workflows.ts` (and its registration) as part
   of this change, or gate it behind service-not-running. This is a *role*
   change to a BOS-source file, not a modification this item makes — it needs
   an explicit decision before `implement`, and may need a `bos-core` follow-up
   delegation if removal is required.
2. **The `workflow_list`/`workflow_read`/`workflow_delete` tools don't exist in
   the current `workflowTools()` set** — FR-002 requires 10 tools but the
   current server set has 7. The design adds list/read/delete as new
   declarations (sourced from the app's existing capabilities). Confirm exact
   names/schemas for these three.
3. **`/api/workflows/run` NDJSON streaming contract**: The service consumes the
   run stream over loopback. Need to confirm the route's request/response shape
   (streaming NDJSON vs. buffered) and that cancellation propagates through the
   loopback fetch (AbortController on the service side → `cancelWorkflow`).
4. **`process.env.BOS_PORT` availability in the worker**: The loopback origin
   must resolve BOS's own port. Confirm how `ServiceManager` exposes BOS's port
   to worker `initialize` (or read it from `runtime.json`/config context) —
   this must be grounded in source before `implement`, not guessed.
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
