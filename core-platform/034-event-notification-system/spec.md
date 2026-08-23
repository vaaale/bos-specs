# Feature Specification: Event & Notification System

**Feature Branch**: `034-event-notification-system`

**Created**: 2026-08-23

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Any application in BOS should be able to produce events. The producer may also define the event type and payload. If a user clicks on the notifications in the top toolbar, an event viewer app must be shown. The notification app must be part of BOS core, but extensible by any app in BOS including marketplace apps. An app can register an event handler UI part that handles certain event types. When the user clicks on an event in the event app, the appropriate handler will launch if such a handler is registered. If multiple apps have registered to handle the same event type, the user must be prompted to select which handler to launch, and have the option for 'Always use this app for this event type'. The event handler app must have a configuration page where these handlers can be configured / re-configured / cleared. If the user clicks on an event that has no handler associated with it, a generic event viewer app / dialog must be shown. In addition to UI handlers triggered by user clicks, an app must also be able to register a headless handler that processes the event automatically upon emission (e.g., the Workflow Manager listening to event types and triggering workflows). The system operates as a pub/sub system: the core event service invokes registered headless handlers on emission, handlers acknowledge processing, and an event is marked processed when all active handlers have acknowledged. UI handlers are a display layer only and do not block processing. The public event API is the same API that agent tools use."

## Architecture Overview *(informative — not a requirement)*

The system is a **pub/sub event broker** with two distinct handler modes and a user-facing read/unread dimension:

- **Core Event Service** (BOS core, starts at boot, runs for lifetime of session):
  - Accepts events from emitters, stores them persistently (indefinite retention)
  - Invokes all active **headless handlers** automatically on emission (fan-out, at-least-once)
  - Maintains a per-handler FIFO queue: each handler processes events sequentially (one at a time, in emission order)
  - Tracks per-event, per-handler acknowledgment status
  - Retries failed handler invocations (3 attempts, exponential backoff: 1s, 5s, 30s)
  - After max retries, a handler is "permanently failed" for that event; the event can complete
  - An event transitions **pending** → **processed** when ALL active headless handlers have either acknowledged or been permanently failed
  - Re-dispatches un-acked events on BOS restart and on late handler registration (at-least-once guarantee)
  - Exposes the **unread** count to the toolbar bell

- **Emitters** (services, apps, assistant):
  - Publish events (type + payload, max 1MB) into the queue via the public event API
  - Background services are the primary emitters
  - For data exceeding 1MB, the payload includes a VFS path reference instead of inline data

- **Headless Handlers** (registered by apps or services):
  - Invoked by the core event service, one event at a time, in emission order (FIFO per handler)
  - Process the event, then acknowledge via the public API with an optional result payload
  - MUST be idempotent (at-least-once delivery means they may receive the same event more than once)
  - A disabled handler is not "active" and does not block event completion
  - A handler whose service is not running is not "active" and does not block completion

- **UI Handlers** (registered by apps):
  - NOT invoked by the core service; do NOT participate in the processing model
  - Triggered only when the user clicks an event in the Event Viewer
  - Display/interaction layer only — they do NOT block or affect pending→processed
  - Can inspect the event including all handler results and processing history
  - Do NOT acknowledge events (they are not subscribers)

- **Event Viewer** (built-in app, opens on bell click):
  - Default view: **unread** events (both pending and processed) in reverse-chronological order
  - "Show historical" toggle: **read** events
  - Each event shows a processing status badge: "processing…" (pending) or a checkmark (processed)
  - Clicking an event marks it as **read** and launches the appropriate UI handler (or generic view)
  - Configuration page for managing handlers and preferences

- **Two orthogonal dimensions on each event:**
  - **Processing status**: pending → processed (driven by headless handler acks)
  - **Read status**: unread → read (driven by user viewing in the Event Viewer)
  - The bell shows the **unread** count, not the pending count

- **The public event API is the same API that agent tools use** — one contract for emit, ack, query, register, unregister, mark-read.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View and Triage Notifications (Priority: P1)

The top toolbar bell shows the count of **unread** events. The user clicks the bell and the Event Viewer opens, showing unread events in reverse-chronological order. Each event displays its type, source, timestamp, a summary, and a processing status badge ("processing…" if still pending, checkmark if processed). The user can click an event to view it (marking it read and launching the appropriate UI handler). A "Show historical events" toggle reveals previously read events. A "Mark all as read" action is available.

**Why this priority**: The foundational user journey. Gives users access to the event stream and a way to mark things as seen.

