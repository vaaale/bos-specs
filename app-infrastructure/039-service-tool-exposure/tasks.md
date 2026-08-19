# Tasks — 039 Service Tool Exposure

**Feature:** 039 — Service Tool Exposure
**Status:** Ready for implement
**Depends on:** plan.md (test strategy § Test Strategy 1–8), design.md (§ File/Module Plan), spec.md (US1/US2/US3)

This file decomposes `plan.md` into dependency-ordered, developer-executable tasks grouped by user story (US1/US2/US3 from `spec.md`), with exact file paths from `design.md`'s File/Module Plan and `plan.md`'s Project Structure. Tasks marked **[P]** are parallelizable within their phase. All tasks are executed against the BOS source tree (the `039-service-tool-exposure` feature branch); no `service.json` or source files are created here — this file is instructions for the Developer during `implement`.

> **Logging requirement (first-class):** All new modules emit structured records through the BOS logging system `@/lib/logging` (spec 017-central-logging), using the established component convention (`const LOG = "services.tool-bridge"`), distinct from ServiceManager's raw worker stdout/stderr capture (`appendServiceLog`, `src/core/service/ServiceManager.ts:57`). Logging is threaded through every task that touches registration, dispatch, resolution, gating, timeout/cancellation, and worker-exit cleanup.

---

## PHASE 1 — Foundational (blocks everything)

### T001 — Define ServiceTool data model + message types
- **Files:** `src/core/service/serviceToolTypes.ts` (new), `src/core/service/types.ts` (extend)
- **Story:** US1/US2/US3 (data model prerequisite)
- **Work:**
  - Create `src/core/service/serviceToolTypes.ts` exporting:
    - `ToolDeclaration` — `{ name, description, inputSchema (JSON schema), categories?, version? }`
    - `ServiceTool` — `{ declaration: ToolDeclaration; serviceId: string; transport: "worker-ipc" }`
    - `ToolInvocation` — `{ callId: string; name: string; args: unknown }`
    - `ToolInvocationResult` — `{ callId: string; result?: unknown; error?: { code: string; message: string; stack?: string } }`
    - `deploymentMode` field on the tool-bearing manifest shape (`"tools"` opt-in).
  - Re-export `ToolDeclaration` from `types.ts` for the worker-facing API.
- **Acceptance:** All types compile; `serviceToolTypes.ts` is self-contained and importable from both `ServiceManager` and worker fixtures.

### T002 — Define worker IPC message types (tool_declare / tool_call / tool_result / tool_error)
- **File:** `src/core/service/workerIpc.ts`
- **Story:** US1/US2/US3 (IPC contract prerequisite)
- **Work:**
  - Extend `WorkerToMainMessage` with `{ type: "tool_declare"; payload: { declaration: ToolDeclaration; callId: string } }` and `{ type: "tool_result"; payload: ToolInvocationResult }` / `{ type: "tool_error"; payload: ToolInvocationResult }`.
  - Extend `MainToWorkerMessage` with `{ type: "tool_call"; payload: ToolInvocation }`.
  - Add a `callId` field to the message envelope so every tool message is uniquely keyed (R2 isolation).
- **Acceptance:** Message union types compile; `callId` is required on all tool messages; existing non-tool messages unchanged.

### T003 — Add `deploymentMode` to service manifest schema + validate "tools" mode
- **File:** `src/core/service/manifestValidator.ts`
- **Story:** US1 (opt-in gate), R7/ADR-002
- **Work:**
  - Add optional `deploymentMode` (`"default" | "tools"`, default `"default"`) to the manifest schema (Ajv).
  - Validate that `deploymentMode: "tools"` is a valid enumerated value; reject unknown values.
- **Acceptance:** A manifest with `deploymentMode: "tools"` validates; an invalid mode string is rejected with a clear error.

