# Feature Specification: Assistant Broker Capability

**Feature Branch**: `bos/040-assistant-broker-capability`

**Created**: 2026-08-25

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Add an `assistant` capability to the app sandbox broker so that opaque-origin (marketplace) apps can call the assistant run API through the parent frame, enabling any installed app to embed agentic chat without being same-origin."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A marketplace app embeds working agentic chat (Priority: P1)

A user installs a marketplace item (e.g. the Agentic Text Editor) that wants to embed a chat pane backed by the BOS assistant. The app is served `origin: "marketplace"`, runs in an opaque-origin sandbox, and previously could not reach the assistant run API because direct `fetch` calls are cross-origin with no CORS. With the `assistant` capability granted, the app calls broker methods to start a run, receives streamed text deltas, and sees a complete assistant reply.

**Why this priority**: This is the core capability — without it, every marketplace app that embeds assistant chat is broken. It unblocks an entire class of agentic marketplace items.

**Independent Test**: Install a marketplace-origin app with the `assistant` capability granted. Open it. Send a chat message. Verify a streamed assistant reply appears in the app's UI.

**Acceptance Scenarios**:

1. **Given** a marketplace-origin app has the `assistant` capability granted, **When** the app calls the broker to start an assistant run with a conversation, agent, and message, **Then** the broker returns a run ID and the run begins executing server-side.
2. **Given** an active run exists, **When** the app requests events for that run (with a since-cursor), **Then** the broker returns the buffered events (text deltas, tool calls, run finish) in order.
3. **Given** the app receives a text delta event, **When** the app renders it, **Then** the user sees streaming assistant output in the app's chat pane.
4. **Given** the run completes, **When** the app polls for events, **Then** the broker returns the `run_finished` event and no further events for that run.

---

### User Story 2 - A marketplace app participates in frontend tool calls (Priority: P1)

The assistant run emits a tool call marked `execution: "frontend"`. The app receives it in the event stream, executes the tool locally (e.g. editing a document buffer), and posts the result back through the broker. The run continues with the tool result.

**Why this priority**: Frontend tool execution is what makes agentic apps *agentic* — the assistant can manipulate the app's local state. Without this, the chat is read-only.

**Independent Test**: In a marketplace app with an `agentic_editor_*` surface tool, ask the assistant to "add a paragraph." Verify the tool call arrives, the app executes it, the result is posted, and the assistant acknowledges the edit.

**Acceptance Scenarios**:

1. **Given** a run is active and the model emits a tool call with `execution: "frontend"`, **When** the app polls events, **Then** the broker delivers the tool call event with its name, call ID, and arguments.
2. **Given** the app has received a frontend tool call, **When** the app posts the tool result back through the broker (run ID, call ID, result string), **Then** the server records the result and the run proceeds to the next step.
3. **Given** the app posts a tool result for a call that was already answered, **When** the server processes it, **Then** the first result wins and the duplicate is ignored (existing first-claim-wins behavior preserved).

---

### User Story 3 - The capability is grantable and revocable (Priority: P2)

The `assistant` capability follows the same grant/revoke lifecycle as existing app capabilities. An app declares it in its manifest; the user (or the system) grants it; the grant is persisted; the user can revoke it from Settings.

**Why this priority**: Consistency with the existing capability system is required for trust and security. Without a grant mechanism, every app gets the capability (or none do), which is neither.

**Independent Test**: In Settings → Apps, verify a new "Assistant" capability checkbox exists for an app that declares it. Toggle it off. Verify the app's broker calls to assistant methods are rejected.

**Acceptance Scenarios**:

1. **Given** an app's manifest declares the `assistant` capability, **When** the capability is granted (via Settings or initial install), **Then** the app can invoke assistant broker methods.
2. **Given** the `assistant` capability is revoked, **When** the app invokes an assistant broker method, **Then** the broker rejects the call with a capability-denied error.
3. **Given** an app has NOT declared the `assistant` capability in its manifest, **When** the app invokes an assistant broker method, **Then** the broker rejects the call (no grant possible without declaration).

---

### User Story 4 - Existing same-origin apps are unaffected (Priority: P2)

Apps that are served same-origin (`origin: "local"`) and use direct `fetch` to the assistant API continue to work without any change. The new broker methods are additive, not a replacement.

**Why this priority**: Regression protection. The capability must not break existing working integrations.

**Independent Test**: Open a same-origin app that uses `fetch("/api/assistant/runs")`. Verify it still works after the feature is deployed.

**Acceptance Scenarios**:

1. **Given** a same-origin app makes a direct `fetch` to `/api/assistant/runs`, **When** the feature is deployed, **Then** the fetch succeeds as before (no behavior change).
2. **Given** a same-origin app does NOT declare the `assistant` capability, **When** it uses direct fetch, **Then** it still works (the capability gates broker methods, not direct HTTP).

---

### Edge Cases