**Independent Test**: Emit a test event → bell increments → open viewer → event appears as unread → click it → marked read → bell decrements → event moves to historical.

**Acceptance Scenarios**:

1. **Given** no events exist, **When** a service emits a new event, **Then** the toolbar bell count increments by one and the event appears in the Event Viewer (unread list) when opened.
2. **Given** an unread event exists, **When** the user clicks it in the Event Viewer, **Then** the event is marked as read, the bell count decrements, and the appropriate UI handler (or generic view) launches.
3. **Given** multiple unread events exist, **When** the Event Viewer is opened, **Then** they are displayed in reverse-chronological order, each showing type, source, timestamp, summary, and processing status badge.
4. **Given** the user clicks "Mark all as read", **When** the action completes, **Then** all unread events are marked read, the bell shows 0, and all events are available in the historical view.
5. **Given** processed events exist that have been read, **When** the user toggles "Show historical events", **Then** read events are listed with their processing history.
6. **Given** an event is still pending (not all handlers have acked), **When** the user views it, **Then** it shows a "processing…" badge; when the final handler acks, the badge updates to a checkmark.
7. **Given** an event was processed while the user had the viewer open, **When** the event's status changes, **Then** the badge updates in real-time without requiring a refresh.

---

### User Story 2 - Emit Events from Any BOS Component (Priority: P1)

Any BOS component — background services (primary emitters), built-in apps, marketplace apps, or the assistant — can emit events through the public event API. The emitter specifies the event type (namespaced) and a payload (structured JSON, max 1MB). The event is durably recorded, assigned a sequence number, and immediately dispatched to matching headless handlers.

**Why this priority**: The producer side. Without emission, nothing to process.

**Independent Test**: Call the emission API from a service with a custom type and payload; verify the event is stored, dispatched, and visible in the viewer.

**Acceptance Scenarios**:

1. **Given** a background service is running, **When** it emits an event with type "com.bos.gsuite.email.received" and a payload, **Then** the event is durably recorded, a sequence number is assigned, and all matching active headless handlers are invoked.
2. **Given** a marketplace app emits an event, **When** the event is displayed in the viewer, **Then** it shows the marketplace app's name and icon as the source.
3. **Given** the assistant (agent) emits an event via the same API contract, **When** the event is stored, **Then** it shows the assistant as the source.
4. **Given** an emitter attempts to emit an event with a payload exceeding 1MB, **When** the API is called, **Then** the emission is rejected with a clear error message; the emitter should use a VFS path reference for large data.

---

### User Story 3 - Automatic Headless Event Processing (Priority: P2)

A service or app registers one or more **headless handlers** for specific event types. The core event service invokes these handlers automatically when a matching event is emitted. Each handler processes events sequentially (one at a time, in emission order) and acknowledges via the public API, optionally including a result payload. If a handler fails, the system retries up to 3 times with exponential backoff before marking it as permanently failed for that event.

**Why this priority**: The programmatic processing layer. Enables automation and drives the pending→processed transition.

**Independent Test**: Register a headless handler, emit an event, verify it's invoked, verify ack stores the result, verify event transitions to processed.

**Acceptance Scenarios**:

1. **Given** a service has a headless handler for "com.example.workflow.trigger", **When** an event of that type is emitted, **Then** the handler is invoked with the full event record (type, payload, source, event ID, sequence).
2. **Given** three services have headless handlers for the same event type, **When** an event is emitted, **Then** all three handlers receive the event (fan-out), each processing it independently and sequentially.
3. **Given** a handler processes an event and acks with result `{"workflowId": "wf-123", "status": "triggered"}`, **When** the user views the event in the viewer, **Then** the result payload is visible in the processing history.
4. **Given** a handler throws an error on first attempt, **When** the retry (after 1s) also fails, **Then** the system retries a third time (after 5s); if it fails again, the handler is marked "permanently failed" for that event after a final retry (30s).
5. **Given** a handler is permanently failed for an event, **When** all other handlers have acked, **Then** the event transitions to processed; the failure is visible in the processing history.
6. **Given** a handler is idempotent, **When** the same event is delivered twice (due to at-least-once), **Then** the second invocation is safely ignored (no duplicate side effects).
7. **Given** a handler's service is not running when an event is emitted, **When** the service starts and registers its handler, **Then** the un-acked event is delivered to it (late registration catch-up).
8. **Given** BOS restarts, **When** the event service comes back online, **Then** any events that were dispatched but not yet acked are re-dispatched to their handlers.

