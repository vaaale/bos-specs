# Implementation Plan: Service Tool Exposure (039)

**Branch**: `bos/039-service-tool-exposure` | **Date**: 2026-08-10 | **Spec**: [spec.md](spec.md) | **Design**: [design.md](design.md)

**Input**: Feature specification from `spec.md` (Draft, App Target: `bos-core`). This is a **kernel** feature: it modifies BOS's own service-runtime (`src/core/service/`), assistant tool registry (`src/lib/assistant/`), and capability registry (`src/lib/agent/`). The marketplace service is a **consumer** of the new capability; nothing ships as a marketplace item. Implementation is delegated to the Developer (Claude) sub-agent on the feature branch `bos/039-service-tool-exposure` per Constitution IV/III.

## Summary

Add a native, non-MCP path for a marketplace item's background service (a worker-thread daemon) to expose agent-callable tools to the BOS assistant. A service declares one or more tools (name, description, input JSON-schema) at startup via a new worker IPC message `tool_declare`; BOS validates and registers them into the existing `AssistantTool` registry through a new `ServiceToolBridge`; the assistant invokes them over worker IPC (`tool_call`/`tool_result`/`tool_error` keyed by `callId`); and service tools are governed by the exact same tool-gating/authorization model as built-in tools. Opt-in via a `deploymentMode` manifest field (default `"default"` ⇒ no tools, backward compatible). Tools are removed on service stop, crash, or uninstall.

## Technical Context

**Language/Version**: TypeScript, Node.js backend. BOS is a single-page, server-side-rendered Next.js (App Router) app with a CopilotKit-based assistant.

**Primary Dependencies**:
- **Worker threads**: `node:worker_threads` — marketplace services run as worker threads; v1 uses the existing worker-thread backend exclusively (ADR-001). ServiceManager owns the `Worker` reference and the typed `MainToWorkerMessage`/`WorkerToMainMessage` protocol.
- **Ajv**: JSON-schema validation — already used by `src/core/service/manifestValidator.ts` (`new Ajv({ strict: false })`); reused for service-tool input-schema validation (ADR-005).
- **Existing integration points (called into, NOT modified)**: `src/lib/assistant/tools.ts` (`ToolDeclaration`/`AssistantTool` shape), `src/lib/assistant/agent-loop.ts` (`runServerTool`), `src/lib/agent/capabilities-registry.ts` (`registerAdditionalCapabilities`), `src/lib/secrets/auth-scope.ts` (`isLoopbackOnly`), `src/system/marketplace/install/serviceInstaller.ts` + `src/core/service/ServiceRegistry.ts` (`service:uninstalled` / `service:status:changed`), `src/core/service/CrashRecovery.ts`.

**Storage**: No new persistent storage — tool declarations are static per-process (declared once at startup). `ServiceToolBridge` holds declarations + compiled Ajv schemas in memory; lifecycle-bound (no unbounded retention).

**Testing**: Unit tests under `tests/` via Playwright unit config (`npm run test:unit`, `playwright.unit.config.ts`, `testMatch: /.*\.test\.ts/`, `testDir: ./tests`); e2e under `e2e/*.spec.ts` via `npm run test:e2e`. Existing conventions: `tests/services/workerIpc.test.ts`, `tests/services/ServiceManager.test.ts`, `_worker-fixtures.ts` (real `worker_threads` inline scripts + `installFixtureService`), `_test-env.ts` (`useTestDataDir`/`resetServiceSingletons`), `_stub-server-only.ts`. **Automated testing is a first-class deliverable of this plan (§ Test Strategy).**

**Target Platform**: BrowserOS kernel (`src/`). Marketplace item is only a consumer.

**Performance Goals**: Quality-first (Q3); per-call Ajv validation cost accepted (ADR-005). No hard latency targets up-front.