- **Run finishes between polls**: The app polls for events, the run completes before the next poll. The broker must return the remaining events (including `run_finished`) on the next poll, with a terminal flag so the app knows the run is done.
- **App reconnects after a transient drop**: The app loses its event poll (timeout, navigation). It reconnects with the last-seen sequence number. The broker replays events from that cursor.
- **Multiple active runs across apps**: Two different apps each have an active run (different conversation IDs). Broker calls for each are independent and do not interfere.
- **App posts a tool result after the run has already finished**: The broker accepts the post (no error), but the server ignores it since the run is no longer waiting for that call.
- **Very long assistant reply**: The event stream produces many text deltas. The broker delivers them in batches across multiple polls without loss or reordering.
- **Capability revoked mid-run**: The app has an active run, the user revokes the capability. In-flight event polls are still delivered (the run is server-owned); new run starts are rejected.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide an `assistant` app capability that, when granted to an app, allows that app to invoke the assistant run API through the broker.
- **FR-002**: System MUST expose broker methods for: listing available agents, starting a run (with conversation, agent, message, and optional surface tools), reading run events (with a since-cursor), posting frontend tool results, querying the active run for a conversation, and cancelling a run.
- **FR-003**: System MUST deliver run events to the requesting app in strict order, preserving the sequence numbers assigned by the run.
- **FR-004**: System MUST support resuming event delivery after a transient interruption, so an app that missed events (a poll gap, an iframe reload, a network blip) receives the missed events in order — no loss, no duplication — before continuing with new events.
- **FR-005**: System MUST deliver frontend tool call events (tool name, call ID, arguments) to the app through the event stream, enabling the app to execute the tool and post the result back.
- **FR-006**: System MUST return broker method results with the same semantic content as the equivalent direct HTTP API response (success returns the API response body; errors return the API error message and status).
- **FR-007**: System MUST validate the `assistant` capability against the app's granted capabilities before processing any assistant broker method call, rejecting with a capability-denied error if not granted.
- **FR-008**: System MUST expose the `assistant` capability in the app management Settings UI, allowing the user to grant or revoke it for apps that declare it.
- **FR-009**: System MUST support surface tools: when a run is started through the broker with surface tool declarations, those tools MUST be available to the model for the duration of the run, exactly as if the run were started via direct HTTP.
- **FR-010**: System MUST preserve the existing first-claim-wins semantics for frontend tool results: if multiple surfaces post a result for the same call ID, the first received result is used.

### Non-Functional Requirements

- **NFR-001**: Event delivery through the broker MUST have at most 500 ms additional latency compared to direct HTTP streaming (measured from event emission to broker delivery).
- **NFR-002**: The broker MUST NOT buffer unbounded event history in memory per run — events already delivered and acknowledged by the app can be discarded.
- **NFR-003**: Assistant broker methods MUST be non-blocking with respect to other broker calls — an in-progress event poll MUST NOT prevent other apps from making concurrent broker calls.
- **NFR-004**: The feature MUST be backward-compatible: existing direct-HTTP assistant API consumers (same-origin apps, the BOS UI's own chat) continue to function without modification.

### Key Entities

- **Assistant Capability Grant**: A per-app permission record (capability name `assistant`, granted/revoked state, timestamp) stored alongside existing app capability grants.
- **Broker Assistant Session**: The runtime association between a sandboxed app's iframe and its active assistant run(s) — holds the per-run event buffer, tracks the replay position (last-acknowledged sequence number) per app, and correlates frontend tool calls to their results.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A marketplace-origin app with the `assistant` capability can complete a full agentic conversation (start run → receive streamed text → execute a frontend tool call → post result → run completes) without any direct `fetch` to `/api/assistant/*`.
- **SC-002**: An app without the `assistant` capability receives a capability-denied rejection (not a timeout) when it attempts an assistant broker call, within 100 ms.
- **SC-003**: Event delivery latency through the broker is within 500 ms of the equivalent direct-HTTP delivery, measured over a 50-event run.
- **SC-004**: Zero regression: all existing same-origin apps and the BOS UI's own assistant chat continue to function identically after deployment.
- **SC-005**: The `assistant` capability appears in Settings → Apps for apps that declare it, and toggling it off immediately blocks subsequent assistant broker calls.

## Assumptions

- The assistant run API (`/api/assistant/runs` and its sub-routes) is the stable, server-owned contract. This feature proxies it; it does not change the API itself.
- The broker's postMessage transport (parent frame ↔ iframe) is the correct mechanism — not a WebSocket side-channel or a fetch-through-parent HTTP proxy. The parent frame already owns the same-origin context.
- The exact event-delivery transport — **parent-push** (the parent frame streams the run's events into the child as they arrive) versus **child-poll** (the child requests "events since seq N" on a timer/long-poll) — is a design-stage decision, not fixed here. Both satisfy the requirements, and both require the parent to hold a per-run event buffer so a child can replay from its last-acknowledged sequence number after a gap or reload. (A raw NDJSON stream cannot cross postMessage, so the parent must be the stream owner in either model.)
- Sequence numbers are monotonic per run and are the unit of resumption. The parent retains the run's events (at least from the lowest still-undelivered sequence) until the run finishes, then briefly afterward for reconnection (NFR-002).
- The `assistant` capability is declared in the app's manifest (`app.json` capabilities array), following the same pattern as `fs:read`, `storage`, etc.
- This feature does not change the assistant's model, tool execution semantics, conversation storage, or run lifecycle — it only adds a transport path to reach the existing API from a sandboxed origin.
- The Agentic Text Editor item will be updated separately (as a follow-up) to use the new broker methods when running opaque-origin; this spec covers only the BOS-side capability.