---

### User Story 4 - Register UI Handlers (Priority: P2)

An app registers one or more **UI handlers** for specific event types. A UI handler is a UI component launched when the user clicks an event of that type in the Event Viewer. UI handlers are a display/interaction layer only — they do NOT participate in event processing, do NOT block completion, and do NOT acknowledge events.

**Why this priority**: Gives apps a way to present event-specific UI when the user interacts with an event.

**Independent Test**: Register a UI handler, emit an event, open viewer, click the event, verify the UI handler launches with the full event record.

**Acceptance Scenarios**:

1. **Given** an app has a UI handler for "com.bos.gsuite.email.received", **When** the user clicks an event of that type in the Event Viewer, **Then** the UI handler component is launched with the full event record (including processing history and handler results).
2. **Given** a UI handler is registered, **When** the event is still pending (processing), **Then** the user can still click it; the UI handler launches and shows the current processing state.
3. **Given** an app is uninstalled, **When** the user clicks an event whose only UI handler belonged to that app, **Then** the generic event detail view is shown instead.

---

### User Story 5 - Click an Event to Launch Its UI Handler (Priority: P2)

The user clicks an event in the Event Viewer. The event is marked as read. If a UI handler is registered for that type, it launches. If multiple UI handlers exist, the user is prompted to select. If none exist, the generic event detail view is shown.

**Why this priority**: The primary user interaction that makes the viewer actionable.

**Independent Test**: Register a UI handler, click an event, verify the handler launches and the event is marked read.

**Acceptance Scenarios**:

1. **Given** an event has exactly one UI handler (or a user-set default), **When** the user clicks the event, **Then** it is marked read and the UI handler launches with the full event record.
2. **Given** an event has no UI handler for its type, **When** the user clicks it, **Then** it is marked read and the generic event detail view is shown (full payload + processing history).
3. **Given** the user has set an "always use" preference, **When** they click an event of that type, **Then** the preferred handler launches directly without prompting.

---

### User Story 6 - Resolve UI Handler Ambiguity (Priority: P3)

Multiple UI handlers exist for the same event type. The user is presented with a selection dialog and can set a default.

**Why this priority**: Only activates with competing UI handlers — a refinement.

**Independent Test**: Register two UI handlers, click an event, verify the dialog with "always use" option.

**Acceptance Scenarios**:

1. **Given** two UI handlers for "com.example.alert", **When** the user clicks an event of that type (no default), **Then** a selection dialog appears.
2. **Given** the dialog is shown, **When** the user selects handler A and checks "Always use", **Then** A launches and becomes the default.
3. **Given** the dialog is shown, **When** the user selects A without "Always use", **Then** A launches but the next event of that type will prompt again.

---

### User Story 7 - Configure Event Handlers (Priority: P4)

The Event Viewer's Configuration section lets the user:
- View all registered handlers (headless and UI) grouped by event type
- Enable/disable individual headless handlers
- Change the default UI handler per type
- Clear a UI handler default
- View retry/failure status for active handlers

**Why this priority**: Power-user control. System works without configuration.

**Independent Test**: Open config, disable a headless handler, emit an event, verify it's not invoked.

**Acceptance Scenarios**:

1. **Given** handlers are registered, **When** the user opens configuration, **Then** all handlers are listed grouped by event type with mode, app name, enabled state, and recent failure count.
2. **Given** a headless handler is enabled, **When** the user disables it, **Then** it is no longer invoked and does not block event completion.
3. **Given** a headless handler is disabled, **When** the user re-enables it, **Then** it resumes processing new events (and any un-acked queued events).
4. **Given** the user changes the default UI handler, **Then** subsequent clicks use the new default.

---

### Edge Cases