**Constraints**: Worker reaches BOS only via worker IPC (primary) or loopback HTTP (opt-in, future); no `src/lib` imports from worker code. Tool declarations are static for v1 (spec assumption). No UI for managing service tools in v1.

**Scale/Scope**: Single-user personal OS. v1: worker-thread backend only; child-process/container backends out of scope but contract designed backend-agnostic via `ServiceTool.transport`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Verdict | Notes |
|-----------|---------|-------|
| **I. Spec-Driven** | ✅ PASS | `spec.md` exists under `specs/` and is the source of truth; plan/design are derivative. Implement only after user confirms. |
| **II. Server Authority & SSR Boundary** | ✅ PASS | All worker IPC + loopback-HTTP handling is server-side; loopback route (if implemented) is gated `isLoopbackOnly`; no secrets to client. |
| **III. Always Delegate; Claude Codes** | ✅ PASS | All coding delegated to the Developer (Claude) sub-agent at implement; Build Studio never writes application code. |
| **IV. Minimize Blast Radius** | ✅ PASS | All changes on the `bos/039-service-tool-exposure` feature branch, implemented via `dev_delegate` in an isolated worktree (never the running tree); self-tests (Playwright) must pass before promotion per the constitution's promotable-change requirement. |
| **V. The VFS Is Not the Source** | ✅ PASS | BOS source is edited only under `src/` via the developer sub-agent; no VFS file tools touch `src/`. Runtime state stays under `./data` (gitignored). |
| **VI. Specs & Docs Stay in Sync** | ✅ PASS | spec.md, checklists/requirements.md, design.md, plan.md updated together; drift recorded in `specs/discrepancies.md`; `docs/dev` noted where the feature adds a capability. |
| **VII. Respect Boundaries** | ✅ PASS | No secrets/`package.json`/lockfile/build-config changes; destructive ops (tool unregistration on uninstall) follow existing orchestration; `npx tsc --noEmit` and `npm run lint` are mandatory pre-promote gates; no `npm run build` while `next dev` runs. |

**Complexity Tracking**: No violations — `bos-core` classification is correct (modifies `src/` kernel behavior, not a marketplace item/app). Changes are additive to existing registries/protocols, and blast radius is bounded to the feature branch per IV. No justification needed.

## Project Structure

### Documentation (this feature)

```text
specs/user-specs/039-service-tool-exposure/
├── spec.md              # Feature specification (agreed, Draft)
├── checklists/requirements.md
├── design.md            # Architect classification, C4 container/component design, ADRs
├── plan.md              # This file
└── tasks.md             # Phase 2 output (created at implement gate)
```

### Source Code — BOS kernel (`src/`)

> Created files are NEW; modified files are EDITS to existing source. All paths are real `src/` paths. This is the implementation plan — no files are created by this plan; they are produced by the Developer at implement.

**Created:**

| Path | Purpose |
|---|---|
| `src/lib/agent/service-tool-bridge.ts` | `ServiceToolBridge` singleton: `registerServiceTools` / `unregisterServiceTools` / `invoke` / `serviceToolsFor`. Hot-reload-safe (`globalThis`), mirrors `serviceRegistry()`/`RunManager` pattern. Validates declarations (name shape, description, parameters JSON-schema via Ajv) and detects collisions; re-arms the `assistantTools()` cache on register/unregister. |
| `src/core/service/serviceToolTypes.ts` | Shared framework-free types: `ServiceTool`, `ToolDeclaration` re-export, transport enum (`"worker-ipc"` \| `"loopback-http"`), `ToolInvocation`/`ToolInvocationResult`. Placed here (not `src/lib/agent/`) so worker-side code imports without pulling agent deps. |
| `src/app/api/services/[id]/tools/[name]/invoke/route.ts` | Loopback-only HTTP route for `transport: "loopback-http"` (backend-agnostic contract; used in v1 only by opt-in services). Gated by `isLoopbackOnly(req)` from `src/lib/secrets/auth-scope.ts`. |