### T004 — Establish LOG component constant + logging helper for new modules
- **Files:** `src/core/service/serviceToolTypes.ts` (or a new `src/core/service/toolLog.ts`), `src/core/service/workerIpc.ts`, `src/core/service/ServiceManager.ts`, `src/lib/agent/service-tool-bridge.ts`
- **Story:** All (logging requirement)
- **Work:**
  - Declare `const LOG = "services.tool-bridge";` as the canonical component constant for the new tool-bridge modules (mirroring the `"assistant.a2ui"`, `"memory.slow-loop"`, `"skills"` convention).
  - Wire `logger()` from `@/lib/logging` in each module:
    - `logger().info(LOG, "tool_declare:received", { serviceId, name, callId })`
    - `logger().info(LOG, "tool:registered", { serviceId, name })`
    - `logger().info(LOG, "tool:unregistered", { serviceId, name })`
    - `logger().info(LOG, "tool_call:dispatch", { serviceId, name, callId })`
    - `logger().info(LOG, "tool_call:resolved", { serviceId, name, callId, durationMs, ok })` / `logger().error(...)` on error result
    - `logger().warn(LOG, "tool_call:schema-rejected", { serviceId, name, callId, error })` (R7)
    - `logger().warn(LOG, "tool_call:timeout", { serviceId, name, callId })` / `logger().warn(LOG, "tool_call:cancelled", ...)` (R10)
    - `logger().info(LOG, "worker:exit-cleanup", { serviceId, toolCount })`
- **Acceptance:** All new modules import `logger` from `@/lib/logging`; component constant `services.tool-bridge` is used consistently; structured `{ data }` carries serviceId/name/callId/durationMs.

### T005 — service-tool-bridge globalThis singleton scaffolding
- **File:** `src/lib/agent/service-tool-bridge.ts` (new)
- **Story:** US1/US2/US3 (bridge scaffold)
- **Work:**
  - Create a `globalThis`-backed singleton `ServiceToolBridge` exposing `registerTool(serviceId, declaration)`, `unregisterTool(serviceId, name)`, `unregisterServiceTools(serviceId)`, and a read-only `registry` (Map keyed by `serviceId:name`).
  - Initial scaffold: empty registry, no-op handlers, typed against `serviceToolTypes.ts`.
- **Acceptance:** Singleton is importable from server-only modules; registry is empty at import; type-safe.

### T006 — Unit-test harness: worker fixtures for a tool-declaring stub service
- **Files:** `tests/services/_worker-fixtures.ts` (extend), `tests/services/_stub-server-only.ts` (new), `tests/services/_tool-service-fixtures.ts` (new)
- **Story:** All (test harness prerequisite)
- **Work:**
  - Add a stub worker service fixture that posts `tool_declare` after `initialized` and handles `tool_call` → `tool_result`/`tool_error`, following the existing `_worker-fixtures.ts` convention (`installFixtureService`, `parentPort` guard).
  - Add `_stub-server-only.ts` for any server-only dependencies the new modules pull in (e.g. `server-only` import shim in tests).
  - Reusable across unit/integration/e2e (plan § Test Strategy 8).
- **Acceptance:** Fixtures compile and load in the test environment; a stub service can declare a tool and echo results.

> **CHECKPOINT — foundation ready.** All types, IPC contract, manifest validation, logging component, bridge scaffold, and test fixtures exist and compile. No functionality is wired yet; nothing downstream can start until this phase is green.

---

## PHASE 2 — US1 (P1, MVP): Service exposes a native tool

### T010 — [P] Unit test: bridge register/unregister + collision + schema validation
- **File:** `tests/agent/service-tool-bridge.test.ts` (new)
- **Story:** US1 (FR-001)
- **Work:**
  - `registerTool` adds to registry; duplicate `serviceId:name` collides and is rejected/logged.
  - `unregisterTool` / `unregisterServiceTools` remove correctly.
  - `inputSchema` validation path rejects malformed schemas.
  - Assert `logger().info(LOG, "tool:registered", ...)` is called on successful registration (logging requirement).
- **Acceptance:** All assertions pass; registration emits a structured `logger().info` record.