- **Emitter uninstalled after emitting**: Event remains, attributed by name. Headless handlers still process it. UI shows generic source label if the app is absent.
- **High volume (1000+ events)**: Viewer paginates/virtualizes. Bell caps display at "99+". Per-handler FIFO queue absorbs the load (events are queued, not dropped).
- **Simultaneous events of the same type**: Both stored with unique IDs and distinct sequence numbers. Dispatched to handlers in emission order.
- **Handler service offline at emission**: Handler is not "active" — does not block completion. When the service returns and registers, any matching un-acked events are delivered (at-least-once catch-up).
- **BOS restart with in-flight events**: Un-acked events are re-dispatched on startup. Handlers must be idempotent.
- **All handlers for a type disabled/uninstalled**: Event has no active subscribers → immediately processed on emission.
- **Handler permanently fails (3 retries exhausted)**: Event can still complete. Failure is visible in processing history. The handler's "recent failure count" in configuration shows the issue.
- **Payload > 1MB**: Emission rejected. Error message directs emitter to use VFS path reference.
- **UI handler + headless handler for same type**: Independent. Headless fires on emission; UI fires on click. They don't interfere.
- **User closes BOS while events are still pending**: Events persist. On restart, un-acked events are re-dispatched. Read/unread state persists.
- **Sequence gap**: If events are emitted but a handler is offline, sequence numbers may have gaps from the handler's perspective. The `sequence` field is for ordering, not completeness verification.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST provide a public event API (identical to the API exposed to agent tools) supporting: `emit` (type + payload, max 1MB), `ack` (event ID + optional result payload), `query` (by type, status, read state, time range), `register` (handler), `unregister` (handler), `mark-read` (event ID), and `set-preference` (default UI handler). All BOS components (services, apps, assistant) MUST use this same API.
- **FR-002**: The core event service MUST start at BOS boot, run for the lifetime of the session, and durably store all events with indefinite retention.
- **FR-003**: When an event is emitted, the core event service MUST assign it a monotonically increasing sequence number (per event type) and durably record it before the emission API call returns.
- **FR-004**: The core event service MUST invoke all active headless handlers for an event type upon emission (fan-out). Each handler MUST receive events sequentially in emission order (per-handler FIFO). The emitter's API call MUST NOT block on handler completion.
- **FR-005**: Delivery MUST be at-least-once: events that were dispatched but not yet acknowledged MUST be re-dispatched (a) on BOS restart, and (b) when a handler's service registers late after being offline. Handlers MUST be idempotent.
- **FR-006**: A headless handler MUST be able to acknowledge an event with an optional **result payload** (structured JSON). The result MUST be stored on the event's processing history.
- **FR-007**: If a headless handler invocation fails (exception or timeout), the core event service MUST retry up to 3 times with exponential backoff (1s, 5s, 30s). After the final retry fails, the handler MUST be marked "permanently failed" for that specific event.
- **FR-008**: An event MUST transition from **pending** to **processed** when ALL active headless handlers for its type have either acknowledged or been permanently failed. If an event type has no active headless handlers, it MUST be immediately processed upon emission.
- **FR-009**: The top toolbar bell icon MUST display the count of **unread** events. Clicking it MUST open the Event Viewer app.
- **FR-010**: The Event Viewer MUST be a built-in BOS core app whose default view displays **unread** events (both pending and processed) in reverse-chronological order, showing: event type, source name/icon, timestamp, summary, processing status badge ("processing…" or ✓), and sequence number.
- **FR-011**: The Event Viewer MUST provide a "Show historical events" toggle that reveals **read** events, including their full processing history.
- **FR-012**: Clicking an event in the Event Viewer MUST mark it as **read** and launch the appropriate UI handler (or generic view). A "Mark all as read" action MUST be available.
- **FR-013**: BOS MUST support handler registration in two modes: **headless** (invoked by core service on emission) and **UI** (invoked by user click). Any app or service MUST be able to declare handlers of either mode.
- **FR-014**: UI handlers MUST NOT participate in the event processing model. They MUST NOT block or affect the pending→processed transition, and they MUST NOT call `ack` on events.
- **FR-015**: When the user clicks an event with exactly one UI handler (or a user-set default), the system MUST launch that UI handler with the full event record.
- **FR-016**: When the user clicks an event with multiple UI handlers and no default, the system MUST present a selection dialog with an "Always use this app for this event type" option.
- **FR-017**: When the user clicks an event with no UI handler, the system MUST display a generic event detail view (full payload + processing history in structured format).
- **FR-018**: The Event Viewer MUST include a configuration section: view all handlers (grouped by type), enable/disable headless handlers, change/clear default UI handler, and view per-handler recent failure counts.
- **FR-019**: A disabled headless handler MUST NOT be invoked and MUST NOT block event completion. When re-enabled, it MUST resume processing (including any un-acked events in its queue).
- **FR-020**: When an app or service is uninstalled, its handler registrations MUST be removed. Pending events that were waiting for that handler MUST be re-evaluated: if no other active handlers remain, the event transitions to processed.
- **FR-021**: A headless handler failure MUST NOT affect other handlers processing the same event. Failures MUST be logged and visible in the event's processing history and in the configuration page's per-handler failure count.
- **FR-022**: The `ack` API MUST validate that the caller's app/service ID matches the handler being acknowledged. A component MUST NOT be able to ack events on behalf of a handler it does not own.
- **FR-023**: Handler registration MUST be validated: an app/service can only register handlers for event type namespaces it owns (namespace prefix must match its identifier or a granted namespace).
- **FR-024**: The emission API MUST reject payloads exceeding 1MB with a clear error. Emitters needing to pass large data MUST use a VFS path reference within the payload.
- **FR-025**: The Event Viewer MUST paginate or virtualize its event list to remain usable with 100,000+ events.
- **FR-026**: The feature MUST ship with documentation: (a) **Developer docs** — API reference (emit, ack, query, register, mark-read), handler registration (UI + headless), idempotency requirements, retry/timeout semantics, namespace conventions, payload limits, integration examples; and (b) **User docs** — viewing events, processing status, configuration, enabling/disabling handlers, interpreting processing history and failures. Both published in the BOS documentation hub.
- **FR-027**: User preferences (default UI handler per type, headless handler enabled/disabled state, read/unread state) MUST persist across sessions.
- **FR-028**: The Event Viewer MUST update in real-time: when an event's processing status changes (pending → processed) or a new unread event arrives while the viewer is open, the UI MUST reflect the change without requiring a manual refresh.