**Modified:**

| Path | Change |
|---|---|
| `src/core/service/types.ts` | Add `tool_declare` to `WorkerToMainMessage`; `tool_call` to `MainToWorkerMessage`; `tool_result`/`tool_error` to `WorkerToMainMessage`; `deploymentMode` to `ServiceManifest`. |
| `src/core/service/workerIpc.ts` | Add `waitForToolResult(worker, callId, timeoutMs)` helper (generalize `waitForMessage`), keyed by `callId` so concurrent calls don't cross-talk; rejects on worker `exit` and on timeout/run-abort (R10). |
| `src/core/service/ServiceManager.ts` | Handle `tool_declare` in `handleWorkerMessage` (validate + `registerServiceTools`); register on start, unregister on stop/crash/uninstall; route `tool_call`/`tool_result`/`tool_error`; pass deployment-mode to validation; reject `tool_declare` from non-opted-in services (`deploymentMode !== "tools"`). |
| `src/core/service/manifestValidator.ts` | Validate `deploymentMode` (enum `"default"` \| `"tools"`; reject unknown); absent ⇒ `"default"` (backward compatible). |
| `src/lib/assistant/registry.ts` | Merge bridge-registered service tools into `assistantTools()`; re-arm the cache on registration/unregistration. |
| `src/lib/assistant/gate.ts` | Source `registryIds` from `listCapabilities()` (includes dynamic capability ids) instead of the static `CAPABILITIES` array, so service tools are allowlist/deferred-gated by name like built-ins (FR-005). `gateFromAgent` is already per-call; fix stands. |
| `src/lib/agent/tool-gate.ts` | **Per-step fix**: `REGISTRY_IDS` (line 27, `const REGISTRY_IDS = new Set(CAPABILITIES.map((c) => c.id))`) is a module-level constant evaluated once at import time and MUST NOT be a frozen import-time snapshot. Derive membership per step inside `transformParams` via `listCapabilities().map((c) => c.id)` (or a lazy accessor), so dynamically-registered service tools receive full allowlist/deferred gating (FR-005). |

### Test fixtures (created for this feature)

```text
tests/services/_tool-service-fixtures.ts   # stub marketplace item/service that declares tools — reusable across unit/integration/e2e
e2e/039-service-tool-exposure.spec.ts      # e2e: assistant calls a service tool end-to-end
```

## Design Notes (folded from design.md)

