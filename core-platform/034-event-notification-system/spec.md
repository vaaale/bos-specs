# Feature Specification: Event & Notification System

**Feature Branch**: `034-event-notification-system`

**Created**: 2026-08-23

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Any application in BOS should be able to produce events. The producer may also define the event type and payload. If a user clicks on the notifications in the top toolbar, an event viewer app must be shown. The notification app must be part of BOS core, but extensible by any app in BOS including marketplace apps. An app can register an event handler UI part that handles certain event types. When the user clicks on an event in the event app, the appropriate handler will launch if such a handler is registered. If multiple apps have registered to handle the same event type, the user must be prompted to select which handler to launch, and have the option for 'Always use this app for this event type'. The event handler app must have a configuration page where these handlers can be configured / re-configured / cleared. If the user clicks on an event that has no handler associated with it, a generic event viewer app / dialog must be shown. In addition to UI handlers triggered by user clicks, an app must also be able to register a headless handler that processes the event automatically upon emission (e.g., the Workflow Manager listening to event types and triggering workflows)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View and Triage Notifications (Priority: P1)

A user receives events from various BOS applications (e.g., GSuite integration emails, system alerts, workflow completions). The top toolbar bell shows the count of unread events. The user clicks the bell, and the Event Viewer opens as a new app window showing a chronological list of events. Each event displays its type, source app, timestamp, and a brief summary. The user can mark events as read, and the bell count updates accordingly.

**Why this priority**: This is the foundational user journey — without the ability to view notifications, no other feature is useful. It delivers immediate value by finally giving users access to the events that are already being emitted by integrations.

**Independent Test**: Can be fully tested by emitting a test event from any source, observing the bell count increment, clicking the bell, and verifying the event appears in the viewer with correct metadata.

**Acceptance Scenarios**:

1. **Given** no events exist, **When** an integration emits a new event, **Then** the toolbar bell count increments by one and the event appears in the Event Viewer when opened.
2. **Given** the Event Viewer is open showing a list of events, **When** the user clicks on an event, **Then** the event is marked as read and the bell count decrements.
3. **Given** multiple events of different types and sources exist, **When** the Event Viewer is opened, **Then** events are displayed in reverse-chronological order, each showing its type, source app name, timestamp, and a one-line summary.
4. **Given** the Event Viewer is open, **When** the user dismisses the window, **Then** the unread count is preserved and the bell continues to display the correct count.

---

### User Story 2 - Emit Events from Any BOS App (Priority: P1)

Any application running in BOS — built-in apps, marketplace apps, or the assistant itself — can emit events through a standard event emission API. The emitting app specifies the event type (a namespaced identifier) and a payload (structured data describing the event). Events are immediately available in the notification system.

**Why this priority**: This is the producer side of the system. Without a way for apps to emit events, the viewer has nothing to show. It must exist alongside the viewer (User Story 1) for the system to be functional.

**Independent Test**: Can be fully tested by calling the event emission API from any app context with a custom event type and payload, then verifying the event appears in the Event Viewer with the correct type, payload, and source attribution.

**Acceptance Scenarios**:

1. **Given** a BOS app is running, **When** it calls the event emission API with a type "com.myapp.task.completed" and a payload, **Then** the event is stored and immediately visible in the Event Viewer.
2. **Given** a marketplace app is installed, **When** it emits an event via the same API, **Then** the event appears with the marketplace app's name and icon as the source.
3. **Given** an app emits an event, **When** the event is displayed in the viewer, **Then** the payload is available in full for the generic viewer and for any registered handler.

---

### User Story 3 - Register Event Handler UI (Priority: P2)

An app (built-in or marketplace) can declare one or more event handler UI components in its manifest/configuration. Each declaration maps an event type (or set of event types) to a UI component that the app provides. When that event type is triggered by a user click, the registered component is launched in the appropriate context (new window, pane, or dialog) with the event payload.

