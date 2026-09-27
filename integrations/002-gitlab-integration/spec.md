# Feature Specification: GitLab Integration (Webhook → Workflow)

**Feature Branch**: `002-gitlab-integration`

**Created**: 2026-09-27

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Add GitLab webhook support so that when a GitLab issue (or MR) is created, BOS is notified, runs the appropriate workflow to resolve it (bug: analyze → verify → propose fix; feature: spec → build), and updates the item throughout. Settings → Integrations → GitLab must let you enable the webhook, show its URL, and generate a token — identical in shape to the Telegram webhook. The inbound call generates a BOS event; a toggleable service/handler listens and can be disabled so the user can route the event elsewhere (e.g. the Workflow app)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Connect a GitLab account (Priority: P1)

The user connects their GitLab (gitlab.com or a self-hosted instance) to BOS in Settings → Integrations → GitLab. This establishes the credentials BOS uses to act on issues later (comment, relabel, close). Connection follows the existing GitLab OAuth flow already used for git remotes, but is surfaced as a first-class integration card with its own scope grant.

**Why this priority**: Everything downstream depends on it — the webhook receiver refuses to process events while the integration is not connected, and the resolution workflow needs API credentials to update the issue. Without this there is no functional system.

**Independent Test**: Open Settings → Integrations → GitLab, click Connect, complete OAuth with the `api` scope, and confirm the card shows "Connected" with the instance URL. Disconnect and confirm the credential is cleared. This delivers a usable, re-usable outbound channel.

**Acceptance Scenarios**:

1. **Given** the GitLab card is not connected, **When** the user clicks Connect and completes OAuth, **Then** the card shows Connected, the granted scopes include `api`, and the instance URL (gitlab.com default or self-hosted) is recorded.
2. **Given** a connected account, **When** the user clicks Disconnect, **Then** the credential is cleared, the card returns to Not connected, and any enabled webhook stops being processed (inbound calls return not-connected).
3. **Given** the card is already connected, **When** the user reconnects (re-authorizes), **Then** the previous credential is replaced and the connection stays healthy.

---

### User Story 2 - Enable the webhook and receive a new-issue event (Priority: P1)

The user enables the GitLab webhook. BOS displays the public webhook URL to paste into GitLab, and offers a button to generate (and later rotate) a shared token. The user copies the URL and token into a GitLab project's webhook configuration ("Secret token" field). When an issue is opened in that project, GitLab POSTs to BOS, BOS verifies the token, and emits an event into the BOS event system.

**Why this priority**: This is the core of the feature — the inbound signal. It is independent of what *does* with the event; even with no handler enabled, the event must be produced and visible.

**Independent Test**: With the account connected and webhook enabled + a generated token, register the URL/token against a GitLab project, open an issue, and confirm a `gitlab.issue.opened` event appears in the BOS event system (Event Viewer) carrying the issue's project id, iid, title, labels, author, and URL. A request with a wrong or missing token is rejected with 401; a request while the webhook is disabled is rejected with 404 (no information leak).

**Acceptance Scenarios**:

1. **Given** the account is connected and the webhook is disabled, **When** the user enables it, **Then** the public URL `…/api/integrations/webhooks/gitlab/webhooks` is shown and a "Generate token" button is available.
2. **Given** the webhook is enabled, **When** the user clicks Generate token, **Then** a new secret token is created and displayed once, and a "Rotate" action is available to replace it later.
3. **Given** a registered URL + token, **When** GitLab POSTs a well-formed issue event with a correct `X-Gitlab-Token`, **Then** BOS returns 2xx and emits one `gitlab.issue.opened` event with the issue payload.
4. **Given** a registered URL, **When** a POST arrives with a missing or incorrect token, **Then** BOS returns 401 and emits no event.
5. **Given** the webhook is disabled, **When** a POST arrives, **Then** BOS returns 404 (indistinguishable from a disabled or unknown webhook) and emits no event.
6. **Given** the integration is connected but a POST arrives while not connected (e.g. mid-disconnect), **When** the receiver checks connection state, **Then** it returns 404 not-connected and emits no event.
7. **Given** GitLab retries the same delivery, **When** an identical payload arrives again within the dedup window, **Then** BOS returns success but emits no duplicate event.

---

### User Story 3 - Automatic triage and resolution of a new issue (Priority: P2)

