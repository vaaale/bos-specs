# Feature Specification: Event & Notification System

**Feature Branch**: `034-event-notification-system`

**Created**: 2026-08-23

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Any application in BOS should be able to produce events. The producer may also define the event type and payload. If a user clicks on the notifications in the top toolbar, an event viewer app must be shown. The notification app must be part of BOS core, but extensible by any app in BOS including marketplace apps. An app can register an event handler UI part that handles certain event types. When the user clicks on an event in the event app, the appropriate handler will launch if such a handler is registered. If multiple apps have registered to handle the same event type, the user must be prompted to select which handler to launch, and have the option for 'Always use this app for this event type'. The event handler app must have a configuration page where these handlers can be configured / re-configured / cleared. If the user clicks on an event that has no handler associated with it, a generic event viewer app / dialog must be shown. In addition to UI handlers triggered by user clicks, an app must also be able to register a headless handler that processes the event automatically upon emission (e.g., the Workflow Manager listening to event types and triggering workflows). The system operates as a pub/sub system: the core event service invokes registered headless handlers on emission, handlers acknowledge processing, and an event is marked processed when all active handlers have acknowledged. UI handlers are a display layer only and do not block processing. The public event API is the same API that agent tools use."

## Architecture Overview *(informative — not a requirement)*

The system is a **pub/sub event broker** with two distinct handler modes:

- **Core Event Service** (BOS core, starts at boot):
  - Accepts events from emitters, stores them persistently (indefinite retention)
  - Invokes all registered **headless handlers** automatically on emission (fan-out)
  - Tracks per-event, per-handler acknowledgment status
  - An event transitions from **pending** → **processed** when ALL active headless handlers for its type have acknowledged
  - Exposes the pending count to the toolbar bell
  - Does NOT invoke UI handlers — those are triggered by user interaction in the Event Viewer

- **Emitters** (apps, services, assistant):
  - Publish events (type + payload) into the queue via the public event API
  - Background services are the primary emitters

- **Headless Handlers** (registered by apps or services):
  - Invoked by the core event service when a matching event is emitted
  - Process the event, then acknowledge via the public API
  - May include a **result payload** in the acknowledgment (stored on the event for later inspection)
  - A disabled handler is not "active" and does not block event completion

- **UI Handlers** (registered by apps):
  - NOT invoked by the core service
  - Triggered only when the user clicks an event in the Event Viewer
  - Serve as a display/interaction layer — they do NOT participate in the ack model
  - Can inspect the event including any handler result payloads
  - Do NOT block event processing

- **Event Viewer** (built-in app, opens on bell click):
  - Shows **pending** events by default
  - "Show historical events" reveals processed events
  - Routes user clicks to UI handlers
  - Configuration page for managing subscriptions and preferences

- **The public event API is the same API that agent tools use** — emitters, subscribers, and agents all interact through one contract.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View Pending Events (Priority: P1)

The top toolbar bell shows the count of **pending** events (events where not all active headless handlers have acknowledged). The user clicks the bell, and the Event Viewer opens showing pending events in reverse-chronological order. Each event displays its type, source, timestamp, and a summary. The user can also toggle to view **historical (processed) events**, which show the full processing history including handler results.

**Why this priority**: This is the foundational user journey — the user needs to see what's happening. It delivers immediate value by giving visibility into the event stream.

**Independent Test**: Can be fully tested by emitting a test event, observing the bell count increment, clicking the bell, and verifying the event appears as pending. After a registered headless handler acks it, the bell decrements and the event moves to historical.

**Acceptance Scenarios**:

1. **Given** no events exist, **When** a service emits a new event, **Then** the toolbar bell count increments by one and the event appears in the Event Viewer (pending list) when opened.
2. **Given** a pending event exists, **When** all active headless handlers for that event's type have acknowledged, **Then** the event is marked as processed, the bell count decrements, and the event is no longer shown in the default (pending) view.
3. **Given** processed events exist, **When** the user toggles "Show historical events" in the Event Viewer, **Then** processed events are listed with their processing history (which handlers acked, with what results, and when).
4. **Given** the Event Viewer is open, **When** the user dismisses the window, **Then** the pending count is preserved and the bell continues to display the correct count.
5. **Given** a pending event has been partially processed (1 of 3 handlers acked), **When** the user views the event in the Event Viewer, **Then** the event's processing status is visible (e.g., "1/3 handlers processed").

---

### User Story 2 - Emit Events from Any BOS Component (Priority: P1)

Any BOS component — background services (primary emitters), built-in apps, marketplace apps, or the assistant — can emit events through the public event API. The emitter specifies the event type (a namespaced identifier) and a payload (structured JSON). The event is durably recorded and immediately dispatched to matching headless handlers.

**Why this priority**: This is the producer side. Without emission, the system has nothing to process. Must exist alongside the viewer for the system to function.

**Independent Test**: Can be fully tested by calling the event emission API from a service context with a custom event type and payload, verifying the event is stored and dispatched to matching headless handlers.

**Acceptance Scenarios**:

1. **Given** a background service is running, **When** it emits an event via the public API with type "com.bos.gsuite.email.received" and a payload, **Then** the event is durably recorded and all matching headless handlers are invoked.
2. **Given** a marketplace app is running, **When** it emits an event via the same API, **Then** the event is recorded with the marketplace app's name and icon as the source.
3. **Given** the assistant (agent) is running, **When** it emits an event via the same API contract used by agent tools, **Then** the event is recorded with the assistant as the source.

---

### User Story 3 - Register Headless Handlers (Priority: P2)

An app or service registers one or more **headless handlers** for specific event types. The core event service invokes these handlers automatically when a matching event is emitted. Each handler processes the event and acknowledges via the public API, optionally including a result payload that is stored on the event.

**Why this priority**: This is the programmatic processing layer. It enables automation (workflow triggers, auto-archival, cross-app reactions) and is what transitions events from pending to processed.

**Independent Test**: Can be fully tested by registering a headless handler for a test event type, emitting an event, verifying the handler is invoked with the correct event data, and confirming the event transitions to processed after ack.

**Acceptance Scenarios**:

1. **Given** a service has registered a headless handler for "com.example.workflow.trigger", **When** any emitter emits an event of that type, **Then** the headless handler is invoked automatically with the full event record (type, payload, source, event ID).
2. **Given** three services have registered headless handlers for the same event type, **When** an event of that type is emitted, **Then** ALL three handlers are invoked (fan-out), each receiving the same event independently.
3. **Given** a headless handler processes an event and acknowledges with a result payload `{"workflowId": "wf-123", "status": "triggered"}`, **When** the user later views the event in the Event Viewer (historical), **Then** the handler's result payload is visible in the event's processing history.
4. **Given** a headless handler throws an error during processing, **When** the event has other handlers, **Then** the other handlers are unaffected; the failure is logged and the event remains pending for that handler.
5. **Given** a headless handler is disabled by the user in configuration, **When** a matching event is emitted, **Then** the disabled handler is not invoked and does not block event completion.
6. **Given** an app/service is uninstalled, **When** the event system checks active handlers, **Then** its headless handler registrations are removed.

---

### User Story 4 - Register UI Handlers (Priority: P2)

An app registers one or more **UI handlers** for specific event types. A UI handler is a UI component that is launched when the user clicks an event of that type in the Event Viewer. UI handlers do NOT participate in the event processing model — they are a display/interaction layer only.

**Why this priority**: This gives apps a way to present event-specific UI when the user wants to interact with an event. It's the "deep link" from notification to app context.

**Independent Test**: Can be fully tested by registering a UI handler for a test event type, emitting an event, waiting for it to be processed, then opening the Event Viewer (historical), clicking the event, and verifying the UI handler component launches with the event data.

