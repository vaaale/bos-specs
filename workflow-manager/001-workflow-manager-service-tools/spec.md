# Feature Specification: Workflow Manager Service Tools

**Feature Branch**: `bos/001-workflow-manager-service-tools` (per-project numbering under the Workflow Manager project)

**Created**: 2026-08-18

**Status**: Draft

**App Target**: marketplace-item

**Input**: User description: "The Workflow Manager App is not working according to the new architecture (Spec 039). I want you to rewrite it so that it's compliant and working as intended, including exposing the proper tools."

## Context

The Workflow Manager is an installed marketplace item (app id `workflows`) composed of an app facet (a UI that lists and drives workflows) and a background-service facet (a lifecycle-only worker). BOS spec 039 (`user-specs/app-infrastructure/039-service-tool-exposure`, bos-core, implemented) established the architecture by which a marketplace item's background service declares native, agent-callable tools: a service sets `deploymentMode: "tools"` on its manifest and emits `tool_declare` at startup; BOS registers those tools into the `AssistantTool` registry (via `serviceToolBridge`), validates invocations against their JSON-schemas, dispatches `tool_call` over worker IPC, and gates them exactly like built-in tools.

The current Workflow Manager item does not follow this architecture:

- The workflow tools live as BOS-source server tools (`src/lib/assistant/tools/server/workflows.ts` — `workflowTools()`) rather than being declared by the item's own service via `tool_declare`.
- The item's service is lifecycle-only (its `isWorkflowsServiceRunning()` state merely gates `workflow_run`); it does not opt into `deploymentMode: "tools"` and declares no tools.
- The app lists workflows by reading `/Workflows`, and the recurring marketplace VFS-path-mapping bug applies: content can land under `dataDir()/system/...` instead of the real VFS root `dataDir()/vfs/`, making workflows invisible to the user even though internal APIs agree — the reported "lists no workflows even though one exists" symptom.

This spec rewrites the Workflow Manager item so the workflow tools are exposed as service-declared native tools (039-compliant), the service persists workflows through the real VFS (loopback `/api/fs` bridge, the sanctioned path for unbundled worker-thread services), and the app lists workflows from the same real-VFS location.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Workflow Tools Appear as Service-Declared Native Tools (Priority: P1)

The Workflow Manager's service declares the workflow tools (create, modify, run, status, cancel, export, validate) at startup via `tool_declare`, so they appear to the assistant as native, gated tools owned by the item — no BOS-source server tool registration in the loop.

**Why this priority**: This is the core of "compliant with Spec 039." Without it the item still relies on BOS-source server tools and is architecturally non-compliant. It delivers the headline value: the workflow surface becomes a marketplace-contributed native tool set.

**Independent Test**: Install the rewritten item; confirm the service starts in `deploymentMode: "tools"` mode; ask the assistant to call a workflow tool (e.g. `workflow_list`) by name; verify the tool executes through the service and returns a result. Testable in isolation.

**Acceptance Scenarios**:

1. **Given** the rewritten Workflow Manager item is installed and its service starts, **When** the service emits `tool_declare` for each workflow tool, **Then** the tools appear in the assistant's tool registry with a name, description, and JSON-schema, and are discoverable by the assistant.
2. **Given** a workflow tool is registered, **When** the assistant calls it with valid input, **Then** the service executes the workflow operation and returns a result to the assistant.
3. **Given** a workflow tool's schema requires a field, **When** the assistant calls it without that field, **Then** BOS rejects the call with a schema-validation error and does not dispatch it.

---

### User Story 2 - Service Tools Respect Tool Gating and Lifecycle (Priority: P2)

The service-declared workflow tools are governed by the same allowlist/deferred gating as built-in tools, and disappear from the registry when the service is stopped or the item is uninstalled.

**Why this priority**: Security/trust. A marketplace item gaining the ability to run workflows on the user's behalf must respect the user's tool-permission model, exactly per 039 US2.