A default, built-in headless handler is subscribed to new-issue events and is enabled by default. When a new issue arrives, the handler classifies it — using GitLab labels first (`bug`, `feature`/`enhancement`), falling back to reading the title and description — and runs the matching workflow: for a bug, analyze and verify the report and propose a fix; for a feature request, produce a specification for it. Throughout the run the handler updates the GitLab item (comments with progress, labels to mark state, and a final proposal). This handler is a normal event handler: it has its own enable/disable toggle and can be turned off.

**Why this priority**: This is the value the user asked for — "BOS should go through the appropriate workflow to resolve the issue." It builds on US1 (credentials to update the issue) and US2 (the event to trigger on), which is why it is P2 behind them.

**Independent Test**: With US1+US2 in place, create a new issue labeled `bug` in a connected project. Observe that a `gitlab.issue.opened` event is consumed by the default handler, that the issue receives a classification comment, that the bug workflow runs, and that the issue is updated with the proposed fix and a status label. Then disable the default handler and create another issue: the event is still emitted and visible, but no automatic processing occurs.

**Acceptance Scenarios**:

1. **Given** the default handler is enabled and a new issue labeled `bug`, **When** the issue is opened, **Then** the handler comments its classification on the issue, runs the bug-resolution workflow, and posts the proposed fix with a status label.
2. **Given** a new issue labeled `feature` or `enhancement`, **When** it is opened, **Then** the handler runs the feature-specification workflow and posts the draft specification on the issue.
3. **Given** a new issue with no classification label, **When** it is opened, **Then** the handler infers the type from the title and description, states the inferred type in a comment, and proceeds with the matching workflow.
4. **Given** the default handler is enabled, **When** the triage run fails partway (e.g. the agent errors), **Then** the event remains recorded in BOS and the issue receives a comment noting the failure; the failure does not crash the receiver or lose the event.
5. **Given** the default handler is disabled, **When** a new issue is opened, **Then** the `gitlab.issue.opened` event is still emitted and visible, but no automatic classification or workflow is triggered.

---

### User Story 4 - Route the event elsewhere instead of the default handler (Priority: P2)

Because the inbound webhook only *emits* an event, and handlers subscribe independently, the user can disable the built-in triage handler and instead point another consumer — for example the Workflow app — at the same `gitlab.issue.opened` event. Exactly one (or more) enabled handler(s) act on the event according to their own toggles; the webhook and the handler are separate switches.

**Why this priority**: This is the flexibility the user explicitly called for ("the user can optionally use, for example, the workflow app to handle these events if he so desires"). It confirms the event/handler split is the real mechanism rather than a fixed pipeline.

**Independent Test**: Disable the default triage handler and enable a Workflow-app handler registered for `gitlab.issue.opened`. Open a new issue: confirm the default handler does not act, but the Workflow-app handler runs against the emitted event. Toggling either handler on/off takes effect for subsequent events without touching the webhook itself.

**Acceptance Scenarios**:

1. **Given** the default handler is disabled and a Workflow-app handler is enabled for `gitlab.issue.opened`, **When** a new issue is opened, **Then** the Workflow-app handler is the consumer that acts, and the built-in triage does not.
2. **Given** two handlers are enabled for the same event type, **When** a new issue is opened, **Then** both are notified for the event (per the event system's multi-handler semantics), each acting independently.
3. **Given** the webhook is enabled, **When** the user toggles a handler's enabled state, **Then** only that handler's participation changes — the webhook keeps receiving and emitting events unchanged.

---

### User Story 5 - Merge request events (Priority: P3)

In addition to issues, the webhook also accepts merge-request events from GitLab. A new (opened) MR produces a `gitlab.mr.opened` event carrying the MR's project id, iid, title, labels, author, source/target branches, and URL, so the same handler-routing model applies to MRs.

**Why this priority**: The user chose issue **+ MR** coverage, but the resolution workflow (bug/feature triage) is primarily an issue concept; MR handling is the extension and is least certain, so it lands after the issue path is proven.

**Independent Test**: With US2 in place, open a merge request in a connected project and confirm a `gitlab.mr.opened` event is emitted with the MR payload. Verify which workflow (if any) an enabled handler runs for a new MR.

**Acceptance Scenarios**:

1. **Given** the webhook is enabled and the account connected, **When** a merge request is opened, **Then** BOS emits a `gitlab.mr.opened` event with the MR's project id, iid, title, labels, author, branches, and URL.
2. **Given** a new MR, **When** the default triage handler is enabled, **Then** the handler applies the behavior decided in Clarification Q1 (see below) to the MR.

---

### Edge Cases

- **Webhook disabled vs unknown**: a disabled webhook and an unregistered one both return 404 — never 401 — so a probing caller cannot distinguish "disabled" from "doesn't exist."
- **Token rotation grace**: immediately after a token is rotated, deliveries signed with the *previous* token are still accepted for a short grace window so an in-flight GitLab retry is not dropped.
- **Retry / duplicate delivery**: GitLab retries on slow or failed responses; within the dedup window an identical re-delivery is acknowledged but not re-emitted.
- **No classification label**: an issue with no `bug`/`feature` label is classified by the agent from title + description; the inferred type is stated in a comment.
- **Unconnected integration**: while disconnected, inbound webhooks are refused (404 not-connected) and no events are emitted, even if a token is still valid.
- **Non-issue/MR object kinds**: GitLab can send many other event types (push, pipeline, tag, …); the receiver must ignore `object_kind` values other than `issue` and `merge_request` for this feature (no event emitted, success response returned).
- **Large description payloads**: the emitted event carries a bounded projection (id, iid, title, labels, author, url, and a truncated description) rather than the full raw payload, staying within the event system's size limits.
- **Agent run is long, HTTP response is short**: the receiver acknowledges the webhook quickly and runs the (multi-minute) workflow asynchronously afterward — it never blocks the HTTP response on the agent run.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST register a GitLab integration (id `gitlab`) with an integration card in Settings → Integrations → GitLab, distinct from the existing GitLab OAuth git-remote provider, using the generic integration framework.
- **FR-002**: The GitLab integration MUST support connecting a GitLab account (gitlab.com or a self-hosted instance) via OAuth granting at least the `api` scope, and disconnecting it.
- **FR-003**: The GitLab integration card MUST expose a webhook control with: an enable/disable toggle, the public webhook URL to paste into GitLab, and a button to generate and a button to rotate a shared token.
- **FR-004**: When the webhook is enabled, the integration MUST accept inbound HTTP POSTs to its webhook URL and, for a request whose `X-Gitlab-Token` header matches the current (or grace-window previous) token, treat it as authentic.
- **FR-005**: The webhook MUST reject with 401 a request whose token does not match, and MUST reject with 404 (not 401) a request when the webhook is disabled or the integration is not connected, to avoid leaking existence/state.
- **FR-006**: For an authentic request, the webhook MUST emit an event into the BOS event system for each new (action `open`) issue (`gitlab.issue.opened`) and each new (action `open`) merge request (`gitlab.mr.opened`), carrying at least: project id, iid, title, labels, author, and URL.
- **FR-007**: The webhook MUST ignore (return success, emit nothing) requests whose `object_kind` is other than `issue` or `merge_request`, and actions other than `open` are at minimum not treated as new-item triggers.
- **FR-008**: The webhook MUST acknowledge the HTTP request promptly and run any downstream handler asynchronously, so a long-running workflow does not delay or fail the provider's delivery.
- **FR-009**: The webhook MUST deduplicate retried deliveries within a bounded window so a single logical event is not emitted more than once for byte-identical retries.
- **FR-010**: BOS MUST register a default headless handler for new-issue events (id distinct, e.g. `gitlab-issue-triage`), enabled by default, that is independently toggleable via the event system's handler enable/disable.
- **FR-011**: The default handler MUST classify a new issue by GitLab labels first (`bug`, `feature`/`enhancement`) and, when no such label is present, by reading the issue title and description, stating the inferred type in an issue comment.
- **FR-012**: The default handler MUST run the matching resolution workflow — bug: analyze the report, verify it, and propose a fix; feature: produce a specification for the requested feature — and MUST update the GitLab issue during the run with progress comments, state labels, and a final proposal.
- **FR-013**: The default handler MUST, on a failed or errored run, record the event and post a failure comment on the issue without crashing the receiver or losing the event.
- **FR-014**: The user MUST be able to disable the default handler and, in its place, have another consumer (e.g. the Workflow app) handle the same emitted event, by toggling handlers independently of the webhook.
- **FR-015**: New merge requests MUST be processed by the default handler according to Clarification Q1.
- **FR-016**: The system MUST prevent the agent's own GitLab updates (comments, labels) from re-triggering the resolution workflow in a loop.

### Key Entities *(include if feature involves data)*

- **GitLab Integration**: the connected GitLab account/instance (instance URL, connection state, granted scopes) — the outbound credential source.
- **GitLab Webhook Config**: per-integration webhook state (enabled flag, current + previous token, the public URL) — the inbound authentication + switch.
- **GitLab Event**: the BOS event emitted per new issue/MR (type `gitlab.issue.opened` / `gitlab.mr.opened`; payload: project id, iid, title, labels, author, url, truncated description).
- **Triage Handler**: the toggleable headless consumer of new-issue events (enabled flag owned by the event system; classification → workflow → issue-updates behavior).
- **GitLab Item**: the out-of-band issue or MR the workflow acts on (referenced by project id + iid; updated via the GitLab API: comments, labels, state).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can go from a fresh BOS to a working GitLab→BOS→issue-update loop (connect, enable webhook, register URL+token, open an issue, see the issue updated by BOS) in under 10 minutes following the in-app instructions.
- **SC-002**: 100% of well-formed new-issue webhook deliveries with a valid token produce exactly one emitted event; 0% with an invalid token produce an event.
- **SC-003**: With the default handler enabled, ≥90% of newly opened issues that carry a `bug` or `feature`/`enhancement` label are classified correctly and receive a substantive update (classification comment + a workflow artifact: proposed fix or draft spec).
- **SC-004**: Disabling the default handler makes 100% of new-issue events still emit and remain visible while 0% are auto-processed, and a separately-enabled consumer (e.g. Workflow app) is able to act on the same event.
- **SC-005**: A token rotation results in 0 dropped deliveries during the grace window and old tokens being rejected after it.
- **SC-006**: No new issue opened by or on behalf of the user's own account is processed more than once (no self-trigger loop) across repeated self-updates.

## Assumptions

- The generic integration/webhook framework and the event kernel (emit, register handlers, enable/disable) already exist and are reused as-is; this feature adds a `gitlab` integration service, a `gitlab-issue-triage` headless handler, and the settings panel — it does not redesign those mechanisms.
- Webhook registration is manual (the user pastes the URL + token into GitLab's project webhook settings); automatic per-project `POST /projects/:id/hooks` registration is out of scope for v1.
- The primary resolution trigger is the `open` (new) action for both issues and MRs; close/reopen/update are accepted for event completeness but do not start a resolution workflow.
- Classification relies on a small, conventional label set (`bug`, `feature`, `enhancement`) present in typical GitLab projects; absence of labels triggers the agent-based fallback rather than an error.
- Self-hosted GitLab is supported at the instance-URL level; specific self-hosted version differences (older payload shapes) are not individually guaranteed.
- The resolution workflows ("bug pipeline", "feature pipeline") map onto BOS's existing spec/build capabilities; the handler orchestrates them and writes results back to GitLab — it does not redefine those workflows.
- The default triage handler's autonomy is bounded by Clarification Q2 (whether it may close the item itself vs. only propose).

## Clarifications

### Q1: How are new merge requests handled by the default triage handler?

**Context**: The user chose issue **+ MR** coverage (Decision 2c), but the bug/feature resolution workflow is fundamentally an *issue* concept — an MR is submitted work, not a request to resolve.

**What we need to know**: What should an enabled default handler do when a new MR opens?

| Option | Answer | Implications |
|--------|--------|--------------|
| A      | Event only — emit `gitlab.mr.opened`, no built-in workflow | SMALLEST scope; MRs rely on the user routing them (e.g. Workflow app). The handler still emits the event for both issue and MR. |
| B      | MR review workflow — a built-in handler that reviews the MR (correctness, spec conformance) and posts a review | Larger scope; needs a distinct MR-review flow and possibly a different handler than issue triage. |
| C      | Reuse issue triage — treat an MR like an issue (classify bug/feature) | Likely a poor fit; MRs usually aren't bug/feature requests. Not recommended. |
| Custom | Describe your own behavior | — |

*Recommended: **A** for v1 (emit both, triage issues, leave MR action to the user's routing) with **B** as a planned follow-up.*

### Q2: How much autonomy does the default handler have over the GitLab item?

**Context**: The user's standing philosophy is that autonomous self-modification should have an explicit safety boundary.

**What we need to know**: When a resolution run completes, may the default handler mutate the item's state, or only propose?

| Option | Answer | Implications |
|--------|--------|--------------|
| A      | Comment + label only; a human closes the item / opens the MR | Safest; the agent proposes, never finalizes. Matches "propose a fix" wording. |
| B      | Comment + label + close (for a verified bug) | More autonomous; a verified-and-fixed bug closes itself; risk of premature close. |
| C      | Full — including opening a draft MR for the fix | Most autonomous; turns a bug report into a concrete MR; highest value, highest risk. |
| Custom | Describe your own boundary | — |

*Recommended: **A** for v1 (propose, don't finalize) with the boundary explicitly recorded so it can be widened deliberately.*