**Acceptance Scenarios**:

1. **Given** an app has registered a UI handler for "com.bos.gsuite.email.received", **When** the user clicks an event of that type in the Event Viewer, **Then** the UI handler component is launched (in a window/pane/dialog) with the full event record including any handler results.
2. **Given** an event type has a UI handler registered, **When** the event is processed (all headless handlers acked), **Then** the UI handler's availability is unaffected — the user can still click the event (in historical view) to launch the UI handler.
3. **Given** an app is uninstalled, **When** the user clicks an event whose only UI handler belonged to that app, **Then** the generic event detail view is shown instead.

---

### User Story 5 - Click an Event to Launch Its UI Handler (Priority: P2)

The user clicks an event in the Event Viewer. If a UI handler is registered for that event type, it launches. If multiple UI handlers exist, the user is prompted to select. If none exist, the generic event detail view is shown.

**Why this priority**: This is the user-facing interaction that makes the Event Viewer actionable beyond just reading.

**Independent Test**: Can be fully tested by registering a UI handler, clicking an event of that type, and verifying the handler launches.

**Acceptance Scenarios**:

1. **Given** an event has exactly one registered UI handler and no user default is set, **When** the user clicks the event, **Then** the UI handler launches with the event's full record.
2. **Given** an event has no registered UI handler for its type, **When** the user clicks the event, **Then** a generic event detail view is shown displaying the full event payload and processing history in a structured format.
3. **Given** the user has set a "always use this handler" preference for an event type, **When** they click an event of that type, **Then** the preferred UI handler launches directly without prompting.

---

### User Story 6 - Resolve UI Handler Ambiguity (Priority: P3)

Multiple apps have registered UI handlers for the same event type. When the user clicks such an event, a selection dialog appears listing all available UI handlers with app names and icons. The user selects which to launch and can optionally set it as the default for that type.

**Why this priority**: Only activates in the case of competing UI handlers — a refinement of the click flow.

**Independent Test**: Can be fully tested by registering two UI handlers for the same type, clicking an event, and verifying the selection dialog with the "always use" option.

**Acceptance Scenarios**:

1. **Given** two apps have registered UI handlers for "com.example.alert", **When** the user clicks an event of that type (no default set), **Then** a selection dialog appears listing both with app name and icon.
2. **Given** the selection dialog is shown, **When** the user selects handler A and checks "Always use this app for this event type", **Then** handler A launches and becomes the default; subsequent clicks on that type launch A directly.
3. **Given** the selection dialog is shown, **When** the user selects handler A without checking "Always use", **Then** handler A launches but the next event of that type will prompt again.

---

### User Story 7 - Configure Event Handlers (Priority: P4)

