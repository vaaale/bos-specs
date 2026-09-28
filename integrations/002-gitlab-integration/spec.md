# Feature Specification: GitLab Integration (Webhook → Workflow)

**Feature Branch**: `002-gitlab-integration`

**Created**: 2026-09-27

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Add GitLab webhook support so that when a GitLab issue (or MR) is created, BOS is notified, runs the appropriate workflow to resolve it (bug: analyze → verify → propose fix; feature/improvement: draft a spec), and updates the item throughout. Settings → Integrations → GitLab must let you enable the webhook, show its URL, and generate a token — identical in shape to the Telegram webhook. The inbound call generates a BOS event; a toggleable service/handler listens and can be disabled so the user can route the event elsewhere (e.g. the Workflow app)."

**Revisions (2026-09-27)**:
- **R1**: No OAuth / no BOS-owned GitLab credential — all GitLab read/write is done via the **GitLab MCP server** (Settings → MCP Servers). The GitLab integration is **webhook-only**.
- **R2**: Classification labels fixed to three — `Bug`, `Feature`, `Improvement`.
- **R3/R5**: Handler-applied **status labels** defined; a terminal `Needs Review` label added (agent finished, human verifies, per Q2).
- **R4**: The GitLab MCP server is a dependency of the **handler/agent**, not the webhook — the integration panel has no MCP configuration surface.
- **R6 (handling model)**: The default handler is a **thin dispatcher**, not an intelligence. On a new issue it hands the issue context to a **configured agent** ("Gitlab Issue Agent", a normal agent in Settings → Agents). That agent's behavior is defined by a **skill** that is the state machine: read the issue's current (type, status) → classify → run the matching workflow → apply status labels → end at `Needs Review`. The panel lets the user select **which agent** handles new issues. There is no handler-internal step loop and no deterministic "is MCP configured" gate — the agent is the unit of capability, and if it cannot act the run fails gracefully. The self-healing feature is **fully separate** (no forwarding, no shared pipeline).
- **R7 (comment-driven re-entry)**: The pipeline is **driven from GitLab by commenting**, not only by issue creation. A new **human** comment on a triaged issue re-dispatches the agent so the process continues from GitLab alone (answering a `Needs Input` question, revising a `Needs Review` result, etc.). Loop prevention is **agent-side identity**: the agent determines its **own GitLab identity from the MCP server's token** (the project access token the user assigned) and acts only on new **human** notes — on its own notes (or no new human input) it posts nothing and changes no label, which self-terminates the chain. `Needs Review` is therefore **resumable** on a new human comment, not a hard stop. The handler stays thin (it learns no GitLab identity — R4 boundary intact).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Enable the webhook and receive a new-item event (Priority: P1)

The user opens Settings → Integrations → GitLab and enables the webhook. The GitLab integration is **webhook-only** — there is no account-connect step, no credential, and no MCP configuration in this panel. BOS displays the public webhook URL to paste into GitLab, and offers buttons to generate (and later rotate) a shared token. The user copies the URL and token into a GitLab project's webhook configuration ("URL" and "Secret token" fields). When a new issue (or MR) is opened in that project, GitLab POSTs to BOS, BOS verifies the token, and emits an event into the BOS event system.

**Why this priority**: This is the core of the feature — the inbound signal. It is independent of what *does* with the event; even with no handler enabled, the event must be produced and visible. There is no preceding "connect" step to build on, and no MCP-server dependency: the webhook only receives and emits. Whether a GitLab MCP server is configured (and assigned to the handling agent) is the agent's concern, not the webhook's.

**Independent Test**: With the webhook enabled and a generated token, register the URL + token against a GitLab project, open an issue, and confirm a `gitlab.issue.opened` event appears in the BOS event system (Event Viewer) carrying the issue's project id, iid, title, labels, author, and URL. A request with a wrong or missing token is rejected with 401; a request while the webhook is disabled is rejected with 404 (no information leak).

**Acceptance Scenarios**:

1. **Given** the GitLab integration is open and the webhook is disabled, **When** the user enables it, **Then** the public URL `…/api/integrations/webhooks/gitlab/<service>` is shown and a "Generate token" button is available.
2. **Given** the webhook is enabled, **When** the user clicks Generate token, **Then** a new secret token is created and displayed once (in a one-time reveal), and a "Rotate" action is available to replace it later.
3. **Given** a registered URL + token, **When** GitLab POSTs a well-formed new-issue event with a correct `X-Gitlab-Token`, **Then** BOS returns 2xx and emits one `gitlab.issue.opened` event with the issue payload.
4. **Given** a registered URL, **When** a POST arrives with a missing or incorrect token, **Then** BOS returns 401 and emits no event.
5. **Given** the webhook is disabled, **When** a POST arrives, **Then** BOS returns 404 (indistinguishable from a disabled or unknown webhook) and emits no event.
6. **Given** GitLab retries the same delivery, **When** an identical payload arrives again within the dedup window, **Then** BOS returns success but emits no duplicate event.
7. **Given** no GitLab MCP server is configured, **When** the user enables the webhook and registers the URL + token, **Then** the webhook still receives and emits events normally — the webhook has no MCP dependency; only the handling agent's ability to act is affected.

---

### User Story 2 - Automatic triage and resolution of a new issue (Priority: P2)

A default, built-in headless handler is subscribed to new-issue events and is enabled by default. It is a **thin dispatcher**: when a new issue arrives, it hands the issue context (project, iid, title, description, labels, url) to a **configured agent** — by default the **"Gitlab Issue Agent"**, a normal agent the user manages in Settings → Agents. The agent's behavior is defined by a **skill** that acts as the state machine: it reads the issue's current labels, classifies the issue by its type label (**`Bug`**, **`Feature`**, **`Improvement`**) with an agent-inference fallback, runs the matching workflow, applies a **status label** reflecting the current pipeline stage, and ends in the terminal **`Needs Review`** state. All GitLab reads and writes (reading the issue, posting comments, applying/removing labels) are performed by the agent **through the GitLab MCP server** configured on the agent. This handler is a normal event handler: it has its own enable/disable toggle and can be turned off.

**Why this priority**: This is the value the user asked for — "BOS should go through the appropriate workflow to resolve the issue." It builds on US1 (the event to trigger on), which is why it is P2. The handling logic lives in a user-configurable agent + skill rather than being hardcoded, so the user can change skills, MCP servers, or tools without touching the integration — the flexibility the user explicitly wanted.

**Independent Test**: With US1 in place, a GitLab MCP server configured on the Gitlab Issue Agent, create a new issue labeled `Bug` in a hooked project. Observe that a `gitlab.issue.opened` event is dispatched to the agent, that the issue receives a classification comment and status labels advancing through the bug pipeline, and that it ends at `Needs Review` with a proposed fix posted. Repeat with an issue labeled `Feature` and one labeled `Improvement`, confirming a spec is drafted in each case. Then disable the default handler and create another issue: the event is still emitted and visible, but no agent is dispatched.

**Acceptance Scenarios**:

1. **Given** the default handler is enabled and a new issue labeled `Bug`, **When** the issue is opened, **Then** the handler dispatches the issue to the configured agent, which comments its classification, applies a bug status label, runs the analyze → verify → propose-fix workflow, and ends the issue at the terminal `Needs Review` status with the proposed fix posted.
2. **Given** a new issue labeled `Feature`, **When** it is opened, **Then** the agent runs the feature-specification workflow and ends the issue at `Needs Review` with a draft specification posted.
3. **Given** a new issue labeled `Improvement`, **When** it is opened, **Then** the agent runs the improvement-specification workflow (draft a spec to improve the existing feature) and ends the issue at `Needs Review`.
4. **Given** a new issue with no classification label, **When** it is opened, **Then** the agent infers the type from the title and description, states the inferred type in a comment, and proceeds with the matching workflow.
5. **Given** the default handler is enabled, **When** the agent run fails partway (e.g. the agent errors or cannot reach GitLab), **Then** the event remains recorded in BOS, a failure is surfaced, and the issue receives a `Failed` status if the agent was able to reach it — the failure does not crash the receiver or lose the event.
6. **Given** the default handler is disabled, **When** a new issue is opened, **Then** the `gitlab.issue.opened` event is still emitted and visible, but no agent is dispatched.
7. **Given** the configured agent has no GitLab MCP server available, **When** a new issue is opened, **Then** the event is dispatched and recorded, the agent run fails gracefully with a "GitLab MCP not available" failure surfaced, and the issue is left unmodified.
8. **Given** the default handler is enabled, **When** the user changes which agent handles new issues (the agent picker), **Then** subsequent issues are dispatched to the newly selected agent.

