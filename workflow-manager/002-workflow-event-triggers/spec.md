# Feature Specification: Workflow Event Triggers

**Feature Branch**: `bos/002-workflow-event-triggers` (per-project numbering under the Workflow Manager project)

**Created**: 2026-08-25

**Status**: Draft

**App Target**: marketplace-item

**Input**: User description: "Add functionality to the Workflow Manager so that a workflow can be triggered on a specific event from the event system."

## Context

This is an incremental scope-add to the already-installed Workflow Manager marketplace item (app id `workflows`), whose baseline is spec `001-workflow-manager-service-tools` (implemented & promoted). That baseline established the 039-compliant shape: a worker-thread background service owns the complete workflow engine (node model, dynamic routing, parallel scheduling, persistence, execution) and exposes the workflow lifecycle as service-declared native tools (`workflow_list` / `create` / `read` / `modify` / `run` / `status` / `run_list` / `run_get` / `cancel` / `delete` / `export` / `validate`). Workflows and their run logs persist to the real VFS under `/Workflows/`.

BOS's event system (spec 034, `event-notification-system`, implemented) publishes durable events (type + JSON payload) onto an event bus and supports **headless handlers** that a background service registers at runtime over worker IPC — the same runtime-declaration channel the workflow service already uses to declare its 039 tools. When a published event matches a registered headless handler's event type, the event kernel invokes that handler in the service; the service then decides what to do.

Today a workflow can only be started manually (via the app or a `workflow_run` tool call). This feature adds **event triggers**: a workflow may be configured with one or more event triggers, each naming a specific event type (optionally with a payload filter); the service subscribes to the distinct event types its workflows reference, and when a matching event is published it fires the workflow's run automatically — no user or assistant action in the loop.

The trigger is a **configuration of the workflow** (a declarative `triggers` list on the workflow), not a separate entity. It composes with everything the baseline already does: an event-triggered run is an ordinary workflow run — it streams step events, persists a run log, supports cancellation, and is inspectable in history — with the added provenance of *which event started it*.

**No BOS-source change is required.** The headless-handler runtime-declaration and dispatch machinery already exists (spec 034). This feature is entirely inside the `workflows` item's service facet (subscribe + match + fire) and app facet (configure/view triggers), plus the tool surface the service already declares.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A Workflow Runs Automatically When Its Trigger Event Fires (Priority: P1)

A workflow is configured with an event trigger that names a specific event type. When an event of that type is published on the event bus, the workflow's run is started automatically by the service — with no manual `workflow_run` call and no user/assistant in the loop.

**Why this priority**: This is the headline capability the user asked for. Without it there is no event triggering. It delivers the core value: the workflow manager becomes reactive to BOS's event stream, so a long tail of "when X happens, run Y" automations become declarative.

**Independent Test**: Configure a workflow with a trigger on event type `com.bos.assistant.task.done`; emit an event of that type via the event system; confirm a run of the workflow starts automatically and a run record is created. Testable in isolation against a running service.

**Acceptance Scenarios**:

1. **Given** a workflow has a trigger for event type `T` and the service is running, **When** an event of type `T` is published, **Then** a run of the workflow is started automatically without any manual invocation.
2. **Given** a workflow has a trigger for event type `T`, **When** an event of a different type is published, **Then** that workflow does not start.
3. **Given** a workflow has triggers for types `A` and `B`, **When** an event of type `A` is published, **Then** the workflow runs; when an event of type `B` is later published, the workflow runs again.
4. **Given** a workflow has a trigger for type `T` but the service is stopped, **When** an event of type `T` is published, **Then** the workflow does not run (the handler is not registered) and no run is recorded.

---

### User Story 2 - Event Triggers Are Configurable Through the Tool Surface (Priority: P1)

Because the trigger is part of a workflow's authorable configuration, the service-declared workflow tools create/modify a workflow's triggers, and a dedicated inspection tool returns the triggers attached to a workflow — so the assistant (or any tool caller) can build, edit, and read event triggers without the UI.