**Independent Test**: With a restrictive tool-gate config, verify a workflow tool is not auto-executed; stop the service and verify the tools disappear. Testable in isolation.

**Acceptance Scenarios**:

1. **Given** a tool-gate config that defers or blocks a workflow tool, **When** the service declares it, **Then** BOS does not execute it without approval.
2. **Given** a service-declared workflow tool, **When** the service is stopped or the item uninstalled, **Then** the tool disappears from the registry and the assistant can no longer call it.

---

### User Story 3 - Workflows Are Listed and Managed from the Real VFS (Priority: P2)

The Workflow Manager app lists workflows that live in the user's real VFS `/Workflows` folder (the same location the workflow engine reads and writes), so workflows created by the engine or placed by the user are visible in the app, and workflows created in the app are visible to the engine and Files app.

**Why this priority**: This fixes the reported "lists no workflows" bug and is the correctness guarantee that content persists where the user can see it. It is required for the item to be "working as intended."

**Independent Test**: Create a workflow through the app; confirm the workflow JSON appears under the real VFS `/Workflows/` (visible in Files) and is listed again by the app on reload; confirm the workflow engine can read it. Testable in isolation.

**Acceptance Scenarios**:

1. **Given** a workflow JSON exists at the real VFS `/Workflows/<id>-workflow.json`, **When** the app lists workflows, **Then** the workflow is shown.
2. **Given** the user creates a workflow in the app, **When** the workflow is persisted, **Then** it is written to the real VFS `/Workflows/` root and appears in Files and in the engine's `listWorkflows()`.
3. **Given** a workflow was previously stranded in a legacy non-VFS location (e.g. under `dataDir()/system/`), **When** the rewritten service initializes, **Then** it recovers/migrates that workflow into the real VFS (additive-only, archiving rather than deleting the legacy copy).

---

### User Story 4 - Workflow Execution Runs Through the Service (Priority: P3)

Running a workflow is gated by the service being in a healthy running state, and the run streams step events back so the app and assistant see progress and results.

**Why this priority**: Ensures the service remains the execution authority for workflows, consistent with the existing `workflow_run` gating (`isWorkflowsServiceRunning()`), and keeps the item coherent end-to-end.

**Independent Test**: Start the service, run a workflow via the app or a tool, observe step events stream; stop the service and verify a run attempt is refused with a clear message. Testable in isolation.

**Acceptance Scenarios**:

1. **Given** the service is running, **When** a user runs a workflow, **Then** step events stream and the final state (completed/failed/cancelled) is reported.
2. **Given** the service is stopped, **When** a user attempts to run a workflow, **Then** BOS returns a clear error that the Workflows service is not running.
3. **Given** a running workflow is cancelled, **When** the user cancels it, **Then** in-progress steps are marked cancelled and the scheduler halts.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The Workflow Manager service MUST set `deploymentMode: "tools"` on its manifest and emit `tool_declare` for each workflow tool at startup.
- **FR-002**: The service MUST declare at minimum the following workflow tools, each with a name, description, and input JSON-schema: list/create/read/modify/run/status/cancel/delete/export/validate workflows (the 039-compliant native-tool surface replacing the current BOS-source `workflowTools()` set).
- **FR-003**: Each declared tool MUST validate its arguments against its input JSON-schema before executing (per 039 FR-004); invalid calls MUST be rejected without dispatch.
- **FR-004**: The service MUST persist workflows to the real VFS `/Workflows/` root via the loopback `/api/fs` bridge (the sanctioned unbundled-worker-thread path), so workflows are visible in Files and to the workflow engine.
- **FR-005**: On initialization, the service MUST detect workflows stranded in legacy non-VFS locations and migrate them into the real VFS `/Workflows/` additively (copying/archiving, never deleting the legacy copy).
- **FR-006**: The app MUST list workflows by reading the real VFS `/Workflows/` root through the service, showing workflows created by the engine, by other tools, or placed by the user.
- **FR-007**: Workflow execution MUST be gated on the service being in a running state; a run attempt while the service is stopped MUST return a clear error.
- **FR-008**: A workflow run MUST stream step events (start/complete/fail/retry/cancel) back to the caller so the app and assistant observe progress and final state.
- **FR-009**: Cancelling a running workflow MUST mark in-progress steps cancelled and halt the scheduler.
- **FR-010**: Service-declared workflow tools MUST be gated by the existing tool-gating model (allowlist, deferred approval) exactly like built-in tools (per 039 FR-005).
- **FR-011**: Stopping or uninstalling the service MUST remove its declared tools from the registry, so no stale tool call to a stopped service is possible (per 039 FR-006).

