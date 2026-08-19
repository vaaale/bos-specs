# Design — Service Tool Exposure (039)

**Feature Branch**: `bos/039-service-tool-exposure`
**Designer**: architect
**Date**: 2026-08-10
**Status**: Draft

---

## 1. Classification

**App Target**: `bos-core` *(matches spec.md's App Target field)*

This is a kernel capability that lives in `src/`. It modifies how BOS's own service-runtime (`src/core/service/`), assistant tool registry (`src/lib/assistant/`), and capability registry (`src/lib/agent/`) behave. The marketplace service is only a **consumer** of the new capability; nothing ships as a marketplace item. Therefore this is NOT a marketplace item and NOT a builtin app — it is core BOS source.

---

## 2. Context

### Problem
Today, a marketplace item that ships a background service (a worker-thread daemon) can only expose agent-callable functions to the assistant by standing up an MCP server. There is no native path for a service to register and expose a tool directly. This adds that native path: a service declares tools at startup, BOS surfaces them into the existing `AssistantTool` registry, and BOS invokes them over a defined contract.

### Design goals
1. **Native, non-MCP tool exposure** for worker-thread services — tools surface as `AssistantTool` entries with `execution: "server"` semantics.
2. **Backend-agnostic contract** — the tool-invocation contract must not bake in `worker_threads`; a child-process backend could adopt it later (v1 is worker-thread only, per spec assumptions).
3. **Reuse existing mechanisms** — gating (016 allowlist + 025 deferred + description overrides), schema validation (Ajv already used by `manifestValidator`), loopback auth (`isLoopbackOnly`), and the worker IPC protocol — rather than inventing parallel machinery.
4. **Backward compatible** — opt-in via a manifest deployment-mode field; default behavior (no tools) is unchanged.

### Trust boundary
The worker thread is **not** trusted with BOS's full authority. It runs with `resourceLimits` and can only reach BOS over the worker IPC channel or loopback HTTP. Tool exposure is the reverse direction: BOS reaches **into** the worker. Trust is enforced by (a) the manifest deployment-mode opt-in, (b) schema validation before dispatch, (c) loopback-scoped auth for any HTTP-based invocation path, and (d) BOS-owned lifecycle cleanup.

---

## 3. Container Design

```
┌────────────────────────────────────────────────────────────────────┐
│  BOS kernel (Next.js server container)                              │
│                                                                    │
│  ┌────────────┐   ┌────────────────────────────┐                    │
│  │ Agent loop  │   │  AssistantTool registry    │                   │
│  │ (agent-loop)│──▶│  (assistantTools() map)    │                   │
│  └─────┬──────┘   └───────────┬────────────────┘                   │
│        │                       │ registers                          │
│        │ execute               │                                    │
│  ┌─────▼───────────────────────┴───────┐                           │
│  │ ServiceToolBridge (new)               │                          │
│  │  - maps serviceId+toolName → executor │                          │
│  │  - validates args (Ajv)               │                          │
│  │  - dispatches via ServiceToolRouter   │                          │
│  └─────┬───────────────────────────────┘                           │
│        │ tool_call / tool_result (worker IPC)                       │
│  ┌─────▼───────────────────────┐   ┌──────────────────────┐        │
│  │ ServiceManager (worker)     │   │  Worker (per service)│        │
│  │  - lifecycle                 │──▶│  - service entrypoint│        │
│  │  - tool registry add/remove  │   │  - declares tools    │        │
│  └─────┬───────────────────────┘   └──────────────────────┘        │
└────────┼────────────────────────────────────────────────────────────┘
         │ (optional) loopback HTTP for backend-agnostic invocation
         └──▶ /api/services/<id>/tools/<name>/invoke (loopback-only)
```

The container has three cooperating components:
- **ServiceToolBridge** (new, `src/lib/agent/`): the single facade that turns a service's tool declarations into `AssistantTool` entries and provides an executor that dispatches invocations to the owning service.
- **ServiceManager** (`src/core/service/`): extended to (a) accept tool declarations on the worker IPC channel, (b) register/unregister tools with the bridge on lifecycle transitions, and (c) route invocations to the worker.
- **Worker** (the service entrypoint): declares tools via a new IPC message; receives `tool_call` and sends `tool_result`/`tool_error`.

---

## 4. Component Design

### 4.1 `ServiceTool` — the core type
A `ServiceTool` is a `ToolDeclaration` (name, description, parameters JSON-schema) plus a reference to its owning service and the invocation transport.

```ts
interface ServiceTool extends ToolDeclaration {
  serviceId: string;
  /** v1: "worker-ipc". Reserved for future: "loopback-http". */
  transport: "worker-ipc" | "loopback-http";
}
```

Maps 1:1 to `AssistantTool` with `execution: "server"` and a generated `execute` that dispatches through the bridge.

### 4.2 `ServiceToolBridge`
New module `src/lib/agent/service-tool-bridge.ts` (hot-reload-safe singleton, mirroring `serviceRegistry()` / `RunManager`).

Responsibilities:
- **Registration**: `registerServiceTools(serviceId, tools)` — validates each declaration (name shape, description present, parameters is a valid JSON-schema via Ajv), stores them, and re-arms the `assistantTools()` cache.
- **Unregistration**: `unregisterServiceTools(serviceId)` — removes all tools owned by that service; called on stop, crash-termination, and uninstall.
- **Execution**: `invoke(serviceId, toolName, args)` — resolves the tool, validates args against its schema (FR-004), dispatches via the transport, returns a string result (matching the `AssistantTool.execute` contract).
- **Listing**: `serviceToolsFor(serviceId)` — for lifecycle bookkeeping and tests.

The bridge is the **only** place that knows the serviceId→toolName→executor mapping. The `assistantTools()` map in `registry.ts` is extended to merge bridge-registered tools; the bridge owns their lifecycle so `registry.ts` stays a pure composition of static + bridge tools.

### 4.3 Tool registration handoff (FR-001)
A service declares tools **at startup**. Two candidate paths were considered (see ADR-003):

- **New worker IPC message `tool_declare`** (Worker→Main): the worker sends `{ type: "tool_declare", tools: ToolDeclaration[] }` after `initialized`. ServiceManager receives it in `handleWorkerMessage`, validates, and calls `registerServiceTools`.
- **Manifest field** `manifest.tools[]`: static declarations in `service.json`. Rejected for v1 because tool schemas are code-shaped (functions, dynamic schema) and the manifest validator would need to ship a full JSON-schema validator into the manifest path.

**Decision (v1)**: the `tool_declare` IPC message. Rationale: keeps tool logic in the service's code where it belongs, avoids bloating the manifest, and the manifest only carries the **opt-in flag** (ADR-002), not the tools themselves.

### 4.4 Invocation transport (FR-003, FR-008)
Two candidate channels (see ADR-001/ADR-004):

- **New worker IPC message types `tool_call` (Main→Worker) / `tool_result` / `tool_error` (Worker→Main)**.
- **Loopback HTTP route** `/api/services/<id>/tools/<name>/invoke`, gated loopback-only.

**Decision (v1)**: **worker IPC messages** as the primary transport. Rationale:
- The worker already has a `Worker` reference and a typed message protocol; no port binding required.
- A service that declares tools need not bind a port just to be callable.
- Request/response correlation uses a `callId` echoed in `tool_result`/`tool_error`, mirroring `waitForMessage`'s pattern but keyed by callId so concurrent calls don't cross-talk.
- **Backend-agnostic escape hatch**: the bridge's `transport` field is designed so a future child-process backend (or a service that prefers HTTP) can swap in a loopback-HTTP dispatcher without changing the `AssistantTool` surface. The loopback route is defined in ADR-004 as the contract for that path but is **not** implemented in v1 unless a service explicitly opts into `transport: "loopback-http"`.

### 4.5 Schema validation (FR-004)
Validation happens in the **BOS kernel**, before dispatch, inside `ServiceToolBridge.invoke`:
- Each declared tool's `parameters` is compiled once at registration with Ajv (`new Ajv({ strict: false })` — same config as `manifestValidator.ts`).
- On invocation, `ajv.validate(schema, args)` runs first; a failed validation returns an in-band `Error: <tool>: <validation message>` string to the agent loop **without** sending a `tool_call` to the worker.
- This guarantees FR-004 (reject before invoking the service) and keeps the worker from seeing malformed input.

### 4.6 Gating (FR-005)
Service tools are gated like built-in server tools, but this now requires a change to the gating model. Because service tools surface as `AssistantTool` entries with `execution: "server"`, the per-step `visibleTools(gate, revealed)` logic and `runServerTool` apply; gate identity is the tool's **name** (same as built-ins). **Gating is now modified**: both `src/lib/assistant/gate.ts` (`gateFor`/`gateFromAgent`) and `src/lib/agent/tool-gate.ts` (`withToolGate`) build their `registryIds` from the static `CAPABILITIES` array, which does **not** include dynamically-registered capability ids. A service tool registered via `registerAdditionalCapabilities` is therefore absent from `registryIds` and would be **auto-executed regardless of allowlist/deferred settings**. Fix: both files source `registryIds` from `listCapabilities()` (which includes dynamic capability ids), so service tools receive the full allowlist/deferred gating like built-ins. This change is tracked in §5 (Modified).

> **tool-gate.ts fix must be per-step.** `src/lib/agent/tool-gate.ts:33` declares `const REGISTRY_IDS = new Set(CAPABILITIES.map((c) => c.id))` as a **module-level constant evaluated once at import time**, before any service registers tools. Swapping the source array alone still yields a frozen import-time snapshot that misses dynamically-registered service tools, so they would be auto-executed on the main-agent path. The fix is therefore to derive the registry-id membership **per step** inside `transformParams` — i.e. call `listCapabilities().map((c) => c.id)` inside `transformParams` (or use a lazy accessor) rather than a module-level `Set` — so the gate reflects the live registry at every step. `gate.ts` (`registryIds` in `gateFromAgent`) is already per-call, so its fix stands as-is.

### 4.7 Lifecycle cleanup (FR-006)
Hooked into existing lifecycle events:
- **Stop** (`ServiceManager.stop`): before/after `dispose`, call `unregisterServiceTools(serviceId)`.
- **Crash-termination** (`handleWorkerMessage` crash / worker `exit`): `ServiceManager` unregisters tools on the worker's `exit` event when it was not an expected exit.
- **Uninstall** (`src/system/marketplace/install/serviceInstaller.ts` uninstall orchestration): the uninstall orchestration emits `service:uninstalled`, which triggers `unregisterServiceTools`.
- **Restart**: stop-then-start naturally unregisters on stop and re-registers on the next `tool_declare`.

The bridge is the single source of truth; ServiceManager calls into it. SC-005 (no stale calls to a stopped service) is satisfied because `unregisterServiceTools` removes the `AssistantTool` entry, so `deps.tools[call.name]` is undefined after removal → the loop returns "unknown tool" for **new** runs. Note that `run.tools` is snapshotted once at run start (`Object.assign(run.tools, assistantTools())` in `src/lib/assistant/start-run.ts`), so an **in-flight** run still holds a stale `AssistantTool` entry; a stale call within an active run degrades gracefully to an in-band "unknown tool" error (no crash). The "no stale calls" guarantee holds for new runs; active runs settle the call in-band.

### 4.8 Trust/auth (FR-009)
- For the **worker-IPC transport**, trust is implicit in the worker lifecycle: only `ServiceManager` holds the `Worker` reference, and only BOS posts `MainToWorkerMessage` to it. No external party can send a `tool_call` — the worker only ever receives messages from its owning manager thread.
- For the **loopback-HTTP transport** (future/opt-in), the route is gated by `isLoopbackOnly(req)` from `src/lib/secrets/auth-scope.ts` — the same trust boundary as `/api/secrets/[service]/verify`. Any request that crossed Bastion (session or `secret:<service>`) is rejected with 403.

### 4.9 Error reporting (FR-007)
The worker sends `tool_error` with `{ callId, message, stack? }`; `ServiceManager` maps it to an in-band `Error: <tool>: <message>` string returned to `runServerTool`, which already guarantees the loop never crashes on a tool failure. A worker crash mid-call is handled by existing crash recovery: the in-flight call settles as failed via the worker `exit` path, and the tool remains available for subsequent calls (re-registered on the restart's `tool_declare`).

---

## 5. File / Module Plan (creates / modifies)

> Only files this feature creates or modifies. Existing files it merely calls into are listed in §6, not here.

### Created
| Path | Purpose |
|---|---|
| `src/lib/agent/service-tool-bridge.ts` | `ServiceToolBridge` singleton: register/unregister/invoke/validate service tools; hot-reload-safe (`globalThis`), mirrors `serviceRegistry()` pattern. |
| `src/core/service/serviceToolTypes.ts` | Shared framework-free types: `ServiceTool`, `ToolDeclaration` re-export, transport enum, `ToolInvocation`/`ToolInvocationResult`. Placed here (not `src/lib/agent/`) so worker-side code can import without pulling agent deps. |
| `src/app/api/services/[id]/tools/[name]/invoke/route.ts` | Loopback-only HTTP route for `transport: "loopback-http"` (defined as the backend-agnostic contract; used in v1 only by opt-in services). |

### Modified
| Path | Change |
|---|---|
| `src/core/service/types.ts` | Add `tool_declare` to `WorkerToMainMessage`; add `tool_call` to `MainToWorkerMessage`; add `tool_result`/`tool_error` to `WorkerToMainMessage`; add `deploymentMode` to `ServiceManifest`. |
| `src/core/service/workerIpc.ts` | Add `waitForToolResult(worker, callId, timeoutMs)` helper (or generalize `waitForMessage`), keyed by callId for concurrent calls. |
| `src/core/service/ServiceManager.ts` | Handle `tool_declare` in `handleWorkerMessage`; register/unregister via bridge on start/stop/crash/uninstall; route `tool_call`/`tool_result`/`tool_error`; pass deployment-mode to validation. |
| `src/core/service/manifestValidator.ts` | Validate `deploymentMode` field (enum: `"default"` \| `"tools"`; reject unknown values); keep backward-compatible (absent ⇒ `"default"`). |
| `src/lib/assistant/registry.ts` | Merge bridge-registered service tools into `assistantTools()` and re-arm the cache on registration/unregistration. |
| `src/lib/assistant/gate.ts` | Source `registryIds` from `listCapabilities()` (which includes dynamic capability ids) instead of the static `CAPABILITIES` array, so service tools are allowlist/deferred-gated by name like built-ins (FR-005). |
| `src/lib/agent/tool-gate.ts` | Derive registry-id membership **per step** inside `transformParams` via `listCapabilities()` — `REGISTRY_IDS` is a module-level constant and must not be a frozen import-time snapshot — so dynamically-registered service tools receive full allowlist/deferred gating like built-ins (FR-005). |

---

## 6. Integration Points (existing mechanisms it calls into — NOT modified)

| Existing mechanism | File | How this feature uses it |
|---|---|---|
| Assistant tool contract | `src/lib/assistant/tools.ts` | `ToolDeclaration`/`AssistantTool` shape; service tools implement `execution: "server"` + `execute`. |
| Server-tool execution | `src/lib/assistant/agent-loop.ts` | `runServerTool` runs the bridge's generated `execute` with kernel guarantees (caught, timed out, in-band errors). |
| Capability registry | `src/lib/agent/capabilities-registry.ts` | `registerAdditionalCapabilities(..., group: 'Service Tools')` for informational capability descriptors (catalog only; not gated by it). |
| Loopback auth | `src/lib/secrets/auth-scope.ts` | `isLoopbackOnly` gates the HTTP invocation route (FR-009). |
| Service registry events | `src/system/marketplace/install/serviceInstaller.ts` + `src/core/service/ServiceRegistry.ts` | `service:uninstalled` (emitted by the uninstall orchestration) / `service:status:changed` drive tool unregistration. |
| Crash recovery | `src/core/service/CrashRecovery.ts` | Worker exit path triggers tool unregistration; restart re-registers. |

---

## 7. ADRs

### ADR-001 — Execution backend for v1: worker threads only
**Status**: Accepted
**Context**: Spec scopes v1 to worker threads (`worker_threads`). Child processes/containers are out of scope.
**Decision**: v1 uses the existing worker-thread backend. The invocation contract is designed backend-agnostic (`ServiceTool.transport`) so a child-process backend can adopt it later without changing the `AssistantTool` surface.
**Consequence**: No CPU isolation for service tools (existing worker-thread limitation documented in `ServiceManager`); acceptable for v1.

### ADR-002 — Deployment-mode field: manifest opt-in
**Status**: Accepted
**Context**: FR-010 requires a manifest field opting a service into tool exposure, defaulting to current no-tool behavior.
**Decision**: `ServiceManifest.deploymentMode: "default" | "tools"`, absent ⇒ `"default"` (no tools). Validated in `manifestValidator`.
**Consequence**: Backward compatible; existing manifests unchanged. A service must opt in before the bridge will accept `tool_declare` messages (a `tool_declare` from a non-opted-in service is rejected/logged).

### ADR-003 — Tool registration path: `tool_declare` IPC message (not manifest field)
**Status**: Accepted
**Context**: FR-001 needs a way for the service to hand `ToolDeclaration`s to BOS at startup. Options: new IPC message, manifest `tools[]` field, or config file.
**Decision**: A new Worker→Main IPC message `tool_declare` carrying `ToolDeclaration[]`.
**Rationale**: Tool schemas are code-shaped and belong in the service code; a manifest field would force a JSON-schema validator into the manifest path and bloat `service.json`. The manifest carries only the opt-in flag (ADR-002).
**Consequence**: Tool declarations are static per-process (declared once at startup), matching the spec assumption.

### ADR-004 — Invocation channel: worker IPC messages (primary), loopback HTTP as backend-agnostic contract
**Status**: Accepted
**Context**: FR-003/FR-008 allow either new worker IPC message types or a loopback HTTP route.
**Decision**: Primary v1 transport is **worker IPC** (`tool_call`/`tool_result`/`tool_error` keyed by `callId`). A loopback HTTP route `/api/services/<id>/tools/<name>/invoke` (gated `isLoopbackOnly`) is defined as the contract for a future child-process backend or an opting service (`transport: "loopback-http"`).
**Rationale**: Worker IPC requires no port binding and reuses the existing typed protocol; HTTP is the escape hatch for backend-agnosticism.
**Consequence**: Two transports to maintain, but the HTTP one is inert unless a service opts in.

### ADR-005 — Schema validation in the BOS kernel before dispatch
**Status**: Accepted
**Context**: FR-004 requires validating invocation args against the declared JSON-schema before dispatching.
**Decision**: Ajv validation lives in `ServiceToolBridge.invoke`, compiled once per tool at registration, run on every invocation. Invalid calls return an in-band error string and never reach the worker.
**Rationale**: Kernel-side validation is the only place that can guarantee FR-004's "reject without invoking the service"; the worker should not be trusted to validate its own input.
**Consequence**: Small per-call Ajv cost; acceptable.

### ADR-006 — Trust: loopback-scoped auth + BOS-owned lifecycle
**Status**: Accepted
**Context**: FR-009 requires that only BOS can invoke a service's tool.
**Decision**: Worker-IPC transport is trusted by construction (only `ServiceManager` owns the `Worker`; only BOS posts to it). The HTTP transport is gated by `isLoopbackOnly`. Tool lifecycle (register/unregister) is owned by BOS via the bridge, never by the service.
**Consequence**: No new credential mechanism; reuses the existing trust boundary.

### ADR-007 — Tool identity for gating: tool name (not capability id)
**Status**: Accepted
**Context**: FR-005 requires gating service tools exactly like built-ins.
**Decision**: Gate identity is the tool **name**, consistent with how server tools are gated today. Gating now requires `src/lib/assistant/gate.ts` (`gateFor`/`gateFromAgent`) and `src/lib/agent/tool-gate.ts` (`withToolGate`) to derive registry-id membership from `listCapabilities()` — which includes dynamically-registered capability ids — rather than the static `CAPABILITIES` array, so a service tool's name is present in `registryIds` and subject to full allowlist/deferred gating. `gate.ts` (`registryIds` in `gateFromAgent`) is already per-call, so its fix stands as-is. `tool-gate.ts` must derive membership **per step** inside `transformParams` (call `listCapabilities().map((c) => c.id)` there, or a lazy accessor), because `REGISTRY_IDS` is a module-level constant evaluated once at import time and cannot be a frozen snapshot. A capability descriptor is also registered (metadata) so the Settings catalog can list it, but the actual allowlist/deferred check uses the tool name sourced from `listCapabilities()`.
**Consequence**: Matches existing server-tool gating once `registryIds` sources from `listCapabilities()` (per-call in `gate.ts`, per-step in `tool-gate.ts`); the capability descriptor is purely informational (catalog display) while the runtime gate uses the tool name.

---

## 8. Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | **Stale tool calls** — a tool is invoked after its service stopped/uninstalled | Low | Medium | `unregisterServiceTools` on stop/crash/uninstall removes the `AssistantTool` entry; loop returns "unknown tool" for removed names (SC-005). |
| R2 | **Concurrent invocation cross-talk** — `tool_result` for call A returned for call B | Low | High | `callId`-keyed `waitForToolResult`; each call waits on its own callId, never a bare type match. |
| R3 | **Unvalidated/colliding tool names** — a service declares a name that shadows a built-in or another service's tool | Medium | High | Bridge validates name shape and detects collisions at registration; a collision is rejected/logged (last-registered wins only within a service; cross-service/builtin collisions rejected). |
| R4 | **Schema-validation bypass** — worker returns malformed output | Low | Low | Output is stringified; malformed output surfaces as an in-band error (FR-007), never crashes the loop. |
| R5 | **Worker crash mid-invocation** — tool call fails while worker dies | Medium | Medium | Existing crash recovery: in-flight call settles as failed on worker `exit`; tool unregistered then re-registered on restart (US3 acceptance 2). |
| R6 | **`assistantTools()` cache staleness** — registration happens after cache is built | Medium | Medium | Bridge re-arms the `registry.ts` cache (clear `cache` on register/unregister); `assistantTools()` recomposes on next call. |
| R7 | **Manifest mode/declaration mismatch** — service declares tools but `deploymentMode !== "tools"` | Low | Low | `tool_declare` from a non-opted-in service is rejected/logged (ADR-002); manifest validated at start (CH-011). |
| R8 | **Memory growth from per-service tool caches** | Low | Low | Bridge stores only declarations + compiled schemas; unregistered on lifecycle end; no unbounded retention. |
| R9 | **Backend-agnostic drift** — HTTP transport diverges from worker IPC semantics | Low | Medium | Both transports funnel through `ServiceToolBridge.invoke` so the `AssistantTool` surface is identical; HTTP route mirrors the IPC contract. |
| R10 | **Pending-invocation waiter leak** — `waitForToolResult` has no timeout/cancellation path keyed by callId | Medium | High | Waiter rejects on worker exit (crash) and times out/cancels on run abort (run-abort); no unresolved promise is left pending. |
| R11 | **Gating-fix ripple into the Settings catalog** — the capability descriptor now appears in `listCapabilities()` (Settings) while the gate also gates by it | Low | Medium | Descriptor is **informational** (catalog display only); the actual runtime gate uses the tool name sourced from `listCapabilities()`. State this explicitly so catalog and gate don't diverge. |
| R12 | **`tool_declare` → first `tool_call` race** — a tool can be invoked in the same step it is declared | Low | Medium | A call to an unknown/not-yet-declared tool returns an in-band "unknown tool" error without waiting; no hang on a not-yet-registered name. |

---

## 9. Open Questions (for plan/tasks)
- ~~Gating identity (tool name vs capability id)~~ — resolved in ADR-007: gate identity is the tool name; the gating change (source `registryIds` from `listCapabilities()` in `gate.ts` and `tool-gate.ts`) is now part of the file plan (§5 Modified).
- Exact naming convention for service tool capability descriptors under the `Service Tools` group (namespace per serviceId to avoid collisions).
- Whether `tool_declare` should be bounded (e.g. max tools per service) to limit abuse — recommend a cap for v1.
- Whether the loopback HTTP route ships in v1 at all, or only the contract is documented (leaning: contract-only unless a service opts in).