---

### User Story 3 - Route the event elsewhere instead of the default handler (Priority: P2)

Because the inbound webhook only *emits* an event, and handlers subscribe independently, the user can disable the built-in triage handler and instead point another consumer — for example the Workflow app — at the same `gitlab.issue.opened` event. Enabled handler(s) act on the event according to their own toggles; the webhook and the handler are separate switches.

**Why this priority**: This is the flexibility the user explicitly called for ("the user can optionally use, for example, the workflow app to handle these events if he so desires"). It confirms the event/handler split is the real mechanism rather than a fixed pipeline.

**Independent Test**: Disable the default triage handler and enable a Workflow-app handler registered for `gitlab.issue.opened`. Open a new issue: confirm the default handler does not dispatch an agent, but the Workflow-app handler runs against the emitted event. Toggling either handler on/off takes effect for subsequent events without touching the webhook itself.

**Acceptance Scenarios**:

1. **Given** the default handler is disabled and a Workflow-app handler is enabled for `gitlab.issue.opened`, **When** a new issue is opened, **Then** the Workflow-app handler is the consumer that acts, and the built-in dispatcher does not.
2. **Given** two handlers are enabled for the same event type, **When** a new issue is opened, **Then** both are notified for the event (per the event system's multi-handler semantics), each acting independently.
3. **Given** the webhook is enabled, **When** the user toggles a handler's enabled state, **Then** only that handler's participation changes — the webhook keeps receiving and emitting events unchanged.

---

### User Story 4 - Merge request events (Priority: P3)

In addition to issues, the webhook also accepts merge-request events from GitLab. A new (opened) MR produces a `gitlab.mr.opened` event carrying the MR's project id, iid, title, labels, author, source/target branches, and URL. Per the confirmed scope (Q1), the default triage handler performs **no** built-in workflow for MRs — it is registered for new-issue events only. The MR event is emitted so the user can route it to another consumer (e.g. the Workflow app).

**Why this priority**: The user chose issue **+ MR** coverage, but the resolution workflow (bug/feature/improvement triage) is an issue concept; an MR is submitted work, not a request to resolve. MR event emission is confirmed scope; an MR-review workflow is a planned follow-up (Q1, Option B), so this story lands after the issue path is proven.

**Independent Test**: With US1 in place, open a merge request in a hooked project and confirm a `gitlab.mr.opened` event is emitted with the MR payload. Verify the default triage handler performs no action on it (per Q1).

**Acceptance Scenarios**:

1. **Given** the webhook is enabled, **When** a merge request is opened, **Then** BOS emits a `gitlab.mr.opened` event with the MR's project id, iid, title, labels, author, branches, and URL.
2. **Given** a new MR, **When** the default triage handler is enabled, **Then** the handler does not dispatch an agent for the MR (issue-only, per Q1).

---

### Edge Cases

- **Webhook disabled vs unknown**: a disabled webhook and an unregistered one both return 404 — never 401 — so a probing caller cannot distinguish "disabled" from "doesn't exist."
- **Token rotation grace**: immediately after a token is rotated, deliveries carrying the *previous* token are still accepted for a short grace window so an in-flight GitLab retry is not dropped.
- **Retry / duplicate delivery**: GitLab retries on slow or failed responses; within the dedup window an identical re-delivery is acknowledged but not re-emitted.
- **No classification label**: an issue with no `Bug`/`Feature`/`Improvement` label is classified by the agent from title + description; the inferred type is stated in a comment.
- **Unrecognized label**: an issue labeled with something outside the three classification labels (e.g. `documentation`, `chore`) has its labels ignored for classification purposes and falls to the agent-based inference.
- **No GitLab MCP server available to the agent**: the agent cannot read the issue or post updates; the run fails gracefully (event recorded, failure surfaced) rather than crashing the receiver or losing the event. The handler does not pre-check this — it is the agent's capability to determine.
- **Selected agent does not exist / misconfigured**: if the panel's agent picker points at an agent that is missing or cannot run, the dispatch fails gracefully (event recorded, failure surfaced) rather than crashing.
- **Non-issue/MR/note object kinds**: GitLab can send many other event types (push, pipeline, tag, …); the receiver must ignore `object_kind` values other than `issue`, `merge_request`, and `note` for this feature (no event emitted, success response returned). Note hooks are re-dispatched only when the target is an Issue (`noteable_type === "Issue"`).
- **Agent's own updates (loop prevention)**: the agent's own GitLab updates (relabeling, state changes via MCP) produce GitLab `issue`/`merge_request` events with `action:"update"` — excluded by the receiver's `action === "open"` filter. The agent's own **comments** produce `note` hooks, which ARE re-dispatched (R7), but the agent is self-terminating on its own notes (it recognizes its own GitLab identity from the MCP token and posts nothing when the latest note is its own).
- **Large description payloads**: the emitted event carries a bounded projection (id, iid, title, labels, author, url, and a truncated description) rather than the full raw payload, staying within the event system's size limits.
- **Agent run is long, HTTP response is short**: the receiver acknowledges the webhook quickly, and the handler dispatches the agent asynchronously afterward — it never blocks the HTTP response on the agent run.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST register a GitLab integration (id `gitlab`) with an integration card in Settings → Integrations → GitLab, using the generic integration framework. The integration is **webhook-only**: it stores no GitLab credential, has no account-connect step, and has no MCP-configuration surface in its panel.
- **FR-002**: The GitLab integration card MUST expose the standard webhook control — an enable/disable action, the public webhook URL to paste into GitLab, and buttons to generate and to rotate a shared token — rendered with the same component/behavior as the existing Telegram webhook (the generic Webhook section).
- **FR-003**: When the webhook is enabled, the integration MUST accept inbound HTTP POSTs to its webhook URL and, for a request whose `X-Gitlab-Token` header matches the current (or grace-window previous) token, treat it as authentic.
- **FR-004**: The webhook MUST reject with 401 a request whose token does not match, and MUST reject with 404 (not 401) a request when the webhook is disabled, to avoid leaking existence/state.
- **FR-005**: For an authentic request, the webhook MUST emit an event into the BOS event system for each new (action `open`) issue (`gitlab.issue.opened`) and each new (action `open`) merge request (`gitlab.mr.opened`), carrying at least: project id, iid, title, labels, author, and URL.
- **FR-022**: The webhook MUST also emit a `gitlab.issue.noted` event for each new comment (`object_kind: "note"`, target `noteable_type: "Issue"`), carrying at least: project id, issue iid, the note text, and the note author's username. (R7 — comment-driven re-entry.)
- **FR-006**: The webhook MUST ignore (return success, emit nothing) requests whose `object_kind` is other than `issue` or `merge_request`, and MUST not treat actions other than `open` as new-item triggers.
- **FR-007**: The webhook MUST acknowledge the HTTP request promptly and run any downstream handler asynchronously, so a long-running workflow does not delay or fail the provider's delivery.
- **FR-008**: The webhook MUST deduplicate retried deliveries within a bounded window so a single logical event is not emitted more than once for byte-identical retries.
- **FR-009**: BOS MUST register a default headless handler for `gitlab.issue.*` events (id distinct, e.g. `gitlab-issue-triage`), enabled by default, that is independently toggleable via the event system's handler enable/disable. This handler is a **thin dispatcher**: it forwards the issue context to the configured agent and does not itself contain the triage logic. On `gitlab.issue.opened` it triggers a fresh pipeline run; on `gitlab.issue.noted` it triggers a resume run (R7).
- **FR-010**: BOS MUST provide a **"Gitlab Issue Agent"** — a normal, user-manageable agent (Settings → Agents) — and a **skill** that defines the issue-triage state machine. The agent and its skill MUST be user-configurable (skills, tools, MCP servers, system prompt) like any other agent; the integration MUST NOT hardcode a bespoke, non-configurable agent or a bespoke step pipeline.
- **FR-011**: The configured agent's skill MUST classify a new issue by its GitLab labels first, recognizing exactly three classification labels — `Bug`, `Feature`, `Improvement` — and, when none is present, by reading the issue title and description, stating the inferred type in an issue comment.
- **FR-012**: The agent MUST run the matching resolution workflow — `Bug`: analyze the report, verify it, and propose a fix; `Feature`: draft a specification for the requested new feature; `Improvement`: draft a specification to improve the existing feature.
- **FR-013**: All GitLab reads and writes (reading the issue, posting comments, applying/removing labels) MUST be performed by the agent **through the GitLab MCP server** configured on the agent; the integration and the handler MUST NOT hold or use a BOS-owned GitLab credential, and the handler MUST NOT pre-check or gate on MCP-server configuration (the agent is the unit of capability).
- **FR-014**: The agent MUST apply a status label to the issue that reflects the current pipeline stage, keeping exactly one status label active at a time, and MUST treat the classification label (`Bug`/`Feature`/`Improvement`) as additive and persistent. The status labels are: `Triage`, `Analyzing`, `Verifying`, `Fix Proposed` (Bug path); `Spec Drafted` (Feature/Improvement path); `Needs Review` (terminal — the agent has finished its deliverable and is requesting that a human verify the design / fix / UI mockup, any path); `Needs Input` (blocked, any path); `Failed` (any path). `Needs Review` is the terminal human-facing state: it follows `Fix Proposed` (Bug) or `Spec Drafted` (Feature/Improvement) once the agent has produced its deliverable, and per Q2 the human then verifies and finalizes (closes the issue / opens the MR).
- **FR-015**: The agent MUST, on a failed or errored run, surface the failure (event/notification) and apply the `Failed` status label if it is able to reach the issue, without crashing the receiver or losing the event.
- **FR-016**: When the configured agent cannot act because no GitLab MCP server is available (or the selected agent is missing/misconfigured), the dispatch MUST NOT crash the receiver; the event MUST be recorded, a failure surfaced, and the issue left unmodified. The GitLab integration panel MUST NOT display or require MCP-server configuration (MCP is the agent's dependency, R4).
- **FR-017**: Settings → Integrations → GitLab MUST let the user select **which agent** handles new issues (a "Workflow configuration" / agent picker), defaulting to the Gitlab Issue Agent.
- **FR-018**: The user MUST be able to disable the default handler and, in its place, have another consumer (e.g. the Workflow app) handle the same emitted event, by toggling handlers independently of the webhook.
- **FR-019**: New merge requests MUST emit a `gitlab.mr.opened` event and receive no built-in workflow from the default handler (triage + resolution is issue-only in v1); the MR event exists for the user to route to another consumer (e.g. the Workflow app).
- **FR-020**: The system MUST prevent the agent's own GitLab updates (comments, labels) from re-triggering the resolution workflow in a loop. Structural prevention: the receiver's `object_kind ∈ {issue, merge_request}` + `action === "open"` filter excludes the agent's own issue edits (labels, state). For note re-entry (R7), agent-side identity: the agent determines its own GitLab identity from the MCP server's token and acts only on new **human** notes — on its own notes (or no new human input) it posts nothing and changes no label, self-terminating the chain.
- **FR-021**: A new issue MUST be handled by a single agent run that performs the whole pipeline (classify → workflow → `Needs Review`) end-to-end. A subsequent **human** comment on the issue MUST re-dispatch the agent (R7), which resumes from the issue's current status — the `Needs Input` and `Needs Review` states are both resumable on a new human note. The agent MUST be self-terminating on its own notes (FR-020).

### Key Entities *(include if feature involves data)*

- **GitLab Webhook Config**: per-integration webhook state (enabled flag, current + previous token, the public URL) — the inbound authentication + switch. The integration holds no account credential.
- **GitLab Event**: the BOS event emitted per new issue/MR (type `gitlab.issue.opened` / `gitlab.mr.opened`; payload: project id, iid, title, labels, author, url, truncated description).
- **Issue-triage Handler (dispatcher)**: the toggleable core headless consumer of new-issue events (enabled flag owned by the event system). Thin: it forwards the issue context to the configured agent; it holds no triage logic.
- **Gitlab Issue Agent**: a normal, user-configured agent (Settings → Agents) with the GitLab MCP server in its allowlist and the triage skill. The unit of capability — it classifies, runs the workflow, and updates the issue.
- **Triage Skill**: the skill that is the state machine — reads the issue's current (type, status) and drives classification, the matching workflow, status-label transitions, and the terminal `Needs Review` state.
- **GitLab MCP Server**: the external, user-configured MCP server (Settings → MCP Servers) through which ALL GitLab reads/writes occur; a dependency of the agent (not the webhook), with its own auth.
- **Classification Labels**: the three labels that determine the workflow — `Bug`, `Feature`, `Improvement`; additive and persistent on the issue.
- **Status Labels**: the transient, single-active labels the agent applies as it advances — `Triage`, `Analyzing`, `Verifying`, `Fix Proposed`, `Spec Drafted`, `Needs Review`, `Needs Input`, `Failed`.
- **GitLab Item**: the out-of-band issue or MR the agent acts on (referenced by project id + iid; read/updated via the GitLab MCP server).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can go from a fresh BOS to a working GitLab→BOS→issue-update loop (configure a GitLab MCP server on the Gitlab Issue Agent, enable the webhook, register URL + token, open a labeled issue, see the issue classified, relabeled, and updated by BOS) in under 10 minutes following the in-app instructions.
- **SC-002**: 100% of well-formed new-issue webhook deliveries with a valid token produce exactly one emitted event; 0% with an invalid token produce an event.
- **SC-003**: With the default handler enabled and a GitLab MCP server configured on the agent, ≥90% of newly opened issues that carry a `Bug`, `Feature`, or `Improvement` label are classified correctly and receive a substantive update (classification comment + correct status label + a workflow artifact: proposed fix or draft spec), ending at `Needs Review`.
- **SC-004**: Disabling the default handler makes 100% of new-issue events still emit and remain visible while 0% dispatch an agent, and a separately-enabled consumer (e.g. Workflow app) is able to act on the same event.
- **SC-005**: A token rotation results in 0 dropped deliveries during the grace window and old tokens being rejected after it.
- **SC-006**: No issue opened by or on behalf of the user's own account is processed more than once (no self-trigger loop) across the agent's repeated self-updates.
- **SC-007**: With no GitLab MCP server available to the agent, 100% of new-issue events are still emitted and recorded, 0% crash the receiver, and a graceful failure is surfaced with the issue unmodified.
- **SC-008**: Changing the agent picker to a different valid agent results in subsequent new issues being dispatched to that agent (the handling is agent-driven, not hardcoded).

## Assumptions

- The generic integration/webhook framework and the event kernel (emit, register handlers, enable/disable) already exist and are reused as-is; this feature adds a `gitlab` (webhook-only) integration, a thin `gitlab-issue-triage` dispatcher handler, a seed `Gitlab Issue Agent` + triage skill, and the settings panel — it does not redesign those mechanisms.
- **No GitLab credential lives in BOS.** All GitLab read/write is delegated to the GitLab MCP server configured on the handling agent. The GitLab integration therefore has no connect/OAuth step and no stored token beyond the webhook's own verification token.
- **Handling is agent + skill, not a bespoke pipeline.** The triage state machine lives in a user-editable skill on a normal agent (Settings → Agents), so its behavior (and the workflows it runs) are extensible without changing the integration. This is a deliberate contrast with the self-healing feature, which remains fully separate (R6).
- **One run per trigger** (FR-021, R7): a new issue triggers one agent run that does the whole pipeline; a new human comment on the issue triggers a resume run. The agent determines its own GitLab identity from the MCP server's token and is self-terminating on its own notes. (This supersedes the earlier v1 "single run, re-entry is future" boundary.)
- **Trajectory visibility**: the agent run is a headless run; its full tool/reasoning trajectory is captured in the run transcript (viewable in the Workflow Manager trajectory surface). It is **not** attached to an Assistant conversation (out of scope per user).
- Webhook registration is manual (the user pastes the URL + token into GitLab's project webhook settings); automatic per-project `POST /projects/:id/hooks` registration is out of scope for v1.
- The primary resolution trigger is the `open` (new) action for both issues and MRs; close/reopen/update are accepted for completeness but do not start a resolution workflow.
- Classification relies on exactly three conventional labels (`Bug`, `Feature`, `Improvement`); any other label is ignored for classification and the agent-based fallback is used.
- The resolution workflows ("bug pipeline", "feature/improvement spec pipeline") map onto the agent's capabilities (its skills/tools/MCP); the skill orchestrates them and writes results back to GitLab — it does not redefine a separate engine.
- The default triage handler's autonomy is bounded to commenting and relabeling (Q2, Option A): it proposes, and never closes the item or opens a merge request. Widening this boundary is a deliberate future change.

## Clarifications

### Session 2026-09-27

- **Q1 (new MR handling)** → **A**: Emit a `gitlab.mr.opened` event only; no built-in MR workflow. Triage + resolution is issue-only in v1; the MR event is available for the user to route to another consumer. An MR-review workflow (Option B) is a planned follow-up.
- **Q2 (default-handler autonomy)** → **A**: Comment + label only. The agent proposes (fix or draft spec) and never closes the item or opens a merge request; a human finalizes. The boundary is recorded so it can be widened deliberately.

### Session 2026-09-27 (revision 1 — user review of spec + mockup)

- **R1 (no OAuth)**: The GitLab integration is webhook-only; it stores no GitLab credential. **All** GitLab read/write is done via the **GitLab MCP server** (Settings → MCP Servers). The handler/agent must fail gracefully when no MCP server is available (SC-007).
- **R2 (classification labels)**: Fixed to exactly three — `Bug`, `Feature`, `Improvement`. `Improvement` = improve an existing feature; `Feature` = a new feature. Both draft a spec.
- **R3/R5 (status labels)**: The agent applies these transient, single-active status labels as it advances — `Triage`, `Analyzing` (bug), `Verifying` (bug), `Fix Proposed` (bug), `Spec Drafted` (feature/improvement), `Needs Review` (terminal — agent finished, human verifies), `Needs Input` (blocked, any), `Failed` (any). The classification label is additive and persists.
- **R4 (MCP is a handler/agent dependency, not a webhook one)**: The GitLab MCP server is required by the **agent** that acts on the event — configured on the agent (Settings → Agents → MCP), with its own auth. The webhook and the integration panel have **no** MCP dependency and MUST NOT display or require MCP-server configuration.

### Session 2026-09-27 (revision 2 — handling model, R6)

- **R6 (agent + skill, thin dispatcher)**: The default handler is a **thin dispatcher** (FR-009) that forwards the issue context to a **configured agent** — the **"Gitlab Issue Agent"** (FR-010), a normal agent in Settings → Agents — whose behavior is defined by a **skill** that is the state machine (classify → workflow → status labels → `Needs Review`). The panel adds a **Workflow configuration / agent picker** (FR-017) defaulting to the Gitlab Issue Agent. There is **no handler-internal step loop** and **no deterministic "is MCP configured" gate** (FR-013) — the agent is the unit of capability; when it cannot act, the run fails gracefully (FR-016, SC-007). **Self-heal is fully separate** (no forwarding, no shared pipeline). **One run per issue** does the whole pipeline in v1 (FR-021). **Trajectory** is the headless run transcript (Workflow Manager surface); **no Assistant app attachment**.