- **Classification**: `bos-core` (kernel capability). Not a marketplace item, not a builtin app — modifies how BOS's own service-runtime, assistant tool registry, and capability registry behave. Matches spec.md App Target.
- **Container design**: three cooperating components — `ServiceToolBridge` (new facade turning service tool declarations into `AssistantTool` entries + executor), `ServiceManager` (accepts `tool_declare` on worker IPC, registers/unregisters on lifecycle, routes invocations), `Worker` (service entrypoint declaring tools, receiving `tool_call`, sending `tool_result`/`tool_error`). Optional loopback-HTTP route for backend-agnostic invocation.
- **ADR-001 (worker threads only for v1)**: existing worker-thread backend; contract backend-agnostic via `ServiceTool.transport`; no CPU isolation for service tools (existing limitation), acceptable.
- **ADR-002 (deployment-mode manifest opt-in)**: `ServiceManifest.deploymentMode: "default" | "tools"`, absent ⇒ `"default"`; validated in `manifestValidator`; backward compatible.
- **ADR-003 (registration via `tool_declare` IPC, not manifest field)**: tool schemas are code-shaped, belong in service code; manifest carries only the opt-in flag; static per-process declarations.
- **ADR-004 (invocation via worker IPC primary, loopback HTTP as contract)**: `tool_call`/`tool_result`/`tool_error` keyed by `callId`; HTTP route `/api/services/<id>/tools/<name>/invoke` gated `isLoopbackOnly` is the escape hatch, inert unless a service opts in.
- **ADR-005 (kernel-side schema validation before dispatch)**: Ajv compiled once per tool at registration, run per invocation; invalid calls return in-band error and never reach the worker.
- **ADR-006 (trust: loopback-scoped auth + BOS-owned lifecycle)**: worker-IPC trusted by construction (only ServiceManager owns the `Worker`); HTTP gated by `isLoopbackOnly`; tool lifecycle owned by BOS via the bridge.
- **ADR-007 (gate identity = tool name)**: gating change sources `registryIds` from `listCapabilities()` in `gate.ts` (per-call) and `tool-gate.ts` (per-step inside `transformParams`, because `REGISTRY_IDS` is a module-level constant). A capability descriptor is registered (informational catalog display only); the runtime gate uses the tool name.
- **Lifecycle cleanup (FR-006)**: `unregisterServiceTools` on stop (`ServiceManager.stop`), crash (`worker.on("exit")` unexpected exit → `handleCrash`), uninstall (`service:uninstalled` via `serviceInstaller`), restart (stop-then-start). `run.tools` is snapshotted once at run start (`src/lib/assistant/start-run.ts`): `Object.assign(run.tools, assistantTools())`, and the loop dispatches against that same `run.tools` object, extended only by `addSurfaceTools`. Because ServiceManager unregisters from the *registry* (which only affects `assistantTools()` for NEW runs), an in-flight run KEEPS the stale tool entry for the remainder of that run — it does NOT become "unknown tool" mid-run; the loop dispatches against the stale `run.tools` entry and a call to a stopped service degrades to an in-band tool error (no crash). Only NEW runs get the pruned set. The no-stale-call guarantee (SC-005) holds for new runs; in-flight runs degrade gracefully to an in-band error, not "unknown tool".
- **Error reporting (FR-007)**: worker sends `tool_error` with `{ callId, message, stack? }`; ServiceManager maps to in-band `Error: <tool>: <message>` returned to `runServerTool`, which already guarantees the loop never crashes; worker crash mid-call settles as failed via `exit` path (US3 acceptance 2).

## Test Strategy (mandatory, first-class)

> Automated testing is **non-negotiable** for this feature. Every FR and user story maps to concrete tests below. Tests are written by the Developer alongside the implementation (TDD where practical) and run via the existing harnesses: unit tests `npm run test:unit` (`playwright.unit.config.ts`, `tests/**/*.test.ts`), e2e `npm run test:e2e` (`e2e/*.spec.ts`), plus mandatory typecheck (`npx tsc --noEmit`) and lint (`npm run lint`) gates.

### FR → Test mapping