### Non-Functional Requirements

- **NFR-001**: The emission API MUST durably record the event before returning, and MUST NOT block the emitter for more than 100ms under normal load.
- **NFR-002**: The Event Viewer MUST render its initial list in under 500ms from bell click, even with 100,000+ stored events.
- **NFR-003**: The system MUST support at least 100,000 stored events without degradation in viewer or dispatch performance.
- **NFR-004**: UI handler resolution MUST complete in under 100ms.
- **NFR-005**: Event type namespaces MUST support dot-separated hierarchical identifiers, max 256 characters.
- **NFR-006**: Default timeout per headless handler invocation: 30 seconds (configurable per-app).
- **NFR-007**: Fan-out dispatch to N handlers MUST be concurrent across handlers (total time ≈ slowest handler, not sum). Within a single handler, processing MUST be sequential (one event at a time).
- **NFR-008**: The unread count (bell) MUST update within 1 second of the relevant state change (new event, final ack, user mark-read).
- **NFR-009**: Real-time updates in the Event Viewer (new events, status changes) MUST reflect within 1 second of the underlying state change.

### Key Entities

- **Event**: Attributes: unique ID, event type (namespaced string), payload (JSON, ≤1MB), source (app/service ID, display name, icon), timestamp, sequence number (per type, monotonically increasing), processing status (pending | processed), read status (unread | read), processing history (list of Handler Acknowledgment records), summary (derived from payload).
- **Handler Registration**: Attributes: event type, mode (headless | ui), providing app/service ID, handler identifier (function ref for headless, component ref for UI), display name, description, icon, enabled state (user-controllable for headless), timeout (configurable, default 30s).
- **Handler Acknowledgment**: One record per (event, handler) pair. Attributes: event ID, handler ID, app ID, attempt number, timestamp, status (acked | failed | permanently_failed), result payload (optional JSON, present on ack), error details (present on failure).
- **Handler Preference**: User-set default UI handler. Attributes: event type, preferred app ID, timestamp.
- **Event Source**: The component that emitted an event. Resolved to display name/icon at render time; gracefully handles absent apps.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: From event emission to the user seeing it in the Event Viewer (bell click → list render): under 3 seconds total under normal load.
- **SC-002**: The Event Viewer remains interactive (60fps scrolling, no input lag) with 100,000 stored events.
- **SC-003**: A developer can integrate event emission, headless handler registration with idempotent processing, and acknowledgment by following documented API contracts in under 30 minutes, without reading BOS source.
- **SC-004**: When multiple UI handlers exist for an event type, the user resolves the ambiguity in one dialog interaction; the "always use" preference eliminates future prompts.
- **SC-005**: The bell count accurately reflects unread events with ≤1s latency from any state change.
- **SC-006**: A headless handler begins processing an event within 500ms of emission (from durable record to handler invocation start).
- **SC-007**: A user can inspect the full processing history of an event (which handlers ran, results, failures, retries) in under 2 seconds from clicking the event.
- **SC-008**: No events are silently lost: if a handler was offline at emission time or BOS restarted mid-processing, the handler receives the event upon becoming available (at-least-once delivery).

## Assumptions