### T011 — [P] Unit test: tool_declare → registry surfaces tool (re-arm assistantTools)
- **File:** `tests/agent/service-tool-bridge.test.ts` (new)
- **Story:** US1 (FR-001)
- **Work:**
  - After `registerTool`, the `AssistantTool` registry (via the bridge's cache re-arm) reflects the new tool; a subsequent `listCapabilities()`/`toolsFor` query includes it.
- **Acceptance:** Tool is discoverable by the agent tool registry immediately after `tool_declare`.

### T012 — Unit test: schema-rejection path (no tool_call dispatched)
- **File:** `tests/agent/service-tool-bridge.test.ts` (new)
- **Story:** US1 (FR-004, R7)
- **Work:**
  - Invalid args against `inputSchema` are rejected in the kernel; assert the worker fixture never receives a `tool_call` (no dispatch).
  - Assert `logger().warn(LOG, "tool_call:schema-rejected", ...)` is emitted.
- **Acceptance:** Invalid args are rejected before dispatch; a warn log is recorded; worker sees no call.

### T013 — [P] Implement ServiceToolBridge register/unregister + cache re-arm
- **File:** `src/lib/agent/service-tool-bridge.ts`
- **Story:** US1 (FR-001)
- **Work:**
  - Fill in `registerTool`/`unregisterTool`/`unregisterServiceTools` against the `globalThis` registry.
  - On register/unregister, re-arm the `AssistantTool` cache so the agent loop sees the new/changed tool set.
  - Emit `tool:registered` / `tool:unregistered` info logs (component `services.tool-bridge`).
- **Acceptance:** Registration and removal mutate the registry and re-arm the assistant tool cache; logs emitted.

### T014 — Implement tool_declare handler in ServiceManager (opt-in gate)
- **File:** `src/core/service/ServiceManager.ts`
- **Story:** US1 (FR-001, R7/ADR-002)
- **Work:**
  - Handle `tool_declare` from a worker: only register when `deploymentMode === "tools"`; otherwise reject + `logger().warn` (non-opted-in rejection).
  - Route to the bridge (`registerTool`).
- **Acceptance:** A `tools`-mode service's `tool_declare` registers the tool; a non-opted-in service's declaration is rejected and logged, with no registration.

### T015 — Implement tool_call dispatch + waitForToolResult (callId-keyed)
- **File:** `src/core/service/workerIpc.ts`
- **Story:** US1 (FR-002, R2)
- **Work:**
  - Add `sendToolCall(worker, invocation)` and `waitForToolResult(worker, callId, timeoutMs)`.
  - `waitForToolResult` resolves on a matching `callId` (not bare type match); rejects on worker exit/timeout.
- **Acceptance:** Dispatch sends a `tool_call`; a matching `tool_result`/`tool_error` resolves the pending promise keyed by `callId`.

### T016 — Wire tool_call resolution into agent-loop runServerTool dispatch
- **File:** `src/lib/assistant/agent-loop.ts` (runServerTool dispatch path)
- **Story:** US1 (FR-002)
- **Work:**
  - When the agent invokes a service-exposed tool via `runServerTool`, dispatch through the bridge → `sendToolCall` → `waitForToolResult`, returning the resolved result to the agent loop.
- **Acceptance:** Agent-loop tool invocation on a service tool returns the worker's result to the model.

### T017 — Logging at register/dispatch/resolve (component services.tool-bridge)
- **Files:** `src/lib/agent/service-tool-bridge.ts`, `src/core/service/ServiceManager.ts`, `src/core/service/workerIpc.ts`
- **Story:** US1 (logging requirement)
- **Work:**
  - Ensure `tool_declare:received`, `tool:registered`, `tool_call:dispatch`, `tool_call:resolved` info records carry `{ serviceId, name, callId, durationMs, ok }`.
- **Acceptance:** Key events emit structured `logger()` records under `services.tool-bridge`, distinct from raw stdout capture.

### T018 — Integration test: real worker service → registry → runServerTool → result
- **File:** `tests/services/tool-integration.test.ts` (new; per plan § Test Strategy 5)
- **Story:** US1 (FR-001, FR-002)
- **Work:**
  - Spin up a real worker-thread service that declares a tool → tool surfaces in the `AssistantTool` registry.
  - Agent loop invokes it via `runServerTool` → service executes → result returns.
  - Include the schema-rejection path (invalid args rejected, worker never called).
- **Acceptance:** Full US1 round-trip passes; schema-rejection verified in-band.

> **CHECKPOINT — US1 functional (MVP).** A `deploymentMode: "tools"` service exposes a native tool end-to-end (declare → register → agent invokes → result), with logging on all key events.

---

## PHASE 3 — US2 (P2): Gating + lifecycle

### T020 — [P] Unit test: gate.ts gating of dynamically-registered service tools
- **File:** `tests/services/gate.test.ts` (new; or colocated `tests/assistant/gate.test.ts`)
- **Story:** US2 (FR-005)
- **Work:**
  - Dynamically-registered service tools are allowlist/deferred-gated by **name** like built-ins.
  - `gate.ts` `registryIds` in `gateFromAgent` sources from `listCapabilities()` per call, so a service tool name is present in `registryIds`.
  - Regression: built-in tools still gated identically; description overrides still apply.
- **Acceptance:** Service tools are gated by allowlist + deferred; built-ins unaffected.

### T021 — [P] Unit test: tool-gate.ts per-step gating fix
- **File:** `tests/services/tool-gate.test.ts` (new; or `tests/assistant/tool-gate.test.ts`)
- **Story:** US2 (FR-005)
- **Work:**
  - `REGISTRY_IDS` must derive from `listCapabilities()` **inside `transformParams`**, NOT a frozen module-level constant.
  - Test: register a service tool at runtime, then call `withToolGate`/`transformParams` → the tool is gated (deferred until revealed / allowlisted), proving per-step membership.
- **Acceptance:** A tool registered after import-time is still gated per-step; the module-level frozen `REGISTRY_IDS` is not relied upon.

### T022 — Modify `src/lib/assistant/gate.ts` to source registryIds from listCapabilities()
- **File:** `src/lib/assistant/gate.ts`
- **Story:** US2 (FR-005)
- **Work:**
  - Replace the static `new Set(CAPABILITIES.map((c) => c.id))` in `gateFromAgent` with a per-call `listCapabilities()`-derived set so dynamically-registered service tool ids are included.
- **Acceptance:** `registryIds` reflects dynamic service tools at call time.

### T023 — Modify `src/lib/agent/tool-gate.ts:27` to derive registry-id membership per-step
- **File:** `src/lib/agent/tool-gate.ts` (line 27 `const REGISTRY_IDS = new Set(CAPABILITIES.map((c) => c.id));`)
- **Story:** US2 (FR-005)
- **Work:**
  - Move registry-id membership derivation **inside `transformParams`** (per-step), sourced from `listCapabilities()`, so dynamically-registered tools are gated correctly; remove the module-level frozen constant.
- **Acceptance:** Per-step gating uses live capability ids; no frozen import-time snapshot.

### T024 — [P] Unit test: lifecycle cleanup (stop / crash / uninstall)
- **File:** `tests/services/ServiceManager.test.ts` (extend)
- **Story:** US2 (FR-003)
- **Work:**
  - `ServiceManager.stop` → `unregisterServiceTools(serviceId)`; tools removed.
  - Unexpected worker `exit` → `handleCrash` → tools removed; restart re-registers on next `tool_declare`.
  - `service:uninstalled` (via `serviceInstaller`) → tools removed.
- **Acceptance:** All lifecycle paths unregister tools; crash+restart re-registers.

### T025 — Implement lifecycle cleanup (unregisterServiceTools) in ServiceManager
- **File:** `src/core/service/ServiceManager.ts`
- **Story:** US2 (FR-003)
- **Work:**
  - Wire `unregisterServiceTools(serviceId)` into `stop`, `handleCrash` (worker exit), and the `service:uninstalled` orchestration path.
- **Acceptance:** Tools are removed on stop/crash/uninstall; no stale registry entries.

### T026 — Logging at cleanup events
- **File:** `src/core/service/ServiceManager.ts`
- **Story:** US2 (logging requirement)
- **Work:**
  - Emit `tool:unregistered` info logs on each cleanup path and `worker:exit-cleanup` info with `{ serviceId, toolCount }`.
- **Acceptance:** Cleanup events produce structured `services.tool-bridge` records.

> **CHECKPOINT — US1 + US2.** Service tools are gated (allowlist + deferred, per-step) and cleaned up correctly across stop/crash/uninstall.

---

## PHASE 4 — US3 (P3): Isolated + crash-safe invocation

### T030 — [P] Unit test: worker crash mid-tool-call → pending waiter rejects (no BOS crash)
- **File:** `tests/services/workerIpc.test.ts` (extend)
- **Story:** US3 (R10)
- **Work:**
  - Worker exits mid-`tool_call` → `waitForToolResult` rejects with a `worker exited before...` error; BOS host stays healthy; no unhandled rejection.
- **Acceptance:** Pending waiter rejects cleanly on worker exit.

### T031 — [P] Unit test: timeout/cancellation — waitForToolResult rejects on run abort + worker exit
- **File:** `tests/services/workerIpc.test.ts` (extend)
- **Story:** US3 (R10)
- **Work:**
  - Run abort → pending waiter rejects with timeout/abort error; no promise leak.
  - Worker exit before reply → waiter rejects (`worker exited before...`).
- **Acceptance:** Both cancellation paths reject cleanly; no unresolved promises leak.

### T032 — [P] Unit test: two concurrent tool calls don't cross-talk (callId isolation)
- **File:** `tests/services/workerIpc.test.ts` (extend)
- **Story:** US3 (R2)
- **Work:**
  - Two concurrent calls with distinct `callId`s; worker echoes `tool_result` for A and B; each promise resolves with its own result.
- **Acceptance:** No cross-talk between concurrent calls (callId-keyed).

### T033 — Implement timeout/cancellation keyed by callId
- **File:** `src/core/service/workerIpc.ts`
- **Story:** US3 (R10)
- **Work:**
  - Add per-`callId` timeout; reject the pending waiter on run abort or worker exit.
  - Clean up the waiter map on settle.
- **Acceptance:** Timeouts/aborts/worker-exit reject the correct waiter without leaks.

### T034 — Implement tool-error propagation (worker tool_error → AssistantTool error result)
- **File:** `src/core/service/workerIpc.ts`, `src/lib/agent/service-tool-bridge.ts`, `src/lib/assistant/agent-loop.ts`
- **Story:** US3 (FR-004)
- **Work:**
  - Route `tool_error` back as an `AssistantTool` error result (no BOS crash).
  - Propagate the error code/message into the tool result the agent loop returns.
- **Acceptance:** A throwing worker yields an in-band error result; BOS remains healthy; after restart the tool re-registers and a subsequent call succeeds.

### T035 — Logging at timeout/cancellation/error (warn/error)
- **Files:** `src/core/service/workerIpc.ts`, `src/lib/agent/service-tool-bridge.ts`
- **Story:** US3 (logging requirement)
- **Work:**
  - `logger().warn(LOG, "tool_call:timeout" | "tool_call:cancelled", { serviceId, name, callId })`; `logger().error(LOG, "tool_call:error", ..., { serviceId, name, callId, error })`.
- **Acceptance:** Timeout/cancellation emit warn; error results emit error records under `services.tool-bridge`.

> **CHECKPOINT — all stories.** Invocation is crash-safe and isolated; timeouts/cancellations/errors are logged and non-fatal.

---

## PHASE 5 — E2E + Polish

### T040 — E2E self-test: install stub → assistant calls service tool → verify result
- **File:** `e2e/039-service-tool-exposure.spec.ts` (new; imports `test` from `./fixtures`)
- **Story:** US1/US2/US3 (Constitution IV self-test)
- **Work:**
  - Install the stub marketplace item whose service declares a tool (via the `_tool-service-fixtures` fixture).
  - Ask the assistant to call it by name; verify the result appears in the transcript.
  - Wrap in `test.describe.serial` (per plan § Test Strategy 6 parallelism note) to avoid cross-spec interference on the shared service host.
  - Run via `npm run test:e2e`.
- **Acceptance:** E2E passes serially; proves a service tool is callable by the assistant end-to-end.

### T041 — [P] Typecheck + lint gates (mandatory pre-promote)
- **Commands:** `npx tsc --noEmit`; `npm run lint` (on changed files)
- **Story:** All (Constitution VII)
- **Work:**
  - Typecheck passes on the feature branch; lint passes on changed files.
  - Do **not** run `npm run build` while `next dev` is running.
- **Acceptance:** `tsc --noEmit` and lint are green before any promote.

### T042 — Documentation update (docs/dev)
- **Files:** `docs/dev/` (services-as-tools pattern, `deploymentMode` field)
- **Story:** All (Constitution VI)
- **Work:**
  - Document the services-as-tools pattern: manifest `deploymentMode: "tools"`, `tool_declare`/`tool_call`/`tool_result`/`tool_error` IPC contract, bridge registration, gating behavior.
- **Acceptance:** Docs reflect the new capability; spec/docs/tests updated in the same change.

### T043 — Final: full test suite green + FR coverage
- **Commands:** unit + integration + e2e suites
- **Story:** All
- **Work:**
  - Full test suite green (unit + integration + e2e).
  - Verify all FR-001..010 (from spec.md) are covered by tests.
- **Acceptance:** All suites pass; FR-001..010 explicitly mapped to passing tests; self-test complete before promote (Constitution IV).

---

## Dependencies & Execution Order

```
Phase 1 (foundation)
  T001 ─┬─▶ T002 ─┐
  T004  │        ├─▶ T003 ─▶ T005 ─▶ T006
        └────────┘                        └─▶ CHECKPOINT
Phase 2 (US1)
  T013 [P] ─┐
  T010 [P]  ├─▶ T014 ─▶ T015 ─▶ T016 ─▶ T017 ─▶ T018
  T011 [P]  │
  T012      ┘                          └─▶ CHECKPOINT
Phase 3 (US2)
  T022 ─▶ T023        (gating, sequential)
  T020 [P] ─┐
  T021 [P]  ├─▶ T025 ─▶ T026
  T024 [P]  ┘              └─▶ CHECKPOINT
Phase 4 (US3)
  T033 ─▶ T034 ─▶ T035
  T030 [P] ─┐
  T031 [P]  ├───────────────▶ CHECKPOINT
  T032 [P]  ┘
Phase 5 (E2E + polish)
  T040 ─▶ T041 ─▶ T042 ─▶ T043
```

- **Phase dependencies:** Phase 2 cannot start until Phase 1's checkpoint is green; Phase 3 depends on Phase 2 (bridge + registration exist); Phase 4 depends on Phase 3 (lifecycle wiring in place); Phase 5 depends on all prior phases.
- **Parallel opportunities:**
  - Phase 2: T010/T011/T013 are [P] — independent unit tests + bridge implementation can run together once T006 fixtures exist; T012 (schema rejection) depends on T013's validation path.
  - Phase 3: T020/T021/T024 are [P] — gating unit tests and lifecycle unit tests are independent of each other; T022/T023 (gating impl) can run alongside T024.
  - Phase 4: T030/T031/T032 are [P] — worker IPC crash/timeout/concurrency tests are independent; T033/T034/T035 (implementation) follow.
  - Phase 5: T041 is [P] (typecheck/lint can run anytime after Phase 2).

## Implementation Strategy (MVP-first)

1. **MVP-first (US1 → validate → US2 → US3 → e2e):** Deliver the minimal end-to-end US1 path first (declare → register → agent invokes → result) with logging, then validate it via integration tests before expanding to gating and lifecycle (US2), then crash-safety (US3), then the e2e self-test.
2. **Data-model/contracts first:** T001–T003 lock the `ServiceTool` model, IPC message types, and manifest schema before any behavior — so worker fixtures (T006) and the bridge (T005/T013) compile against stable contracts.
3. **Test alongside implementation:** Each behavior task pairs with its unit test in the same phase; integration (T018) proves the MVP; e2e (T040) proves the user-visible story.
4. **Logging is non-optional:** Every new module carries `const LOG = "services.tool-bridge"` and structured `logger()` records at the key events (declare/register/unregister/dispatch/resolve/schema-reject/timeout/worker-exit), asserted by tests (T010) — distinct from raw worker stdout capture via `appendServiceLog`.
5. **Gating is a correctness fix (US2):** The frozen `REGISTRY_IDS` in `tool-gate.ts:27` must become per-step `listCapabilities()`-derived so dynamically-registered service tools are actually gated; this is a required fix, not a nicety.
6. **Pre-promote gates:** Typecheck + lint (T041), docs sync (T042), and the full green suite + FR coverage (T043) are mandatory before promotion, per Constitution IV/VI/VII.
