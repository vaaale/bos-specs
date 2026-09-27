# Feature Specification: GitLab Integration (Webhook → Workflow)

**Feature Branch**: `002-gitlab-integration`

**Created**: 2026-09-27

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Add GitLab webhook support so that when a GitLab issue (or MR) is created, BOS is notified, runs the appropriate workflow to resolve it (bug: analyze → verify → propose fix; feature/improvement: draft a spec), and updates the item throughout. Settings → Integrations → GitLab must let you enable the webhook, show its URL, and generate a token — identical in shape to the Telegram webhook. The inbound call generates a BOS event; a toggleable service/handler listens and can be disabled so the user can route the event elsewhere (e.g. the Workflow app)."

**Revision (2026-09-27)**: Corrected per user review — (1) **no OAuth / no BOS-owned GitLab credential**; ALL GitLab read/write (reading the issue, posting comments, applying labels) is performed through the **GitLab MCP server** (configured in Settings → MCP Servers, with its own auth). The GitLab integration is therefore **webhook-only**. (2) Issue classification labels are fixed to three: **`Bug`**, **`Feature`**, **`Improvement`**. (3) The handler applies a defined set of **status labels** as it works through the pipeline. The "Connect a GitLab account" user story and all OAuth scope requirements are removed.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Enable the webhook and receive a new-item event (Priority: P1)

The user opens Settings → Integrations → GitLab and enables the webhook. The GitLab integration is **webhook-only** — there is no account-connect step and no credential to store. BOS displays the public webhook URL to paste into GitLab, and offers buttons to generate (and later rotate) a shared token. The user copies the URL and token into a GitLab project's webhook configuration ("URL" and "Secret token" fields). When a new issue (or MR) is opened in that project, GitLab POSTs to BOS, BOS verifies the token, and emits an event into the BOS event system.

**Why this priority**: This is the core of the feature — the inbound signal. It is independent of what *does* with the event; even with no handler enabled, the event must be produced and visible. There is no preceding "connect" step to build on.

**Independent Test**: With the webhook enabled and a generated token, register the URL + token against a GitLab project, open an issue, and confirm a `gitlab.issue.opened` event appears in the BOS event system (Event Viewer) carrying the issue's project id, iid, title, labels, author, and URL. A request with a wrong or missing token is rejected with 401; a request while the webhook is disabled is rejected with 404 (no information leak).

**Acceptance Scenarios**:

1. **Given** the GitLab integration is open and the webhook is disabled, **When** the user enables it, **Then** the public URL `…/api/integrations/webhooks/gitlab/<service>` is shown and a "Generate token" button is available.
2. **Given** the webhook is enabled, **When** the user clicks Generate token, **Then** a new secret token is created and displayed once (in a one-time reveal), and a "Rotate" action is available to replace it later.
3. **Given** a registered URL + token, **When** GitLab POSTs a well-formed new-issue event with a correct `X-Gitlab-Token`, **Then** BOS returns 2xx and emits one `gitlab.issue.opened` event with the issue payload.
4. **Given** a registered URL, **When** a POST arrives with a missing or incorrect token, **Then** BOS returns 401 and emits no event.
5. **Given** the webhook is disabled, **When** a POST arrives, **Then** BOS returns 404 (indistinguishable from a disabled or unknown webhook) and emits no event.
6. **Given** GitLab retries the same delivery, **When** an identical payload arrives again within the dedup window, **Then** BOS returns success but emits no duplicate event.
7. **Given** the panel is open, **When** the user has not yet configured a GitLab MCP server, **Then** the panel shows a hint that issue operations require a GitLab MCP server, with a link to Settings → MCP Servers.

---

### User Story 2 - Automatic triage and resolution of a new issue (Priority: P2)

A default, built-in headless handler is subscribed to new-issue events and is enabled by default. When a new issue arrives, the handler classifies it — using the issue's GitLab labels first (one of **`Bug`**, **`Feature`**, **`Improvement`**), falling back to reading the title and description — and runs the matching workflow. As it works, the handler updates the GitLab issue **through the GitLab MCP server**: it posts a comment stating the classification and applies a status label that reflects the current pipeline stage. For a **Bug** it analyzes and verifies the report and proposes a fix; for a **Feature** or **Improvement** it drafts a specification. This handler is a normal event handler: it has its own enable/disable toggle and can be turned off.