- The existing toolbar bell icon is repurposed to show the unified unread event count.
- Event types use dot-separated namespaces (e.g., "com.bos.gsuite.email.received").
- The payload summary in the list view comes from a `summary` field if present, else a truncated stringification.
- The Event Viewer is a standard BOS app window.
- UI handler components launch in their own window or pane.
- Background services are the primary emitters in practice.
- Handlers MUST be idempotent (documented requirement for all handler implementers).
- Per-handler processing is sequential (FIFO). Cross-handler ordering is not guaranteed.
- v1 does NOT support event replay to handlers that were offline *and* whose events were already fully processed (acked by all other handlers). At-least-once applies only to un-acked events.
- v1 does NOT support event filtering by type/source in the viewer; all events are shown.
- The public event API is one surface: the same functions exposed as agent tools are what apps/services call directly.
- An event with no active headless handlers is immediately processed on emission.
- Handler result payloads are stored indefinitely with the event.
- The 1MB payload limit applies to the JSON payload size, not to VFS-referenced data.

## Clarifications

### Session 2026-08-23

- **Q: Can background services emit events?**
  **A**: Yes. Services are in fact the primary emitters of events. The public event API is available to all BOS components (services, apps, assistant).

- **Q: Who dispatches headless handlers — the core service or the consuming app?**
  **A**: The core event service invokes all registered headless handlers on emission (fan-out), like any pub/sub system. It is NOT the consuming app's job to poll/listen; the service pushes to active handlers.

- **Q: How are UI handlers triggered relative to event processing?**
  **A**: UI handlers are NOT triggered by the core service and do NOT listen to the event queue. They are triggered only when the user opens the Event Viewer and clicks an event. They are a pure display/interaction layer and do NOT participate in the pending→processed ack model.

- **Q: When is an event marked processed?**
  **A**: When ALL active headless handlers for its type have acknowledged (or been permanently failed after max retries). If no active headless handlers exist for the type, it is immediately processed on emission. A handler that is disabled or whose service is not running is not "active" and does not block completion.

- **Q: Does the Event Viewer consume events on display?**
  **A**: No. The viewer "peeks" at the queue. Viewing does not change an event's processing status. What it does change is the **read** status (clicking an event marks it read).

- **Q: How does the user see processed events?**
  **A**: The Event Viewer shows unread events by default (regardless of processing status). A "Show historical events" toggle reveals read events. All events remain in persistent storage indefinitely.

- **Q: Can a handler include data when it acknowledges?**
  **A**: Yes. An ack may carry an optional result payload (structured JSON) that is stored on the event, so a UI handler (or the generic view) can inspect what happened when the event was processed.

- **Q: What is the retention policy?**
  **A**: Indefinite retention. Events are never auto-pruned; performance must hold at 100,000+ events.

- **Q: Is event replay supported?**
  **A**: At-least-once delivery: un-acked events are re-dispatched on restart or late registration. But events that are already fully processed (all handlers acked) are not re-delivered. Handlers must be idempotent.

- **Q: Is the public event API the same as the agent tool API?**
  **A**: Yes. The same API contract is used by emitters, subscribers, handlers, and agent tools — one surface for emit, ack, query, register, unregister, mark-read, set-preference.

### Session 2026-08-23 (architectural review)

- **Q: Should the bell show pending or unread count?**
  **A**: Unread. The user cares about "what haven't I seen yet," not pub/sub internal state. Processing status (pending/processed) is a secondary badge on each event.

- **Q: What happens when a handler permanently fails?**
  **A**: After 3 retries with backoff, the handler is "permanently failed" for that event. The event can still complete. The user sees the failure in processing history and the handler's failure count in configuration.

- **Q: What's the delivery guarantee?**
  **A**: At-least-once. Un-acked events are re-dispatched on restart and on late handler registration. Handlers MUST be idempotent.

- **Q: What's the ordering model?**
  **A**: Per-handler FIFO (sequential processing, one event at a time, in emission order). No cross-handler ordering guarantee. A sequence number is available for handlers that need to reorder.

- **Q: What's the payload size limit?**
  **A**: 1MB. Exceeding it is rejected. Use VFS path references for large data.

- **Q: Who can ack an event?**
  **A**: Only the handler owner. The ack API validates that the caller owns the handler being acked.

- **Q: What's the per-handler concurrency model?**
  **A**: Sequential (concurrency=1) per handler. This gives free FIFO ordering. If a handler is slow, events queue up (not dropped).
