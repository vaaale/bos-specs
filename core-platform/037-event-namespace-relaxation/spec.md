# Feature Specification: Event Namespace Relaxation

**Feature Branch**: `bos/002-workflow-event-integration` (shared with the workflow item scope-add)

**Created**: 2026-09-02

**Status**: Draft

**App Target**: bos-core

**Input**: User decision: "There should be no restrictions to who can receive which events. Any component should be able to listen to any event."

## Context

The 034 event-notification system (implemented and converged) enforces a namespace-ownership check on handler registration (FR-023): a component may only register a headless or UI handler for an event type if that type falls under its owned root (`com.bos.<ownerId>.*`) or a statically-granted `eventNamespaces` list. This check lives in `src/lib/events/api.ts`'s `register()` function, at the `ownsNamespace()` call.

The 034 design doc explicitly characterizes this as "a same-container integrity check, not a network security boundary" — it exists to catch accidental collisions between components, not to defend against adversaries. In BOS's single-container trust model, this is over-restrictive for components that need to react to events from other components (orchestrators, aggregators, monitoring services).

This spec removes the namespace-ownership check on handler registration. Any component can subscribe to any event type. The `ack` ownership check (a service can only ack its own handlers) is unchanged — it's a different concern (integrity of the ack path, not subscription access).

**Why this matters now**: The Workflow Manager (marketplace item `workflows`, scope-add 002) is an event-driven orchestrator — its purpose is to react to events from other components (emails, tasks, health warnings) and start workflows. Without this change, the workflow service would need to declare a broad `eventNamespaces` grant in its `service.json` for every event type it might react to — a workaround that violates BOS's architectural principle against side-steps.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Any Component Can Subscribe to Any Event Type (Priority: P1)

A worker-thread service (or any BOS component) can register a headless handler for any event type, regardless of whether that type falls under its owned namespace or a granted `eventNamespaces` list. This makes event subscription a first-class, unrestricted capability — the pub/sub model is open by default.

**Why this priority**: This is the core of the platform change. Without it, the Workflow Manager (and any future orchestrator/aggregator) cannot function as designed.

**Independent Test**: A service with id `test-svc` registers a headless handler for event type `com.bos.other-svc.thing.happened` (outside its owned namespace, no grant). Confirm the registration succeeds (no `namespace-not-owned` error) and the handler is invoked when an event of that type is emitted. Testable in isolation.

**Acceptance Scenarios**:

1. **Given** a service with id `workflows`, **When** it declares a headless handler for event type `com.bos.gsuite.email.received` (via `handler_declare` over worker IPC), **Then** the registration succeeds — no `namespace-not-owned` error — and the handler is active in the registry.
2. **Given** a registered handler for a non-owned event type, **When** an event of that type is emitted, **Then** the handler is dispatched and invoked normally (FIFO, per 034 dispatch semantics).
3. **Given** a handler registered by a service, **When** the service attempts to ack an event it did NOT register the handler for, **Then** the ack is rejected with `ack-forbidden` (the ack ownership check is unchanged — a service can only ack its own handlers).

---

### User Story 2 - UI Handlers Are Also Unrestricted (Priority: P2)

The namespace-ownership removal applies to UI handlers as well: any app can declare a UI handler for any event type in its manifest. The "Open with" dialog and default-handler preference mechanism continue to work without namespace constraints.

**Why this priority**: Consistency — the restriction is on the subscription axis, not on the handler mode. If headless handlers are unrestricted, UI handlers should be too.

**Independent Test**: An app with manifest id `my-app` declares a UI handler for event type `com.bos.other-app.thing` in its `AppManifest.eventHandlers`. Confirm the handler is registered at boot and the app appears in the "Open with" dialog for that event type. Testable in isolation.

**Acceptance Scenarios**:

1. **Given** an app's manifest declares a UI handler for an event type outside its owned namespace, **When** BOS boots and surfaces UI handlers (`register-ui-handlers.ts`), **Then** the handler is registered without error.
2. **Given** a UI handler for a non-owned event type is registered, **When** the user clicks an event of that type in the Event Viewer, **Then** the app appears in the "Open with" dialog (or opens directly if it's the only/default handler).

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The `register()` function in `src/lib/events/api.ts` MUST NOT reject a handler registration based on namespace ownership. The `ownsNamespace()` check and its `namespace-not-owned` error MUST be removed from the registration path.
- **FR-002**: The `ack` ownership check (the `callerId` must match the handler's `ownerId`, enforced in `kernel.ts`'s `ack()`) MUST be unchanged — a service can still only ack handlers it registered.
- **FR-003**: The `eventNamespaces` field in manifests (`AppManifest.eventNamespaces`) and `service.json` MUST be accepted but ignored for registration purposes. It MAY be retained for documentation/discovery purposes but MUST NOT gate registration.
- **FR-004**: The `namespace-not-owned` error code in `src/lib/events/types.ts` MAY be retained in the error-code table for backward compatibility but MUST NOT be thrown by any code path.
- **FR-005**: The `resolveGrantedNamespaces()` helper in `api.ts` MAY be removed (it exists solely to feed the `ownsNamespace` check) or retained as dead code — either is acceptable.
- **FR-006**: The `ownsNamespace()` function in `types.ts` MAY be removed or retained — either is acceptable, provided no code path calls it to reject registration.

### Non-Functional Requirements

- **NFR-001**: The removal MUST NOT change the registration response shape, the handler registry data model, or the dispatch/ack semantics. Only the rejection path is removed.
- **NFR-002**: Existing registered handlers (from before the change) MUST continue to work without re-registration — no migration needed.

## Key Entities *(include if feature involves data)*

No new entities. The `HandlerRegistration` record shape is unchanged. The `eventNamespaces` field in manifests becomes advisory-only.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A service with id `workflows` can register a headless handler for `com.bos.gsuite.email.received` without declaring `eventNamespaces` in its `service.json` — registration succeeds.
- **SC-002**: An app can declare a UI handler for any event type in its manifest — registration at boot succeeds.
- **SC-003**: The `ack` ownership check still rejects a cross-handler ack attempt with `ack-forbidden`.
- **SC-004**: No existing 034 e2e test fails after the change (the namespace-rejection test case is removed or updated to expect success).

## Assumptions

- The single-container trust model applies: all components in a BOS container are trusted to subscribe to any event. The namespace check was a collision-guard, not a security boundary, and its removal does not weaken the system's security posture.
- The `eventNamespaces` field in manifests is retained for documentation/discovery (it tells a reader what event types a component is "interested in") but is no longer a gate.
- The 034 spec's FR-023 is superseded by this spec. The 034 design doc's characterization of the check ("same-container integrity check, not a network security boundary") is the basis for this removal.
- This change is backward-compatible: existing handlers that were registered under their own namespace continue to work; the only behavioral change is that handlers that would previously have been rejected are now accepted.