**Why this priority**: Consistent with the baseline's FR-014 ("every lifecycle operation achievable via tools without the UI"), triggers must be tool-reachable. An agentic-first workflow builder needs to attach a trigger when it composes a workflow, not only via a UI form.

**Independent Test**: Use the workflow tools to create a workflow whose config includes an event trigger; read it back and confirm the trigger is present; modify the trigger's event type and confirm the change persists; confirm the service re-subscribes to the new event type.

**Acceptance Scenarios**:

1. **Given** the workflow tools, **When** a caller creates or modifies a workflow with an event trigger, **Then** the trigger is persisted with the workflow and survives a service restart.
2. **Given** a workflow with an event trigger, **When** a caller inspects it via tools, **Then** the trigger(s) are returned with their event type (and filter, if set).
3. **Given** a workflow's trigger event type is changed via a tool, **When** the change is persisted, **Then** the service reflects the new subscription (subscribes to the new type, and no longer to the old type if it is no longer referenced by any of that workflow's triggers).
4. **Given** a caller removes all triggers from a workflow, **When** the change persists, **Then** the workflow no longer runs automatically on any event.

---

### User Story 3 - The App Lets You Configure and Inspect Event Triggers (Priority: P2)

The Workflow Manager app's workflow detail view lets the user add, edit, and remove event triggers on a workflow, and shows the workflow's current triggers — so event automation is manageable from the UI alongside the graph.

**Why this priority**: The baseline established the app as a parallel surface to the tools. Trigger management is a first-class authoring task users will do visually (pick an event type, optionally a filter), and the run list already exists to show what ran — so the UI surface completes the loop.

**Independent Test**: Open a workflow in the app; add an event trigger for a type; confirm it appears in the trigger list and persists on reload; emit a matching event; confirm the run shows up in the workflow's run history.

**Acceptance Scenarios**:

1. **Given** the workflow detail view, **When** the user adds an event trigger, **Then** it is shown in the workflow's trigger list and persisted.
2. **Given** a workflow with a trigger, **When** the user edits its event type or filter and saves, **Then** the persisted trigger is updated.
3. **Given** a workflow with a trigger, **When** the user removes it, **Then** the workflow no longer triggers on that event.
4. **Given** the user views the workflow's run history after an event-triggered run, **Then** the run is listed (as in the baseline) and identifiable as event-triggered.

---

### User Story 4 - Triggered Runs Carry Their Trigger Provenance (Priority: P2)

Each run started by an event trigger records which event started it (event id and type), so a later inspection of the run can answer "what fired this?" — distinct from manually started runs.

**Why this priority**: An event-triggered run is otherwise indistinguishable from a manual one in history; provenance is what makes the reactive automation auditable and is a natural, small extension of the baseline's persisted run entity (which already stores id, timestamps, final state, per-step outcomes, event log).

**Independent Test**: Fire a workflow via an event; open the resulting run; confirm it records the triggering event's id and type, and that a manually-run workflow's run does not.

**Acceptance Scenarios**:

1. **Given** a run started by an event trigger, **When** the run is inspected, **Then** it records the triggering event's id and type (and the start source is marked as "event").
2. **Given** a run started manually (tool or UI), **When** the run is inspected, **Then** it is marked as manually started and carries no triggering-event reference.
3. **Given** the run-list tool, **When** it returns runs, **Then** each run's start source (event vs. manual) is distinguishable.

---

### User Story 5 - Trigger Matching and Firing Behavior Are Bounded (Priority: P2)

The trigger's match semantics and the behavior when a trigger event arrives while the workflow is already running are well-defined, so event-triggering is predictable and safe under real event traffic.

**Why this priority**: Event buses can emit the same type repeatedly or in bursts; without defined match granularity and re-entrancy semantics, a busy event type could either miss intended runs or spawn uncontrolled run floods. This is the correctness/safety half of the trigger.