| Requirement | Test file (unit) | What it verifies |
|---|---|---|
| **FR-001** (service declares tools at startup) | `tests/services/ServiceToolBridge.test.ts`, `tests/services/ServiceManager.test.ts` | `tool_declare` IPC handled in `handleWorkerMessage`; `registerServiceTools(serviceId, tools)` stores declarations; a worker fixture that posts `tool_declare` after `initialized` results in registered tools. |
| **FR-002** (surface into `AssistantTool` registry) | `tests/services/ServiceToolBridge.test.ts`, `tests/lib/assistant/registry.test.ts` (or `tests/assistant/`) | After registration, `assistantTools()` includes the service tool as an `AssistantTool` with `execution: "server"`; cache re-armed on register/unregister. |
| **FR-003** (invoke via worker IPC) | `tests/services/workerIpc.test.ts`, `tests/services/ServiceManager.test.ts` | `invoke(serviceId, toolName, args)` dispatches `tool_call`; `tool_result` with matching `callId` resolves; result returns to the loop. |
| **FR-004** (schema validation before dispatch) | `tests/services/ServiceToolBridge.test.ts` | Valid args pass Ajv and dispatch; invalid args return an in-band validation-error string and **no `tool_call` is dispatched** (assert worker never receives it). |
| **FR-005** (gate service tools like built-ins) | `tests/services/tool-gate.test.ts`, `tests/services/gate.test.ts` (or `tests/assistant/`) | Dynamically-registered service tool names are present in `registryIds`; allowlist/deferred gating applies; `tool-gate.ts` fix is **per-step** (module-level `REGISTRY_IDS` is NOT a frozen import-time snapshot — membership derives from `listCapabilities()` inside `transformParams`). |
| **FR-006** (remove tools on stop/crash/uninstall) | `tests/services/ServiceManager.test.ts` | Tools disappear from registry after stop, crash (`worker exit`), and uninstall (`service:uninstalled` via `serviceInstaller`); `assistantTools()` no longer includes them. |
| **FR-007** (report tool error without crash) | `tests/services/ServiceManager.test.ts` | `tool_error` maps to in-band `Error: <tool>: <message>`; BOS process stays healthy. |
| **FR-008** (invocation contract over worker IPC + loopback HTTP) | `tests/services/workerIpc.test.ts` | `tool_call`/`tool_result`/`tool_error` message types exist and are `callId`-keyed; loopback route defined (if implemented) mirrors the IPC contract. |
| **FR-009** (loopback-scoped auth) | `tests/services/auth-scope.test.ts` (existing, extended) | HTTP route (if implemented) rejects requests that crossed Bastion with 403; only loopback reaches the worker. |
| **FR-010** (deployment-mode opt-in) | `tests/services/manifestValidator.test.ts` (existing, extended) | `deploymentMode: "tools"` accepted; `"default"`/absent accepted; unknown rejected; `tool_declare` from a non-opted-in service (`deploymentMode !== "tools"`) is rejected/logged and tools are NOT registered. |

### User story → acceptance scenario mapping

| User story | Acceptance scenario | Test |
|---|---|---|
| **US1** — service exposes a native tool | US1.1 tool appears in registry on service start | Unit: FR-001/FR-002. Integration: real worker-thread service declaring tools → tools surface in `AssistantTool` registry. |
| US1 | US1.2 assistant calls with valid input → result | Unit: FR-003. Integration: agent loop invokes via `runServerTool` → result returns. E2E: `e2e/039-service-tool-exposure.spec.ts`. |
| US1 | US1.3 missing required field rejected, service not invoked | Unit: FR-004 schema-rejection path (assert no `tool_call` dispatched). |
| **US2** — service tools respect tool gating | US2.1 deferred/blocked tool not auto-executed | Unit: FR-005 gating tests (allowlist + deferred). |
| US2 | US2.2 tool disappears on stop/uninstall | Unit: FR-006. |
| **US3** — invocation isolated & crash-safe | US3.1 throwing tool → tool-error result, BOS not crashed | Unit: FR-007. Integration: worker fixture whose tool throws; assert in-band error + process healthy. |
| US3 | US3.2 crashing worker → in-flight call fails, tool re-available after restart | Integration: crash-recovery path; worker restarts, `tool_declare` re-registers, subsequent call succeeds. |

### 1. Unit tests — `ServiceToolBridge` (`src/lib/agent/service-tool-bridge.ts`)
File: `tests/services/ServiceToolBridge.test.ts` (new).
- **registration**: `registerServiceTools(serviceId, tools)` stores valid tools; returns them from `serviceToolsFor(serviceId)`.
- **unregistration**: `unregisterServiceTools(serviceId)` removes all tools for that service; `serviceToolsFor` empty; `assistantTools()` cache re-armed.
- **schema validation (valid args)**: valid args pass Ajv and dispatch `tool_call`.
- **schema validation (invalid args)**: missing/type-mismatched args return in-band validation-error string; **no `tool_call` dispatched** (FR-004).
- **collision detection**: declaring a name that shadows a built-in or another service's tool is rejected/logged; last-registered wins only within a service (R3).
- **cache re-arming**: registration after `assistantTools()` cache built → next call recomposes and includes the tool (R6).