The Event Viewer has a Configuration section where the user can:
- View all registered handlers (headless and UI) per event type
- Enable/disable individual headless handlers (disabled handlers are not invoked and don't block completion)
- Change the default UI handler for an event type
- Clear a UI handler default

**Why this priority**: Power-user control over routing. The system works out-of-the-box without configuration.

**Independent Test**: Can be fully tested by registering handlers, opening configuration, disabling a headless handler, emitting an event, and verifying the disabled handler is not invoked.

**Acceptance Scenarios**:

1. **Given** handlers are registered for various event types, **When** the user opens the configuration page, **Then** all handlers are listed grouped by event type, showing mode (headless/UI), providing app name, and enabled/disabled state.
2. **Given** a headless handler is enabled, **When** the user disables it, **Then** it is no longer invoked for new events and does not block event completion.
3. **Given** a headless handler is disabled, **When** the user re-enables it, **Then** it is invoked again for new events.
4. **Given** the user changes the default UI handler for a type, **Then** subsequent clicks on events of that type use the new default.

---

### Edge Cases

- What happens when an emitter app is uninstalled after emitting? → The event remains in storage, attributed to the (now-absent) app by name. Headless handlers still process it. The UI shows a generic source label.
- How does the system handle high volume (1000+ events)? → The Event Viewer MUST paginate or virtualize. The bell count MUST cap display (e.g., "99+") while tracking the true count.
- What happens when two events of the same type are emitted simultaneously? → Both are stored as distinct events with unique IDs; both are dispatched independently.
- What happens when a headless handler's providing service is not running? → The handler is not "active" and does not block event completion. The event can be processed by other active handlers. When the service comes back online, it does NOT retroactively receive previously emitted events (no replay in v1).
- What happens when the event payload is very large? → The list view shows only a summary. The full payload is loaded lazily when the event is opened.
- What happens if ALL headless handlers for an event type are disabled or uninstalled? → The event has no active subscribers and is immediately marked as processed (no one to ack). It appears only in historical view.
- What happens when a headless handler times out? → The handler is marked as failed (not acked). The event remains pending for that handler. The failure is logged. The event will not auto-complete for that handler.
- Can an app register both a UI handler and a headless handler for the same event type? → Yes. They operate in completely independent contexts: headless is invoked by the core service on emission; UI is triggered by user click.
- What happens to pending events when BOS is restarted? → Events persist. Headless handlers that were in-flight during the crash are considered failed (not acked). The events remain pending until handlers re-ack or are resolved.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST provide a public event API (identical to the API exposed to agent tools) that supports: emitting events (type + payload), acknowledging events (with optional result payload), querying events (by type, status, time range), and registering/unregistering handlers. All BOS components (services, apps, assistant) MUST use this same API.
- **FR-002**: The core event service MUST start at BOS boot and run persistently for the lifetime of the BOS session. It MUST durably store all events with indefinite retention.
- **FR-003**: When an event is emitted, the core event service MUST invoke ALL active headless handlers registered for that event type (fan-out dispatch). The invocation MUST be asynchronous — the emitter's API call returns after the event is durably stored, not after handlers complete.
- **FR-004**: An event MUST transition from **pending** to **processed** when ALL active headless handlers for its type have acknowledged. If an event type has no active headless handlers, it MUST be immediately marked as processed upon emission.
- **FR-005**: A headless handler MUST be able to acknowledge an event with an optional **result payload** (structured JSON). The result payload MUST be stored on the event record and be viewable in the Event Viewer's processing history.
- **FR-006**: The top toolbar bell icon MUST display the count of pending events (events not yet fully processed). Clicking it MUST open the Event Viewer app.
- **FR-007**: The Event Viewer MUST be a built-in BOS core app that, by default, displays **pending** events in reverse-chronological order, showing for each: event type, source app name/icon, timestamp, a summary, and processing status (e.g., "2/3 handlers acked").
- **FR-008**: The Event Viewer MUST provide a "Show historical events" toggle that reveals processed events, including their full processing history (which handlers acked, with what results, and when).
- **FR-009**: BOS MUST support handler registration in two modes: **headless** (invoked by the core service on emission) and **UI** (invoked by user click in the Event Viewer). Any app or service (built-in or marketplace) MUST be able to declare handlers of either mode.
- **FR-010**: UI handlers MUST NOT participate in the event processing model. They MUST NOT block or affect the pending→processed transition. They are a display/interaction layer only.
- **FR-011**: When the user clicks an event whose type has exactly one registered UI handler (or a user-set default among multiple), the system MUST launch that UI handler's component with the full event record (including processing history and handler results).
- **FR-012**: When the user clicks an event whose type has multiple registered UI handlers and no user-set default, the system MUST present a selection dialog listing all matching UI handlers (app name, icon, description) with a checkbox to "Always use this app for this event type."
- **FR-013**: When the user clicks an event whose type has no registered UI handler, the system MUST display a generic event detail view showing the full event payload and processing history in a structured, human-readable format.
- **FR-014**: The Event Viewer MUST include a configuration section where the user can view all registered handlers, enable/disable individual headless handlers, change the default UI handler for any type, or remove a UI handler default.
- **FR-015**: A disabled headless handler MUST NOT be invoked and MUST NOT block event completion. It is treated as if it is not active.
- **FR-016**: When an app or service is uninstalled, its handler registrations (both UI and headless) MUST be automatically removed. Any pending events that were waiting for that handler's ack MUST be re-evaluated: if no other active handlers remain, the event transitions to processed.
- **FR-017**: A headless handler failure (exception, timeout) MUST NOT affect: (a) the durable event record, (b) other headless handlers processing the same event, (c) UI handler availability. The failure MUST be logged with handler app ID, event ID, and error details. The event remains pending for the failed handler.
- **FR-018**: Handler registration MUST be validated: an app/service can only register handlers for event type namespaces it owns (the namespace prefix must match the app's identifier or a namespace it has been granted).
- **FR-019**: The Event Viewer MUST paginate or virtualize its event list to remain usable with 10,000+ events.
- **FR-020**: The feature MUST ship with two documentation artifacts: (a) **Developer documentation** covering the event emission API, handler registration (UI and headless), acknowledgment with result payloads, event type namespace conventions, and integration examples sufficient for a developer to integrate without reading BOS source; and (b) **User documentation** covering how to view pending/historical events, configure handlers, enable/disable handlers, and interpret processing history. Both MUST be published in the BOS documentation hub.
- **FR-021**: User preferences (default UI handler per event type, enabled/disabled state of headless handlers) MUST persist across sessions.

### Non-Functional Requirements

- **NFR-001**: The event emission API MUST durably record the event before the call returns, and MUST NOT block the emitter for more than 100ms under normal load.
- **NFR-002**: The Event Viewer MUST render its initial list (first page) in under 500ms from the moment the user clicks the bell, even with 10,000+ stored events.
- **NFR-003**: The system MUST support at least 100,000 stored events without degradation in viewer performance (indefinite retention).
- **NFR-004**: UI handler resolution (determining which UI handler to launch for a clicked event) MUST complete in under 100ms.
- **NFR-005**: The event type namespace MUST support dot-separated hierarchical identifiers (e.g., "com.bos.gsuite.email.received") with a maximum length of 256 characters.
- **NFR-006**: The default timeout for a single headless handler invocation MUST be 30 seconds (configurable per-app). On timeout, the handler is marked as failed.
- **NFR-007**: Headless handler dispatch MUST be concurrent: invoking N headless handlers for a single event MUST NOT take longer than the slowest individual handler (plus overhead), not the sum of all handler durations.
- **NFR-008**: The pending→processed transition MUST be reflected in the bell count within 1 second of the final ack.

### Key Entities

- **Event**: A single event record. Attributes: unique ID, event type (namespaced string), payload (structured JSON), source (app/service ID, display name, icon), timestamp, status (pending | processed), processing history (list of handler acks with timestamps and result payloads), summary (derived from payload).
- **Handler Registration**: A mapping from an event type to a handler. Attributes: event type, mode (headless | ui), providing app/service ID, handler identifier (function reference for headless, component reference for UI), display name, description, icon, enabled state (user-controllable for headless).
- **Handler Acknowledgment**: A record of a specific handler's processing of a specific event. Attributes: event ID, handler ID, app ID, timestamp, status (acked | failed | timeout), result payload (optional, structured JSON).
- **Handler Preference**: A user-set default UI handler for a given event type. Attributes: event type, preferred app ID, timestamp.
- **Event Source**: The component that emitted an event. Referenced by ID; resolves to display name and icon at render time (gracefully handles absent apps).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can go from "new event emitted" to "viewing event details in a UI handler" in under 3 seconds (bell click → event click → handler render) under normal system load.
- **SC-002**: The Event Viewer remains interactive (60fps scrolling, no input lag) with 100,000 stored events.
- **SC-003**: A developer can integrate event emission, headless handler registration, and acknowledgment by following documented API contracts, without modifying BOS source code, in under 30 minutes.
- **SC-004**: When multiple UI handlers exist for an event type, the user can resolve the ambiguity in a single dialog interaction, and the "always use" preference eliminates all future prompts for that type.
- **SC-005**: The toolbar bell count accurately reflects the number of pending events at all times, with a maximum latency of 1 second from the final handler ack to count update.
- **SC-006**: A headless handler receives and begins processing an event within 500ms of emission (measured from durable record to handler invocation start), with zero user interaction required.
- **SC-007**: A user can inspect the full processing history of a processed event (which handlers ran, with what results, in what order) in under 2 seconds from clicking the event in historical view.

## Assumptions

- The existing top toolbar bell icon (currently showing GSuite notification count) will be repurposed to show the unified pending event count from all sources.
- Event types use a dot-separated namespace convention (e.g., "com.bos.gsuite.email.received") to avoid collisions.
- The payload is arbitrary JSON; the "summary" in the list view is either a `summary` field in the payload (if present) or a truncated stringification.
- The Event Viewer is a standard BOS app window — it opens when the user clicks the bell.
- UI handler components are launched in their own app window or pane (not embedded within the Event Viewer itself).
- Background services are the primary emitters in practice, but the API is available to all components.
- v1 does NOT support event replay: if a headless handler's service was not running when an event was emitted, it does not retroactively receive that event. The event completes based on active handlers only.
- v1 does NOT support event filtering by type/source in the viewer list (the user sees all events); this is a potential future extension.
- The public event API surface (emit, ack, query, register, unregister) is the same API exposed as agent tools — one contract for both.
- An event with no active headless handlers is immediately processed (no one to wait for).
- Handler result payloads are stored indefinitely with the event (no separate retention policy for results).

## Clarifications

### Session 2026-08-23

- **Q: Can background services emit events?**
  **A**: Yes. Services are in fact the primary emitters of events. The public event API is available to all BOS components (services, apps, assistant).

- **Q: Who dispatches headless handlers — the core service or the consuming app?**
  **A**: The core event service invokes all registered headless handlers on emission (fan-out), like any pub/sub system. It is NOT the consuming app's job to poll/listen; the service pushes to active handlers.

- **Q: How are UI handlers triggered relative to event processing?**
  **A**: UI handlers are NOT triggered by the core service and do NOT listen to the event queue. They are triggered only when the user opens the Event Viewer and clicks an event. They are a pure display/interaction layer and do NOT participate in the pending→processed ack model.

- **Q: When is an event marked processed?**
  **A**: When ALL active headless handlers for its type have acknowledged. If no active headless handlers exist for the type, it is immediately processed on emission. A handler that is disabled (by user config) or whose service is not running is not "active" and does not block completion.

- **Q: Does the Event Viewer consume events on display?**
  **A**: No. The viewer "peeks" at the queue. Viewing does not change an event's status. Events remain in the queue until handlers process them.

- **Q: How does the user see processed events?**
  **A**: The Event Viewer shows pending events by default. A "Show historical events" toggle reveals processed events, which remain in persistent storage indefinitely and include their full processing history.

- **Q: Can a handler include data when it acknowledges?**
  **A**: Yes. An ack may carry an optional result payload (structured JSON) that is stored on the event, so a UI handler (or the generic view) can inspect what happened when the event was processed.

- **Q: What is the retention policy?**
  **A**: Indefinite retention. Events are never auto-pruned; performance must hold at 100,000+ events.

- **Q: Is event replay supported (delivering past events to a handler that was offline)?**
  **A**: No, not in v1. If a handler's service was not running at emission, it does not retroactively receive the event; the event completes based on active handlers only.

- **Q: Is the public event API the same as the agent tool API?**
  **A**: Yes. The same API contract is used by emitters, subscribers, handlers, and agent tools — one surface for emit, ack, query, register, and unregister.