**Why this priority**: This is the value the user asked for — "BOS should go through the appropriate workflow to resolve the issue." It builds on US1 (the event to trigger on), which is why it is P2 behind it. It has no dependency on a connect step because GitLab operations go through the MCP server.

**Independent Test**: With US1 in place and a GitLab MCP server configured, create a new issue labeled `Bug` in a connected project. Observe that a `gitlab.issue.opened` event is consumed by the default handler, that the issue receives a classification comment and a status label, that the bug workflow runs, and that the issue is updated with the proposed fix and a final status label. Repeat with an issue labeled `Feature` and one labeled `Improvement`, confirming a spec is drafted in each case. Then disable the default handler and create another issue: the event is still emitted and visible, but no automatic processing occurs.

**Acceptance Scenarios**:

1. **Given** the default handler is enabled and a new issue labeled `Bug`, **When** the issue is opened, **Then** the handler comments its classification, applies a bug status label, runs the analyze → verify → propose-fix workflow, and posts the proposed fix with a `Fix Proposed` status label.
2. **Given** a new issue labeled `Feature`, **When** it is opened, **Then** the handler runs the feature-specification workflow and posts a draft specification with a `Spec Drafted` status label.
3. **Given** a new issue labeled `Improvement`, **When** it is opened, **Then** the handler runs the improvement-specification workflow (draft a spec to improve the existing feature) and posts it with a `Spec Drafted` status label.
4. **Given** a new issue with no classification label, **When** it is opened, **Then** the handler infers the type from the title and description, states the inferred type in a comment, and proceeds with the matching workflow.
5. **Given** the default handler is enabled, **When** the triage run fails partway (e.g. the agent errors), **Then** the event remains recorded in BOS, the issue receives a `Failed` status label and a failure comment; the failure does not crash the receiver or lose the event.
6. **Given** the default handler is disabled, **When** a new issue is opened, **Then** the `gitlab.issue.opened` event is still emitted and visible, but no automatic classification, labeling, or workflow is triggered.
7. **Given** no GitLab MCP server is configured, **When** a new issue is opened, **Then** the handler does not crash; it records the event and surfaces a "GitLab MCP not configured" failure (event + notification), and the issue is left unmodified.

---

### User Story 3 - Route the event elsewhere instead of the default handler (Priority: P2)

Because the inbound webhook only *emits* an event, and handlers subscribe independently, the user can disable the built-in triage handler and instead point another consumer — for example the Workflow app — at the same `gitlab.issue.opened` event. Enabled handler(s) act on the event according to their own toggles; the webhook and the handler are separate switches.

**Why this priority**: This is the flexibility the user explicitly called for ("the user can optionally use, for example, the workflow app to handle these events if he so desires"). It confirms the event/handler split is the real mechanism rather than a fixed pipeline.

**Independent Test**: Disable the default triage handler and enable a Workflow-app handler registered for `gitlab.issue.opened`. Open a new issue: confirm the default handler does not act, but the Workflow-app handler runs against the emitted event. Toggling either handler on/off takes effect for subsequent events without touching the webhook itself.

**Acceptance Scenarios**:

1. **Given** the default handler is disabled and a Workflow-app handler is enabled for `gitlab.issue.opened`, **When** a new issue is opened, **Then** the Workflow-app handler is the consumer that acts, and the built-in triage does not.
2. **Given** two handlers are enabled for the same event type, **When** a new issue is opened, **Then** both are notified for the event (per the event system's multi-handler semantics), each acting independently.
3. **Given** the webhook is enabled, **When** the user toggles a handler's enabled state, **Then** only that handler's participation changes — the webhook keeps receiving and emitting events unchanged.

---

### User Story 4 - Merge request events (Priority: P3)

In addition to issues, the webhook also accepts merge-request events from GitLab. A new (opened) MR produces a `gitlab.mr.opened` event carrying the MR's project id, iid, title, labels, author, source/target branches, and URL. Per the confirmed scope (Q1), the default triage handler performs **no** built-in workflow for MRs — triage + resolution is issue-only in v1. The MR event is emitted so the user can route it to another consumer (e.g. the Workflow app).

**Why this priority**: The user chose issue **+ MR** coverage, but the resolution workflow (bug/feature/improvement triage) is an issue concept; an MR is submitted work, not a request to resolve. MR event emission is confirmed scope; an MR-review workflow is a planned follow-up (Q1, Option B), so this story lands after the issue path is proven.

**Independent Test**: With US1 in place, open a merge request in a connected project and confirm a `gitlab.mr.opened` event is emitted with the MR payload. Verify the default triage handler performs no workflow on it (per Q1).

**Acceptance Scenarios**:

1. **Given** the webhook is enabled, **When** a merge request is opened, **Then** BOS emits a `gitlab.mr.opened` event with the MR's project id, iid, title, labels, author, branches, and URL.
2. **Given** a new MR, **When** the default triage handler is enabled, **Then** the handler performs no built-in workflow on the MR (issue-only triage, per Q1).

---

### Edge Cases

- **Webhook disabled vs unknown**: a disabled webhook and an unregistered one both return 404 — never 401 — so a probing caller cannot distinguish "disabled" from "doesn't exist."
- **Token rotation grace**: immediately after a token is rotated, deliveries carrying the *previous* token are still accepted for a short grace window so an in-flight GitLab retry is not dropped.
- **Retry / duplicate delivery**: GitLab retries on slow or failed responses; within the dedup window an identical re-delivery is acknowledged but not re-emitted.
- **No classification label**: an issue with no `Bug`/`Feature`/`Improvement` label is classified by the agent from title + description; the inferred type is stated in a comment.
- **Unrecognized label**: an issue labeled with something outside the three classification labels (e.g. `documentation`, `chore`) has its labels ignored for classification purposes and falls to the agent-based inference.
- **No GitLab MCP server configured**: the handler cannot read the issue or post updates; it must fail gracefully (record the event, surface a "not configured" failure) rather than crash or lose the event.
- **Non-issue/MR object kinds**: GitLab can send many other event types (push, pipeline, tag, …); the receiver must ignore `object_kind` values other than `issue` and `merge_request` for this feature (no event emitted, success response returned).
- **Agent's own updates (loop prevention)**: the handler's own MCP updates (commenting, relabeling) produce GitLab `note` and `update` events, **not** a new `issue`/`merge_request` with `action=open` — so they never re-trigger the handler.
- **Large description payloads**: the emitted event carries a bounded projection (id, iid, title, labels, author, url, and a truncated description) rather than the full raw payload, staying within the event system's size limits.
- **Agent run is long, HTTP response is short**: the receiver acknowledges the webhook quickly and runs the (multi-minute) workflow asynchronously afterward — it never blocks the HTTP response on the agent run.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST register a GitLab integration (id `gitlab`) with an integration card in Settings → Integrations → GitLab, using the generic integration framework. The integration is **webhook-only**: it stores no GitLab credential and has no account-connect step.
- **FR-002**: The GitLab integration card MUST expose the standard webhook control — an enable/disable action, the public webhook URL to paste into GitLab, and buttons to generate and to rotate a shared token — rendered with the same component/behavior as the existing Telegram webhook (the generic Webhook section).
- **FR-003**: When the webhook is enabled, the integration MUST accept inbound HTTP POSTs to its webhook URL and, for a request whose `X-Gitlab-Token` header matches the current (or grace-window previous) token, treat it as authentic.
- **FR-004**: The webhook MUST reject with 401 a request whose token does not match, and MUST reject with 404 (not 401) a request when the webhook is disabled, to avoid leaking existence/state.
- **FR-005**: For an authentic request, the webhook MUST emit an event into the BOS event system for each new (action `open`) issue (`gitlab.issue.opened`) and each new (action `open`) merge request (`gitlab.mr.opened`), carrying at least: project id, iid, title, labels, author, and URL.
- **FR-006**: The webhook MUST ignore (return success, emit nothing) requests whose `object_kind` is other than `issue` or `merge_request`, and MUST not treat actions other than `open` as new-item triggers.
- **FR-007**: The webhook MUST acknowledge the HTTP request promptly and run any downstream handler asynchronously, so a long-running workflow does not delay or fail the provider's delivery.
- **FR-008**: The webhook MUST deduplicate retried deliveries within a bounded window so a single logical event is not emitted more than once for byte-identical retries.
- **FR-009**: BOS MUST register a default headless handler for new-issue events (id distinct, e.g. `gitlab-issue-triage`), enabled by default, that is independently toggleable via the event system's handler enable/disable.
- **FR-010**: The default handler MUST classify a new issue by its GitLab labels first, recognizing exactly three classification labels — `Bug`, `Feature`, `Improvement` — and, when none is present, by reading the issue title and description, stating the inferred type in an issue comment.
- **FR-011**: The default handler MUST run the matching resolution workflow — `Bug`: analyze the report, verify it, and propose a fix; `Feature`: draft a specification for the requested new feature; `Improvement`: draft a specification to improve the existing feature.
- **FR-012**: The default handler MUST perform ALL GitLab reads and writes (reading the issue, posting comments, applying/removing labels) through the **GitLab MCP server**; it MUST NOT hold or use a BOS-owned GitLab credential.
- **FR-013**: The default handler MUST apply a status label to the issue that reflects the current pipeline stage, keeping exactly one status label active at a time, and MUST treat the classification label (`Bug`/`Feature`/`Improvement`) as additive and persistent. The status labels are: `Triage`, `Analyzing`, `Verifying`, `Fix Proposed` (Bug path); `Spec Drafted` (Feature/Improvement path); `Needs Input` (blocked, any path); `Failed` (any path).
- **FR-014**: The default handler MUST, on a failed or errored run, record the event, apply the `Failed` status label, and post a failure comment on the issue without crashing the receiver or losing the event.
- **FR-015**: When no GitLab MCP server is configured, the default handler MUST NOT crash; it MUST record the event and surface a "GitLab MCP not configured" failure (as an event/notification), leaving the issue unmodified. The GitLab integration panel MUST indicate whether a GitLab MCP server is configured and link to Settings → MCP Servers.
- **FR-016**: The user MUST be able to disable the default handler and, in its place, have another consumer (e.g. the Workflow app) handle the same emitted event, by toggling handlers independently of the webhook.
- **FR-017**: New merge requests MUST emit a `gitlab.mr.opened` event and receive no built-in workflow from the default handler (triage + resolution is issue-only in v1); the MR event exists for the user to route to another consumer (e.g. the Workflow app).
- **FR-018**: The system MUST prevent the handler's own GitLab updates (comments, labels) from re-triggering the resolution workflow in a loop.

### Key Entities *(include if feature involves data)*

- **GitLab Webhook Config**: per-integration webhook state (enabled flag, current + previous token, the public URL) — the inbound authentication + switch. The integration holds no account credential.
- **GitLab Event**: the BOS event emitted per new issue/MR (type `gitlab.issue.opened` / `gitlab.mr.opened`; payload: project id, iid, title, labels, author, url, truncated description).
- **Triage Handler**: the toggleable headless consumer of new-issue events (enabled flag owned by the event system; classification → workflow → status-labeling behavior).
- **GitLab MCP Server**: the external, user-configured MCP server (Settings → MCP Servers) through which ALL GitLab reads/writes occur; a prerequisite for the handler to act, with its own auth.
- **Classification Labels**: the three labels that determine the workflow — `Bug`, `Feature`, `Improvement`; additive and persistent on the issue.
- **Status Labels**: the transient, single-active labels the handler applies as it advances — `Triage`, `Analyzing`, `Verifying`, `Fix Proposed`, `Spec Drafted`, `Needs Input`, `Failed`.
- **GitLab Item**: the out-of-band issue or MR the workflow acts on (referenced by project id + iid; read/updated via the GitLab MCP server).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can go from a fresh BOS to a working GitLab→BOS→issue-update loop (configure a GitLab MCP server, enable the webhook, register URL + token, open a labeled issue, see the issue classified, relabeled, and updated by BOS) in under 10 minutes following the in-app instructions.
- **SC-002**: 100% of well-formed new-issue webhook deliveries with a valid token produce exactly one emitted event; 0% with an invalid token produce an event.
- **SC-003**: With the default handler enabled and a GitLab MCP server configured, ≥90% of newly opened issues that carry a `Bug`, `Feature`, or `Improvement` label are classified correctly and receive a substantive update (classification comment + correct status label + a workflow artifact: proposed fix or draft spec).
- **SC-004**: Disabling the default handler makes 100% of new-issue events still emit and remain visible while 0% are auto-processed, and a separately-enabled consumer (e.g. Workflow app) is able to act on the same event.
- **SC-005**: A token rotation results in 0 dropped deliveries during the grace window and old tokens being rejected after it.
- **SC-006**: No issue opened by or on behalf of the user's own account is processed more than once (no self-trigger loop) across the handler's repeated self-updates.
- **SC-007**: With no GitLab MCP server configured, 100% of new-issue events are still emitted and recorded, 0% crash the receiver, and a "GitLab MCP not configured" failure is surfaced.

## Assumptions

- The generic integration/webhook framework and the event kernel (emit, register handlers, enable/disable) already exist and are reused as-is; this feature adds a `gitlab` (webhook-only) integration, a `gitlab-issue-triage` headless handler, and the settings panel — it does not redesign those mechanisms.
- **No GitLab credential lives in BOS.** All GitLab read/write is delegated to the GitLab MCP server the user configures in Settings → MCP Servers. The GitLab integration therefore has no connect/OAuth step and no stored token beyond the webhook's own verification token.
- Webhook registration is manual (the user pastes the URL + token into GitLab's project webhook settings); automatic per-project `POST /projects/:id/hooks` registration is out of scope for v1.
- The webhook receiver's existing "not connected → 404" gate does not apply to a webhook-only integration; the gitlab integration is treated as available by default, and the webhook enable flag + valid token are the only inbound gates. (Design decision to confirm.)
- The primary resolution trigger is the `open` (new) action for both issues and MRs; close/reopen/update are accepted for completeness but do not start a resolution workflow.
- Classification relies on exactly three conventional labels (`Bug`, `Feature`, `Improvement`); any other label is ignored for classification and the agent-based fallback is used.
- The resolution workflows ("bug pipeline", "feature/improvement spec pipeline") map onto BOS's existing agent/spec capabilities; the handler orchestrates them and writes results back to GitLab via the MCP server — it does not redefine those workflows.
- The default triage handler's autonomy is bounded to commenting and relabeling (Q2, Option A): it proposes, and never closes the item or opens a merge request. Widening this boundary is a deliberate future change.
- A user who wants the handler to act must have configured at least one GitLab MCP server with access to the target project; that is a precondition, not something this feature installs.

## Clarifications

### Session 2026-09-27

- **Q1 (new MR handling)** → **A**: Emit a `gitlab.mr.opened` event only; no built-in MR workflow. Triage + resolution is issue-only in v1; the MR event is available for the user to route to another consumer. An MR-review workflow (Option B) is a planned follow-up.
- **Q2 (default-handler autonomy)** → **A**: Comment + label only. The handler proposes (fix or draft spec) and never closes the item or opens a merge request; a human finalizes. The boundary is recorded so it can be widened deliberately.

### Session 2026-09-27 (revision — user review of spec + mockup)

- **R1 (no OAuth)**: Remove the "Connect a GitLab account" story and all OAuth/scope requirements. The GitLab integration is webhook-only; it stores no GitLab credential. **All** GitLab read/write (read issue, comment, relabel) is done via the **GitLab MCP server** (Settings → MCP Servers). The handler must fail gracefully when no GitLab MCP server is configured (SC-007).
- **R2 (classification labels)**: Fixed to exactly three — `Bug`, `Feature`, `Improvement` (renamed from the earlier `bug` / `feature`/`enhancement`). `Improvement` = improve an existing feature; `Feature` = a new feature. Both draft a spec.
- **R3 (status labels)**: The handler applies these transient, single-active status labels as it advances — `Triage`, `Analyzing` (bug), `Verifying` (bug), `Fix Proposed` (bug), `Spec Drafted` (feature/improvement), `Needs Input` (blocked, any), `Failed` (any). The classification label is additive and persists. (Set proposed by Build Studio, accepted as the design default.)