### Non-Functional Requirements

- **NFR-001**: A workflow tool invocation MUST return a result within 30 seconds by default, with timeout/cancellation handled per 039 (R10) without leaking pending waits or crashing BOS.
- **NFR-002**: Tool-execution errors inside the service MUST be reported to the assistant as tool errors without crashing BOS (per 039 FR-007).
- **NFR-003**: The service MUST bind a configurable port defaulting to `0` (OS-assigned, non-colliding) per marketplace service conventions.
- **NFR-004**: All storage reads/writes MUST go through the real-VFS loopback bridge; the service MUST NOT write to a host path under `dataDir()/system/` derived from its own config dir.
- **NFR-005**: No error path MAY be swallowed without logging; key events (tool_declare received, tool registered, tool_call dispatched/resolved, schema-rejected, timeout, worker-exit cleanup) MUST emit structured logs via the service's log channel.

## Key Entities *(include if feature involves data)*

- **Workflow**: A user-authored multi-step graph — id, name, version, agents, steps (ag-ui/delegate/tool), dependencies, config (max concurrency, default retry/timeout). Persisted as JSON at the real VFS `/Workflows/<id>-workflow.json`.
- **WorkflowTool**: A native tool the service declares via `tool_declare` — name, description, input JSON-schema, mapped into the `AssistantTool` registry.
- **ExecutionEvent / StepRuntimeState**: Streamed events and per-step runtime status during a workflow run, used for progress reporting and cancellation.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: After installing the rewritten item, all workflow tools are visible to the assistant immediately upon service start — no manual registration step.
- **SC-002**: 100% of valid workflow-tool invocations against a healthy service return a result without error.
- **SC-003**: A workflow created in the app, via the engine, or placed by the user under `/Workflows/` is listed by the app in 100% of cases (no invisible-content mismatch between app, engine, and Files).
- **SC-004**: Service-declared workflow tools are gated by the same rules as built-in tools — no tool auto-executes when the user's gate config requires approval.
- **SC-005**: Stopping or uninstalling the service removes its workflow tools from the registry immediately, such that no stale tool call to a stopped service is possible.
- **SC-006**: A workflow run streams step events and reports a final state (completed/failed/cancelled) for every executed workflow.

## Assumptions

- The workflow execution engine stays in BOS source (`src/lib/workflows/runner.ts`) — workflow runs require the full assistant stack on the main thread. The marketplace service orchestrates and gates execution and exposes the tool surface; it does not reimplement the execution engine.
- The service may reuse the existing `workflowTools()` tool definitions (names/descriptions/schemas) as the source of truth for what it declares, so the native tool surface is behaviorally identical to the current server tools.
- BOS source already implements 039 (service-tool-bridge, worker IPC, `deploymentMode` validation, gating fixes); this spec only changes the marketplace item, not BOS source.
- Workflows persist as JSON files under `/Workflows/` (VFS), matching the existing engine's store (`src/lib/workflows/store.ts`).
- Migration is additive-only and idempotent: stranded workflows are copied into the real VFS (skipping any that already exist) and the legacy location is archived (renamed), never deleted.
- No UI redesign for the Workflow Manager app is in scope beyond making it list/read workflows from the real VFS and drive the service-declared tools; the existing app layout is retained.