### 2. Unit tests — gating fix (`src/lib/assistant/gate.ts` AND `src/lib/agent/tool-gate.ts`)
Files: `tests/services/gate.test.ts` and `tests/services/tool-gate.test.ts` (new; or colocated under `tests/assistant/` matching existing layout).
- Dynamically-registered service tools are allowlist/deferred-gated by **name** like built-ins (FR-005).
- `tool-gate.ts` fix is **per-step**: assert that registering a tool *after* import-time still results in gating — i.e. membership derives from `listCapabilities()` inside `transformParams`, and module-level `REGISTRY_IDS` is not a frozen import-time snapshot. Test: register a service tool at runtime, then call `withToolGate`/`transformParams` and verify the tool is gated (deferred until revealed / allowlisted).
- `gate.ts` (`registryIds` in `gateFromAgent`) sources from `listCapabilities()` (per-call) — service tool name present in `registryIds`.
- Regression: built-in tools still gated identically; description overrides still apply.

### 3. Unit tests — worker IPC protocol (`src/core/service/workerIpc.ts`)
File: `tests/services/workerIpc.test.ts` (extend existing, real `worker_threads` inline scripts per convention).
- `waitForToolResult(worker, callId, timeoutMs)` resolves on matching `callId` (not bare type match).
- **callId-keyed concurrency**: two concurrent tool calls with distinct `callId`s do not cross-talk — worker echoes `tool_result` for A and B; each promise resolves with its own result (R2).
- **timeout/cancellation on run abort** (R10): aborting a run rejects the pending waiter with a timeout/abort error; no unresolved promise leaks.
- **cancellation on worker exit** (R10): worker exits before replying → waiter rejects (`worker exited before...`), matching existing `waitForMessage` semantics.

### 4. Unit tests — `ServiceManager` lifecycle (`src/core/service/ServiceManager.ts`)
File: `tests/services/ServiceManager.test.ts` (extend existing, real worker fixtures via `_worker-fixtures.ts` + `_test-env.ts`).
- `tool_declare` handling: a worker posting `tool_declare` after `initialized` → tools registered via bridge.
- **register on start**: starting a service with `deploymentMode: "tools"` that declares tools → tools appear in registry.
- **unregister on stop**: `ServiceManager.stop` → `unregisterServiceTools(serviceId)`; tools removed.
- **unregister on crash**: unexpected worker `exit` → `handleCrash` → tools removed; restart re-registers on next `tool_declare`.
- **unregister on uninstall**: `service:uninstalled` (via `serviceInstaller` orchestration) → tools removed.
- **non-opted-in rejection**: `tool_declare` from a service with `deploymentMode !== "tools"` is rejected/logged; tools NOT registered (ADR-002, R7).

### 5. Integration tests — real worker-thread service → agent loop
File: `tests/services/integration.test.ts` (extend existing) or a new `tests/services/tool-integration.test.ts`.
- A real worker-thread service declares a tool → tool surfaces in `AssistantTool` registry.
- Agent loop invokes it via `runServerTool` → service executes → result returns.
- **Schema-rejection path**: invalid args rejected in the kernel; no `tool_call` dispatched (assert via the worker fixture that it never receives a call).
- Crash path: a worker that throws mid-call → in-band tool error; BOS healthy; after restart the tool is re-registered and a subsequent call succeeds (US3).

### 6. E2E / self-test (Playwright)