**Independent Test**: Configure a workflow trigger; publish the same event type multiple times in quick succession and observe the defined re-entrancy behavior; (if payload filtering is in scope) configure a filter and confirm only matching payloads fire.

**Acceptance Scenarios**:

1. **Given** a workflow trigger for type `T`, **When** multiple events of type `T` are published in succession, **Then** the run-firing behavior follows the defined re-entrancy policy (see Q3), with no crash and no lost/duplicated run beyond the policy.
2. **Given** a trigger with a payload filter, **When** an event of the right type but a non-matching payload is published, **Then** the workflow does not run.
3. **Given** a trigger with a payload filter, **When** an event of the right type and a matching payload is published, **Then** the workflow runs.
4. **Given** an event of type `T` that matches the trigger, **When** the service starts the run, **Then** the run is fire-and-poll (consistent with the baseline: `runId` returned, progress polled) and the trigger does not block on the run's completion.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A workflow's configuration MUST support a list of zero or more **event triggers**; each trigger MUST name exactly one event type from the event system (and MAY carry an optional payload filter, per FR-006).
- **FR-002**: The service MUST subscribe (via the event system's headless-handler runtime-declaration channel) to the **union** of event types referenced by the triggers of all workflows at service start, and MUST keep that subscription set current as workflows and their triggers are created, modified, or deleted.
- **FR-003**: When a published event's type matches a workflow trigger's event type, the service MUST automatically start a run of that workflow, with no manual `workflow_run` call required.
- **FR-004**: An event-triggered run MUST be an ordinary workflow run: it MUST stream step events, persist a run log, support cancellation, and be inspectable in history, exactly as a manually started run does.
- **FR-005**: The service MUST reflect changes to trigger subscriptions promptly: when a workflow's trigger event type is changed or removed, the service MUST stop subscribing to the now-unreferenced type and subscribe to the newly referenced type, without requiring a service restart.
- **FR-006**: [NEEDS CLARIFICATION: payload-filter granularity — is the trigger match on exact event type only (P1 default), or does it also support matching/filtering on the event's JSON payload fields (e.g. only fire when `payload.kind === "error"`)? See Q1.]
- **FR-007**: A run started by an event trigger MUST record its **trigger provenance**: the triggering event's id and type, and a start-source marker distinguishing it from a manually started run.
- **FR-008**: The workflow tools MUST support authoring event triggers: `workflow_create`/`workflow_modify` MUST accept triggers in the workflow config, and a read/inspection tool MUST return a workflow's triggers. (Consistent with the baseline's tool-surface control-plane requirement.)
- **FR-009**: The app's workflow detail view MUST allow the user to add, edit, and remove event triggers on a workflow, and MUST display the workflow's current triggers.
- **FR-010**: When the service is stopped, its event handlers MUST be unregistered such that no event-triggered run can start (no stale dispatch to a stopped service), consistent with the baseline's tool-removal-on-stop guarantee.
- **FR-011**: The run-list/run-get tools and the app run history MUST expose each run's start source (event-triggered vs. manual) so event-triggered runs are distinguishable.
- **FR-012**: An event trigger MUST NOT block the event's delivery path: the service's response to a triggering event MUST be fast (start the run and return), deferring the long run to the existing asynchronous fire-and-poll execution model — a long-running workflow MUST NOT delay or hold the event dispatch.
- **FR-013**: [NEEDS CLARIFICATION: re-entrancy policy — when a trigger event arrives while a run of the same workflow is already executing, what happens? Options: (a) always start a new run regardless (default), (b) coalesce/skip if a run for that trigger is already active, (c) queue. See Q3.]
- **FR-014**: Trigger configuration MUST be validated: a trigger MUST reference a well-formed event type; a trigger with an empty/invalid event type MUST be rejected by the create/modify tools with a clear error.

### Non-Functional Requirements

- **NFR-001**: Subscribing to and reacting to a triggering event MUST NOT crash BOS or the service; an error matching/firing a trigger MUST be logged (structured, via the service's log channel) and MUST NOT be silently swallowed.
- **NFR-002**: The set of event types a workflow's triggers reference MUST be re-derivable from the persisted workflow config alone, so a service restart re-subscribes correctly with no separate subscription store.
- **NFR-003**: Trigger persistence MUST live with the workflow in the real VFS `/Workflows/` (as part of the workflow JSON), consistent with the baseline's storage model — no separate host-path or control-dir storage.
- **NFR-004**: A burst of matching events MUST NOT degrade event-bus delivery to other consumers; the workflow trigger path MUST be a well-behaved subscriber (bounded work per event, no unbounded fan-out of runs beyond the re-entrancy policy).

### Key Entities

- **EventTrigger**: A declarative trigger on a workflow — an event type (required), an optional payload filter (FR-006), and identity metadata. Part of the workflow's configuration, persisted with the workflow.
- **Workflow**: (baseline) A user-authored multi-step graph persisted as JSON at the real VFS `/Workflows/<id>-workflow.json`. This feature extends its config with a `triggers` list of EventTrigger.
- **Run**: (baseline) A single execution of a workflow, persisted to the real VFS. This feature extends it with a start-source marker and, when event-triggered, the triggering event's id + type.
- **Event**: (event system, spec 034) A durable published record — a type and a JSON payload — delivered to matching registered handlers.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A workflow configured with an event trigger runs automatically when a matching event is published, with zero manual intervention — 100% of correctly-typed published events that match a live trigger (and pass any filter) start a run.
- **SC-002**: Events whose type matches no active workflow trigger cause zero workflow runs (no false positives).
- **SC-003**: A workflow's trigger set is fully authorable via the workflow tools (create/modify/read) — 100% of trigger operations are achievable without the UI, and changes persist across a service restart.
- **SC-004**: An event-triggered run is indistinguishable in capability from a manual run (streams, persists, cancellable, inspectable) and is additionally marked with its triggering event's id + type.
- **SC-005**: With the service stopped, publishing a matching event causes no workflow run and no error/crash; starting the service again re-establishes the subscriptions from the persisted workflow config.
- **SC-006**: A burst of matching events does not delay or drop event delivery to other consumers, and does not spawn more runs than the re-entrancy policy allows.
- **SC-007**: Changing a trigger's event type via a tool updates the active subscription without a service restart — a subsequently published event of the new type fires the workflow, and the old type no longer does.

## Assumptions

- The event system's **headless-handler runtime-declaration + dispatch** mechanism (spec 034, implemented) is the transport by which the service subscribes to and is invoked on event types — the same worker-IPC runtime-declaration channel the service already uses for its 039 `tool_declare`. The exact wire message type (e.g. a `handler_declare`/`handler_undeclare` emission) is an implementation detail for design.md; the spec only requires the capability.
- No BOS-source change is required: both the headless-handler machinery and the service tool/IPC channel already exist. This feature is entirely within the `workflows` item (service facet subscribes/matches/fires; app facet configures/views; tool surface authors/reads triggers).
- An event trigger only functions while the service is running (it is the execution authority, per the baseline). Events published while the service is stopped are not consumed by the workflow (the handler is unregistered on stop); they remain durable in the event store but do not retroactively start runs on restart (no replay/catch-up triggering — see Q3 note if the user wants catch-up).
- A run started by a trigger is fire-and-poll (baseline FR-008): the trigger handler starts the run, obtains a `runId`, and returns promptly; it does not wait for the run to finish.
- Trigger configuration is per-workflow (a `triggers` array in the workflow JSON), not a separate global entity. Multiple triggers per workflow are allowed (FR-001); multiple workflows may trigger on the same event type (fan-out to several workflows is allowed).
- The triggering event's payload is available to the trigger handler at dispatch time (the event system delivers the event's type + payload to the matched handler); whether that payload is passed into the run as input/seed context is a Q2 decision.

## Open Questions

<!-- Surfaced at the specify step boundary; resolve during clarify. Max 3. -->