**Why this priority**: Handler registration is what makes events actionable. It's the bridge between "I see a notification" and "I can do something about it." It must exist for the click-to-launch flow (User Story 4) to be meaningful.

**Independent Test**: Can be fully tested by having a test app register a handler for a specific event type, emitting an event of that type, clicking it, and verifying the app's registered UI component appears with the correct payload.

**Acceptance Scenarios**:

1. **Given** an app declares a handler for event type "integration.gsuite.email" in its registration, **When** the BOS event system loads app registrations, **Then** the mapping is available for event routing.
2. **Given** a handler is registered for an event type, **When** the user clicks an event of that type, **Then** the handler's UI component is launched in the context defined by the handler (window/pane/dialog) with the event payload passed as input.
3. **Given** an app is uninstalled, **When** its handler registrations are checked, **Then** those registrations are no longer available; events of those types fall through to the generic viewer.

---

### User Story 4 - Click an Event to Launch Its Handler (Priority: P2)

The user has an event in their inbox that has a registered handler (e.g., a GSuite email event is handled by the GSuite app's mail pane). The user clicks on the event, and the appropriate handler UI launches, presenting the full event content in the context provided by the handler app.

**Why this priority**: This transforms the event system from a passive log into an actionable interface — the core differentiator from a simple notification list. It enables deep-linking from notifications to the relevant app context.

**Independent Test**: Can be fully tested by registering a handler for a test event type, emitting an event of that type, clicking it in the Event Viewer, and verifying the handler UI launches with the event payload.

**Acceptance Scenarios**:

1. **Given** an event with a registered handler exists and no default handler preference is set, **When** the user clicks the event, **Then** the registered handler launches with the event's payload passed to it.
2. **Given** an event has no registered handler for its type, **When** the user clicks the event, **Then** a generic event detail dialog is shown displaying the full event payload in a structured, readable format.
3. **Given** the user has set a "always use this handler" preference for a given event type, **When** they click an event of that type, **Then** the preferred handler launches directly without prompting.

---

### User Story 5 - Resolve Handler Ambiguity (Priority: P3)

Multiple apps have registered handlers for the same event type. When the user clicks such an event, a selection dialog appears listing all available handlers with their source app names and icons. The user selects which handler to launch and can optionally set it as the default for that event type.

**Why this priority**: This is a refinement of the core click-to-launch flow. It only activates in the case of competing handlers, so it can be built after the basic handler resolution works.

**Independent Test**: Can be fully tested by registering two handlers for the same event type from different apps, emitting an event of that type, clicking it, and verifying the selection dialog appears with both options and the "always use" toggle.

**Acceptance Scenarios**:

1. **Given** two apps have registered handlers for event type "com.example.alert", **When** the user clicks an event of that type (no default set), **Then** a selection dialog appears listing both handlers with app name and icon.
2. **Given** the selection dialog is shown, **When** the user selects handler A and checks "Always use this app for this event type", **Then** handler A launches and becomes the default for "com.example.alert"; subsequent events of that type launch handler A directly.
3. **Given** the selection dialog is shown, **When** the user selects handler A without checking "Always use", **Then** handler A launches but no default is set; the next event of that type will prompt again.

---

### User Story 6 - Configure Event Handlers (Priority: P4)

The Event Viewer has a Settings/Configuration section where the user can see all registered event handler mappings (event type → handler app), change the default handler for any type, or clear a handler registration entirely. This gives the user control over how events are routed without needing to go through each individual app.

**Why this priority**: Configuration is a secondary concern — the system must work out-of-the-box with auto-registered handlers before the user needs to manage them. This is power-user functionality.

**Independent Test**: Can be fully tested by registering handlers from multiple apps, opening the Event Viewer's configuration page, verifying all mappings are listed, changing a default, clearing a registration, and confirming the changes take effect on subsequent event clicks.

**Acceptance Scenarios**:

1. **Given** handlers are registered for various event types (both UI and headless), **When** the user opens the Event Viewer configuration page, **Then** all registered event type → handler mappings are listed with the event type, handler mode (UI or Headless), handler app name, and whether a UI default is set.
2. **Given** the configuration page is open, **When** the user changes the default UI handler for an event type, **Then** the change is persisted and subsequent clicks on events of that type use the new default.
3. **Given** the configuration page is open, **When** the user clears the default UI handler for an event type, **Then** the preference is removed; subsequent clicks on events of that type will prompt if multiple handlers exist, or use the single registered handler if only one exists.
4. **Given** a headless handler is registered and enabled for an event type, **When** the user disables it from the configuration page, **Then** the headless handler is no longer invoked when events of that type are emitted, until re-enabled.

---

### User Story 7 - Automatic Headless Event Processing (Priority: P2)

An app registers a **headless handler** for one or more event types. Unlike UI handlers (which respond to a user click in the Event Viewer), headless handlers are invoked **automatically and asynchronously** when a matching event is emitted — no user interaction required. The handler receives the full event (type, payload, source) and processes it programmatically. For example, the Workflow Manager app registers headless handlers for several event types and triggers the appropriate workflow when a matching event is emitted.

**Why this priority**: This is the programmatic side of event handling. It enables automation pipelines (workflow triggers, auto-archival, cross-app reactions) without requiring the user to be present. It's a distinct dispatch modality from UI handlers and has its own failure-isolation and concurrency requirements.

**Independent Test**: Can be fully tested by registering a headless handler for a test event type (e.g., a logging handler that writes to a file), emitting an event of that type without opening the Event Viewer, and verifying the handler was invoked with the correct event data.

**Acceptance Scenarios**:

1. **Given** an app has registered a headless handler for event type "com.example.workflow.trigger", **When** any app emits an event of that type, **Then** the headless handler is invoked automatically (without user interaction) with the full event record.
2. **Given** three apps have registered headless handlers for the same event type, **When** an event of that type is emitted, **Then** ALL three headless handlers are invoked (fan-out), each receiving the same event record independently.
3. **Given** a headless handler throws an error or times out during processing, **When** the event was also handled by other headless handlers, **Then** the other handlers are unaffected; the failure is logged and does not prevent the event from being recorded or from other handlers completing.
4. **Given** a headless handler is registered for an event type, **When** the user clicks an event of that type in the Event Viewer, **Then** the UI handler resolution flow is unaffected (headless handlers are not presented as click-time options; they are not UI components).
5. **Given** a headless handler belongs to an app that is uninstalled, **When** the event system checks active handlers, **Then** the headless handler registration is removed and no longer invoked.

---

### Edge Cases

- What happens when an app emits an event but the app is subsequently uninstalled? → The event remains in the inbox, attributed to the (now-absent) app by name. Clicking it shows the generic event viewer since no handler is registered.
- How does the system handle a very high volume of events (e.g., 1000+ emails in a short period)? → The Event Viewer MUST paginate or virtualize its list to remain responsive. The bell count MUST cap its display (e.g., "99+") while tracking the true count.
- What happens when two events of the same type are emitted simultaneously by the same source? → Both are stored as distinct events with unique identifiers; they appear as separate entries in the viewer.
- What happens when a handler's app is currently not running / not loaded? → The system MUST launch the app (or its handler component) on demand when the user clicks the event.
- What happens when the event payload is very large (e.g., a full email body)? → The Event Viewer list shows only a summary; the full payload is loaded lazily when the event is opened.
- What happens when a handler registration is for an event type that uses a wildcard or pattern? → Wildcard/pattern matching is out of scope for v1; registrations MUST be for exact event type strings.
- What happens when a headless handler takes longer than a reasonable timeout to process? → The system MUST impose a per-handler timeout (configurable, default 30s). On timeout, the handler is considered failed; the event is not re-dispatched to it, and the failure is logged.
- What happens when a very large number of headless handlers (e.g., 50+) are registered for the same event type? → All are invoked, but the system MUST process them concurrently (not sequentially) to avoid unbounded latency. The emitter is never blocked by headless handler processing.
- Can an app register both a UI handler and a headless handler for the same event type? → Yes. They are independent: the headless handler fires on emission, the UI handler is available for click-time routing. They do not interfere with each other.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST provide a standard event emission API available to all running applications (built-in, marketplace, and assistant) that accepts a namespaced event type (string) and a structured payload (JSON), and records the event with a unique ID, timestamp, source app identifier, and the emitting app's display name and icon.
- **FR-002**: The top toolbar bell icon MUST display the count of unread events. Clicking it MUST open the Event Viewer app.
- **FR-003**: The Event Viewer MUST be a built-in BOS core app that displays all events in reverse-chronological order, showing for each: event type, source app name/icon, timestamp, and a one-line summary derived from the payload.
- **FR-004**: The Event Viewer MUST allow the user to mark individual events as read (on click) and MUST provide a "mark all as read" action.
- **FR-005**: BOS MUST support event handler registration in two modes: **UI handlers** (event type → UI component, invoked on user click in the Event Viewer) and **headless handlers** (event type → programmatic callback, invoked automatically on event emission). Any app (built-in or marketplace) MUST be able to declare one or more handlers of either mode in its app manifest or via a registration API.
- **FR-006**: When the user clicks an event whose type has exactly one registered handler (or a user-set default among multiple), the system MUST launch that handler's UI component with the event's full payload as input.
- **FR-007**: When the user clicks an event whose type has multiple registered handlers and no user-set default, the system MUST present a selection dialog listing all matching handlers (app name, icon, description) with a checkbox to "Always use this app for this event type."
- **FR-008**: When the user clicks an event whose type has no registered handler, the system MUST display a generic event detail view showing the full payload in a structured, human-readable format.
- **FR-009**: The Event Viewer MUST include a configuration section where the user can view all registered event type → handler mappings, change the default handler for any type, or remove a user-set default.
- **FR-010**: User preferences (default handler per event type) MUST persist across sessions.
- **FR-011**: Events MUST persist across BOS sessions (they are not ephemeral/in-memory only).
- **FR-012**: The Event Viewer MUST paginate or virtualize its event list to remain usable with 1000+ events.
- **FR-013**: Handler registration MUST be validated: an app can only register handlers for event type namespaces it owns (the namespace prefix must match the app's identifier or a namespace the app has been granted).
- **FR-014**: When an app is uninstalled, its handler registrations (both UI and headless) MUST be automatically removed from the routing table.
- **FR-015**: Headless handlers MUST be invoked automatically and asynchronously when a matching event is emitted. The invocation MUST NOT block the emitting app or the durable event record. All registered headless handlers for a given event type MUST be invoked (fan-out), each receiving the full event record independently.
- **FR-016**: A headless handler failure (exception, timeout) MUST NOT affect: (a) the durable event record, (b) other headless handlers processing the same event, (c) the UI handler routing for subsequent user clicks. Failures MUST be logged with the handler's app ID, event ID, and error details.
- **FR-017**: The Event Viewer's configuration section MUST display headless handler registrations alongside UI handler registrations, clearly labeled by mode. The user MUST be able to enable/disable individual headless handlers without uninstalling the providing app.
- **FR-018**: The feature MUST ship with two documentation artifacts: (a) **Developer documentation** covering the event emission API, handler registration (UI and headless), event type namespace conventions, payload structure, and integration examples sufficient for a marketplace app developer to integrate without reading BOS source code; and (b) **User documentation** covering how to view events, configure handler defaults, enable/disable headless handlers, and interpret the generic event detail view. Both MUST be published in the BOS documentation hub.

### Non-Functional Requirements

- **NFR-001**: The event emission API MUST durably record the event before the call returns, and MUST NOT block the emitting app's main thread for more than 100ms under normal load.
- **NFR-002**: The Event Viewer MUST render its initial list (first page) in under 500ms from the moment the user clicks the bell, even with 10,000+ stored events.
- **NFR-003**: The system MUST support at least 10,000 stored events without degradation in viewer performance.
- **NFR-004**: Handler resolution (determining which handler to launch for a clicked event) MUST complete in under 100ms.
- **NFR-006**: The default timeout for a single headless handler invocation MUST be 30 seconds. On timeout, the handler is marked as failed and the error is logged. The timeout MUST be configurable per-app.
- **NFR-007**: Headless handler dispatch MUST be concurrent: invoking N headless handlers for a single event MUST NOT take longer than the slowest individual handler (plus overhead), not the sum of all handler durations.
- **NFR-005**: The event type namespace MUST support dot-separated hierarchical identifiers (e.g., "com.bos.gsuite.email.received") with a maximum length of 256 characters.

### Key Entities

- **Event**: A single notification record. Attributes: unique ID, event type (namespaced string), payload (structured JSON), source app ID, source app display name, source app icon, timestamp, read/unread status, summary (derived from payload).
- **Handler Registration**: A mapping from an event type to a handler provided by a specific app. Two modes: **UI** (handler component identifier, launch context: window/pane/dialog) and **Headless** (programmatic callback endpoint). Common attributes: event type, mode (ui | headless), providing app ID, display name, description, icon, enabled/disabled state (user-controllable for headless).
- **Handler Preference**: A user-set default for a given event type. Attributes: event type, preferred app ID, timestamp of preference.
- **Event Source**: The app that emitted an event. Referenced by app ID; resolves to display name and icon at render time (gracefully handles absent apps).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can go from "new event emitted" to "viewing event details in a handler" in under 3 seconds (bell click → event click → handler render) under normal system load.
- **SC-002**: The Event Viewer remains interactive (60fps scrolling, no input lag) with 10,000 stored events.
- **SC-003**: A marketplace app developer can integrate event emission and handler registration by following documented API contracts, without modifying BOS source code, in under 30 minutes of development time.
- **SC-004**: When multiple handlers exist for an event type, the user can resolve the ambiguity (select a handler) in a single dialog interaction, and the "always use" preference eliminates all future prompts for that type.
- **SC-005**: The toolbar bell count accurately reflects the number of unread events at all times, with a maximum rendering latency of 1 second from event emission to count update.
- **SC-006**: A headless handler registered for an event type receives and begins processing the event within 500ms of emission (measured from durable record to handler invocation start), with zero user interaction required.

## Assumptions

- The existing top toolbar bell icon (currently showing GSuite notification count) will be repurposed/extended to show the unified event count from all sources, not just GSuite.
- Event types use a reverse-DNS or dot-separated namespace convention (e.g., "com.bos.gsuite.email.received", "com.marketplace.myapp.task.done") to avoid collisions between apps.
- The payload is arbitrary JSON; the "summary" shown in the list view is either a human-readable `summary` field within the payload (if present) or a truncated stringification of the payload.
- The Event Viewer is a standard BOS app window (not a popover or dropdown) — it opens as a full app surface.
- Handler UI components are launched in their own app window or pane (not embedded within the Event Viewer itself), preserving the existing app window model.
- v1 does not support event filtering by type/source in the viewer list (the user sees all events); this is a potential future extension.
- v1 does not support event actions (e.g., "reply", "snooze", "archive") beyond marking as read; handlers provide their own actions within their own UI.
- The configuration page (FR-009) allows changing user preferences (defaults) for UI handlers and enabling/disabling headless handlers, but does NOT allow removing the underlying handler registration (that is owned by the app itself and removed on uninstall).
- Headless handlers are dispatched in a fan-out model: ALL registered headless handlers for a matching event type are invoked. This contrasts with UI handlers where the user selects one (or a default is used). There is no "default headless handler" concept — they all run.
- An app MAY register both a UI handler and a headless handler for the same event type; they operate independently in their respective dispatch contexts.