> **Parallelism note**: `playwright.config.ts` sets `fullyParallel: true`, but this e2e installs a service that mutates global singleton state (`__bosServiceRegistry`, worker threads) and the filesystem data dir. To avoid cross-spec interference, the 039 spec MUST be run with `test.describe.serial` (or a unique service id / isolated data dir), so concurrent e2e specs cannot collide on the shared service host.
File: `e2e/039-service-tool-exposure.spec.ts` (new, following the `e2e/<id>.spec.ts` naming convention; imports `test` from `./fixtures`).
- **End-to-end scenario proving a service tool is callable by the assistant**: install the stub marketplace item whose service declares a tool (via the fixture), ask the assistant to call it by name, and verify the assistant receives the result in the transcript.
- Per the constitution's **self-test requirement for promotable changes**, this e2e test must pass before promotion (Constitution IV). Run via `npm run test:e2e`.

### 7. Typecheck + lint gates (mandatory pre-promote)
- `npx tsc --noEmit` must pass on the feature branch before any promote.
- `npm run lint` must pass on changed files.
- Per Constitution VII, do **not** run `npm run build` while `next dev` is running.
- These gates are enforced at the end of Phase 2 (implement) and again before promote (Constitution: "A change is complete only when: the spec and docs are updated, typecheck and lint pass, and (for promotable changes) self-tests pass").

### 8. Test fixtures (shared)
File: `tests/services/_tool-service-fixtures.ts` (new).
- A stub marketplace item/service that declares tools (a `deploymentMode: "tools"` manifest + a worker entrypoint that posts `tool_declare` and handles `tool_call`/`tool_result`), following the existing `_worker-fixtures.ts` convention (`installFixtureService`, `parentPort` guard).
- Reusable across unit (ServiceManager/ServiceToolBridge), integration, and e2e tests (e2e installs it via the marketplace/uninstall orchestration).

## Phase Breakdown (mapped to pipeline)

**Pipeline**: constitution → specify → clarify → plan → tasks → implement → converge.

- **Phase 0 — Research** (plan): source-path verification completed (all modified files exist; created files absent). Design analysis folded into this plan (§ Design Notes). No separate `research.md` needed — research (worker IPC protocol, gating model, Ajv, lifecycle events) is captured in design.md + this plan. **Constitution Check re-run: PASS.**
- **Phase 1 — Data-model / contracts** (plan): the data model is defined in `design.md` §4 and folded here: `ServiceTool` (ToolDeclaration + serviceId + transport), `ToolInvocation`/`ToolInvocationResult`, `WorkerToMainMessage`/`MainToWorkerMessage` extensions, `ServiceManifest.deploymentMode`, `ToolDeclaration` re-export in `serviceToolTypes.ts`. `contracts/` not needed separately — the worker IPC message shapes + HTTP route contract are documented in design.md ADR-003/004/005. **Constitution Check re-run: PASS.**
- **Phase 2 — Tasks** (tasks gate → implement): `tasks.md` is produced at the implement gate, sequenced as:
  1. `serviceToolTypes.ts` (types) → `types.ts` (message + manifest extensions) → `manifestValidator.ts` (deploymentMode).
  2. `ServiceToolBridge` (register/unregister/invoke/validate) + `registry.ts` merge + cache re-arm.
  3. `workerIpc.ts` `waitForToolResult` (callId-keyed).
  4. `ServiceManager` lifecycle wiring (tool_declare, register on start, unregister on stop/crash/uninstall, route tool_call/tool_result/tool_error).
  5. Gating fix: `gate.ts` (per-call `listCapabilities()`) + `tool-gate.ts` (per-step `listCapabilities()` in `transformParams`, removing the module-level frozen `REGISTRY_IDS`).
  6. Loopback HTTP route (contract; inert unless a service opts in).
  7. Test fixtures + all unit/integration tests (§ Test Strategy 1–5).
  8. E2E `e2e/039-service-tool-exposure.spec.ts` (§ 6).
  9. Typecheck + lint gates; docs sync (`docs/dev`) per Constitution VI; self-test before promote (Constitution IV).
- **Converge** (`/speckit.converge`): check codebase-vs-spec drift; record any drift in `specs/discrepancies.md`; verify spec/docs/tests all updated in the same change (Constitution VI).
