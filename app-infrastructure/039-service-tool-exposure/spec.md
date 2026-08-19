# Feature Specification: Service Tool Exposure

**Feature Branch**: `bos/039-service-tool-exposure`

**Created**: 2026-08-10

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "I want a marketplace item to be able to expose tools to BOS without going through an MCP server. Which execution backend (worker threads / child processes / containers) enables that? Could we modify BOS so this becomes possible, e.g. with different modes of deployment for a marketplace item?"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Marketplace Service Exposes a Native Tool (Priority: P1)

A marketplace item that ships a background service can declare one or more tools (a name, description, and input JSON-schema) and have those tools appear to the BOS assistant as native, callable tools — with no MCP server in the loop.

**Why this priority**: This is the core capability — without it the whole feature is moot. It delivers the primary value: letting a marketplace service contribute agent-callable functions without MCP.

**Independent Test**: Install a marketplace item with a service that declares a tool. Ask the assistant to use the tool by name; verify the assistant can invoke it and receive a result. This is fully testable in isolation and delivers the headline value.

**Acceptance Scenarios**:

1. **Given** a marketplace item with a background service that declares a tool (name `my_tool`, description, JSON-schema input), **When** the item is installed and its service starts, **Then** `my_tool` appears in BOS's tool registry and is visible to the assistant.
2. **Given** `my_tool` is registered, **When** the assistant calls `my_tool` with valid input, **Then** the service executes the tool and returns a result to the assistant.
3. **Given** the service declares a tool whose input schema requires a field, **When** the assistant calls it without that field, **Then** BOS rejects the call with a schema-validation error and does not invoke the service.

---

### User Story 2 - Service Tools Respect Existing Tool Gating (Priority: P2)

Service-declared tools are governed by the same tool-gating and authorization rules as built-in tools, so a service cannot bypass the user's permission settings.

**Why this priority**: Security/trust is the second-most-critical concern. A marketplace item gaining the ability to run code on the user's behalf must respect the user's existing tool-permission model.

**Independent Test**: With a restrictive tool-gate config (allowlist or deferred approval), verify a service tool is not auto-executed — it is gated the same way a built-in tool would be. Testable independently and delivers the trust guarantee.

**Acceptance Scenarios**:

1. **Given** a tool-gate config that defers or blocks a tool by default, **When** a service declares a tool matching that scope, **Then** BOS does not execute it without user approval.
2. **Given** a service-declared tool, **When** the service is stopped or uninstalled, **Then** the tool disappears from the registry and the assistant can no longer call it.

---

### User Story 3 - Service Tool Invocation is Isolated and Crash-Safe (Priority: P3)

A tool call that fails inside the service does not crash BOS, and the failure is reported to the assistant as a tool error.

**Why this priority**: Resilience is important but secondary to core function and trust. A runaway or crashing service tool must not take down the OS.

**Independent Test**: Declare a tool that throws or returns a malformed result; verify BOS reports an error to the assistant and remains healthy. Testable independently and delivers the safety guarantee.

**Acceptance Scenarios**:

1. **Given** a service tool whose execution throws, **When** the assistant invokes it, **Then** BOS returns a tool-error result to the assistant and BOS itself does not crash.
2. **Given** a service that crashes while handling a tool call, **When** the crash-recovery mechanism restarts it, **Then** BOS reports the in-flight tool call as failed and the tool remains available for subsequent calls.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST provide a mechanism for a marketplace item's background service to declare one or more tools, each with a name, description, and input JSON-schema, at service startup.
- **FR-002**: BOS MUST surface service-declared tools into the existing `AssistantTool` registry so they are visible to the assistant alongside built-in tools.
- **FR-003**: BOS MUST be able to invoke a service-declared tool by sending a tool-invocation request to the service and receiving a result.
- **FR-004**: BOS MUST validate a tool invocation against the declared JSON-schema before dispatching it to the service, and reject invalid calls without invoking the service.
- **FR-005**: BOS MUST gate service-declared tools through the existing tool-permission/gating model (allowlist, deferred approval) exactly like built-in tools.
- **FR-006**: BOS MUST remove a service's tools from the registry when the service is stopped or the item is uninstalled.
- **FR-007**: BOS MUST report a tool-execution error from a service back to the assistant as a tool error, without crashing BOS.
- **FR-008**: BOS MUST support the tool-invocation contract over the existing service message-passing protocol (new message types) AND/OR a loopback HTTP route, per the execution backend the service uses.
- **FR-009**: BOS MUST authenticate/trust service tool-invocation requests through the existing loopback-scoped auth mechanism, so only BOS itself can invoke a service's tool.
- **FR-010**: BOS MUST support a deployment-mode field on the service manifest that opts a service into tool exposure (default remains the current no-tool behavior for backward compatibility).

### Key Entities *(include if feature involves data)*

- **ServiceTool**: A tool declared by a marketplace service — name, description, input JSON-schema, and a reference to the owning service. Maps to BOS's `ToolDeclaration`/`AssistantTool`.
- **ToolInvocation**: A request/response pair for calling a service tool — tool name, input arguments, output/error result.
- **ServiceManifest / Mode**: The deployment-mode field on a service's manifest that enables tool exposure.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A marketplace item with a service declaring a tool can be installed and have that tool visible to the assistant immediately upon service start — no manual registration step.
- **SC-002**: 100% of valid tool invocations to a healthy service return a result without error.
- **SC-003**: A crashing service tool does not cause BOS to crash or restart (verified across repeated failures).
- **SC-004**: Service-declared tools are gated by the same rules as built-in tools — no tool can be auto-executed when the user's gate config requires approval.
- **SC-005**: Stopping or uninstalling a service removes its tools from the registry immediately, such that no stale tool call to a stopped service is possible.

## Assumptions

- BOS continues to execute marketplace services as worker threads for v1 of this feature; child-process and container backends are out of scope for this spec but the tool-invocation contract is designed to be backend-agnostic.
- The existing `AssistantTool` registry (`src/lib/assistant/tools.ts`) and capability registry (`src/lib/agent/capabilities-registry.ts`) are the integration points for surfacing service tools.
- The existing loopback-scoped auth (`src/lib/secrets/auth-scope.ts`) is the trust boundary for tool invocation.
- MCP remains fully supported and is unaffected; this feature adds a native non-MCP path alongside it.
- Tool declarations are static (declared at service startup) rather than dynamically discoverable at call time, for v1.
- No UI for managing service tools is in scope for v1 — tool management happens via the service manifest and the assistant's existing tool gating.
