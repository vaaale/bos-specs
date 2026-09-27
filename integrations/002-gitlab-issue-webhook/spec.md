# Feature Specification: GitLab Issue Webhook Integration

**Feature Branch**: `002-gitlab-issue-webhook` (active branch: `bos/002-gitlab-issue-webhook`)

**Created**: 2026-09-26

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Improve the gitlab integration by adding support for a gitlab webhook. When an issue is registered, BOS is notified a new issue was created. BOS then goes through the appropriate workflow to resolve it — a bug is analyzed, verified, and a fix proposed; a new feature or improvement is spec'd. BOS updates the issue throughout the process. Prefer reusing existing skills and agents."

## Context

BOS already ships a **generic webhook platform** in the kernel (`src/lib/integrations/webhooks/`): a single receiver route at `/api/integrations/webhooks/[integrationId]/[serviceId]` that does handler lookup, enable/connect checks, raw-body read, signature `verify()`, delivery **idempotency** (dedup of retries), `receive()` parsing, and emission of each returned event onto the **event bus** (spec 034). Two integrations are registered today — Gmail (Pub/Sub/JWT) and Telegram (bot `setWebhook` / `X-Telegram-Bot-Api-Secret-Token`). This feature adds GitLab as a **third integration** on that same platform, and wires the resulting events into the **existing resolution pipelines** (self-heal for bugs; the Build Studio spec pipeline for features/improvements) so that an issue is worked and reported back to GitLab end-to-end.

The feature has two halves with a clean responsibility split:
- **Intake — BOS kernel, no GitLab credentials.** A `gitlab` integration on the existing **webhook platform** (`src/lib/integrations/services/{gsuite,telegram}/` + `webhooks/`): an `IntegrationManifest`, a `WebhookHandler` that verifies the shared `X-Gitlab-Token` header (constant-time compare — no GitLab API access), parses the payload, and publishes a typed event onto the bus, plus a headless classifier/router on the event bus. The kernel never holds or uses a GitLab API credential.
- **Resolution — headless agents via the GitLab MCP server.** Every agent-side GitLab operation (reading the issue, posting notes, setting `agent:*` labels, provisioning labels, pushing fix branches, opening draft MRs) goes through the existing **GitLab MCP server**, which is the single holder of the GitLab credential and the I/O layer for the issue. No new issue tools are introduced — the MCP already exposes `get_issue`, `create_issue_note`, `update_issue`, `list_labels`/`create_label`, and `create_merge_request`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Connect GitLab and enable the issue webhook (Priority: P1)

A user who has configured the GitLab MCP server (the credential holder for all GitLab operations — see Dependencies D2) opens the GitLab integration's service panel, chooses the **Issues** service, sets the target project, validates the GitLab connection, and enables the issue webhook. The panel shows the exact URL GitLab must POST to, the shared verification token, and copy-ready instructions for creating the project webhook in GitLab's own settings (manual registration: BOS publishes the URL + token; it does not call the GitLab API to create the hook). Once enabled, the panel reflects the live delivery state ("last received"). The full label set (classification + workflow labels) is provisioned in the project via the MCP the first time the system works on an issue there, so both humans (classification) and the system (workflow state) have the labels available.

**Why this priority**: Nothing downstream works until the integration is configured and the provider is pointed at BOS. This is the entry point and the smallest viable slice — after it, issue events are observable in BOS even before any resolution is wired.

**Independent Test**: With the GitLab MCP server configured and the connection validated, enable the Issues webhook for a project, confirm the panel surfaces the URL + token + manual-registration instructions, confirm a delivery from a webhook registered with that URL + token is accepted (401 for a wrong token, 404 while disabled), and confirm the integration state shows the service enabled.

**Acceptance Scenarios**:

1. **Given** the GitLab MCP server is configured and the connection validated, **When** the user enables the Issues webhook and provides the project path, **Then** the panel displays the absolute receiver URL (derived from the app origin), the shared verification token, and instructions for registering that URL + token as a project webhook (issue events) in GitLab.
2. **Given** the GitLab MCP server is not configured or the connection is not validated, **When** the user opens the Issues service, **Then** the integration reports not connected, the service cannot be enabled, and the receiver returns 404 for deliveries.
3. **Given** the system works on the first in-scope issue in a project, **When** it needs the label set, **Then** the full label set (the three classification labels and the five workflow labels from the Label Model) exists in the project, created via the MCP with any that are missing (idempotent).
4. **Given** the webhook is enabled, **When** GitLab delivers any issue event, **Then** the integration's "last received" indicator updates to the current time.
5. **Given** the Issues webhook is enabled, **When** the user disables it, **Then** the receiver no longer accepts deliveries for the service (404); removing the webhook in GitLab is the user's action (FR-004/FR-005).

---

### User Story 2 - A new issue is received, verified, and published to BOS (Priority: P1)

When GitLab creates an issue on a watched project, it POSTs an issue hook to the receiver. BOS authenticates the delivery against the shared token, deduplicates retries, parses the payload into a typed GitLab issue event, and publishes it onto the event bus so other BOS surfaces (event viewer, bell, headless handlers) can observe it. This is the "BOS is notified" half and is unconditional — every qualifying issue event reaches the bus regardless of whether it will later be resolved.

**Why this priority**: This is the core, verifiable contract of the feature — "when an issue is registered, BOS is notified." It is independently testable end-to-end with a single well-formed POST and a single malformed one, and it has value on its own (inbound GitLab issue visibility inside BOS).

**Independent Test**: POST a well-formed `object_kind: issue` payload with the correct token and assert a typed event appears on the bus; repeat the identical POST and assert it is deduplicated (no second event); POST with a wrong/missing token and assert a 401; POST to a disabled service and assert a 404.

**Acceptance Scenarios**:

1. **Given** the Issues webhook is enabled, **When** GitLab POSTs a valid `object_kind: issue` / `action: open` hook with the correct token, **Then** the receiver returns success and exactly one typed GitLab issue event is emitted onto the event bus carrying the project path, issue IID, title, description, labels, author, and issue URL.
2. **Given** a valid hook was already delivered, **When** GitLab retries the byte-identical hook, **Then** the receiver acknowledges success and emits no additional event (deduplication).
3. **Given** the Issues webhook is enabled, **When** a hook arrives with a missing or incorrect verification token, **Then** the receiver rejects it with 401 and emits no event.
4. **Given** the Issues webhook is disabled or the integration is not connected, **When** a hook arrives, **Then** the receiver returns 404 and emits no event, and the response does not distinguish "disabled" from "unknown."
5. **Given** the receiver, **When** a hook for a non-issue `object_kind` (e.g. note, merge_request) arrives, **Then** it is not turned into an issue event.

---

### User Story 3 - A qualifying issue is classified by its label and picked up (Priority: P2)

A headless handler watching the bus sees an issue event (an open, or an update carrying the issue's current labels) and reads the issue's **classification label**. The label is both the trigger and the category: `Bug`, `Feature`, or `Enhancement` selects exactly one pipeline; no label means out of scope; more than one means ambiguous and is rejected. When a pipeline accepts an in-scope issue, BOS posts a note to the GitLab issue announcing pickup and applies the `agent:in-progress` workflow label, so a human watching GitLab sees that work has begun.

**Why this priority**: Classification and pickup are the bridge between "notified" (US2) and "resolved" (US4/US5). They are independently testable: feed a labeled issue and assert it is routed to the right pipeline and annotated, feed an unlabeled or multi-labeled issue and assert it is not started.

**Independent Test**: Create an issue labeled `Bug` and assert it is classified as a bug, routed to the bug pipeline, and the issue gains a pickup note + the `agent:in-progress` label. Create an issue with no classification label and assert no pipeline starts and the issue is unchanged. Create an issue labeled both `Bug` and `Feature` and assert it is rejected with an ambiguity note.

**Acceptance Scenarios**:

1. **Given** a new issue event, **When** the handler reads the issue's classification label, **Then** the issue is routed to the pipeline selected by that label — `Bug` → self-heal, `Feature` → feature-request, `Enhancement` → feature-improvement — with no inference.
2. **Given** an issue carries exactly one classification label, **When** it is processed, **Then** that label is the sole determinant of the category and pipeline.
3. **Given** an issue carries no classification label, **When** it is processed, **Then** no pipeline starts and the issue is left unchanged in GitLab.
4. **Given** an issue carries more than one classification label, **When** it is processed, **Then** no pipeline starts and a note records that exactly one of `Bug`/`Feature`/`Enhancement` must be applied.
5. **Given** a pipeline accepts an in-scope issue, **When** pickup occurs, **Then** a note is posted to the issue stating that automated resolution has started, and the issue is labeled `agent:in-progress`.

---

### User Story 4 - A bug is analyzed, verified, and a fix is proposed (Priority: P2)

For a `Bug`-labeled issue, BOS runs the existing **self-heal / diagnose-verify-fix** pipeline against the issue: it analyzes the report, verifies the failure is real and reproducible, and produces a proposed fix as a **draft** merge request on a feature branch (the preview). At each stage the GitLab issue is updated — analysis findings, verification result, and the link to the proposed fix — so the issue thread becomes a running record of the work. The outcome is a **proposal**, not an applied change: the draft MR is opened for human review and the issue is labeled `agent:preview-ready`, the system's terminal state. Promotion is a human decision.

**Why this priority**: This is the flagship outcome the user asked for ("the bug should be analyzed, verified, and a fix should be proposed") and it reuses the existing self-heal pipeline rather than inventing a new one.

**Independent Test**: Open a `Bug`-labeled issue describing a reproducible defect; assert the issue gains an analysis note, a verification note, and a draft MR is created and linked; assert the issue ends labeled `agent:preview-ready` and the draft MR is in draft (unmerged) state.

**Acceptance Scenarios**:

1. **Given** a `Bug`-labeled issue, **When** the pipeline runs, **Then** the issue receives a note summarizing the root-cause analysis.
2. **Given** a `Bug`-labeled issue, **When** the pipeline verifies the defect, **Then** the issue receives a note recording whether the failure was reproduced and the evidence.
3. **Given** a verified bug, **When** a fix is produced, **Then** a **draft** merge request is created via the GitLab API on a feature branch, the issue is linked to it, and a note posts the MR link.
4. **Given** a bug pipeline completes with a proposal, **When** it finishes, **Then** the issue is labeled `agent:preview-ready` and the draft MR remains unmerged, awaiting human Promote/Discard.
5. **Given** the defect cannot be reproduced or the fix cannot be produced, **When** the pipeline ends, **Then** the issue is labeled `agent:failed` and a note explains why; where it can proceed once a question is answered, the issue is labeled `agent:needs-input` and the question is posted.

---

### User Story 5 - A feature or enhancement is driven to a preview (Priority: P3)

For a `Feature`- or `Enhancement`-labeled issue, BOS runs the existing **Build Studio spec pipeline** scoped to the issue and drives it to an implemented **preview on a feature branch**: it captures confirmed requirements, drafts the spec, and builds + tests the preview. (`Feature` is a new feature; `Enhancement` is an incremental scope-add to an existing one.) The issue is updated as the pipeline advances — requirements captured, spec drafted, implementation, preview-ready — each with a note and a link where applicable, so the GitLab issue and the BOS artifacts stay mutually visible. The outcome is a **preview**, not promoted code: the issue is labeled `agent:preview-ready` and promotion is a human decision.

**Why this priority**: It is the same "route to the appropriate existing workflow" principle as US4 but for the feature/improvement branch, and it is the lower-frequency path, so it is lower priority than the bug path while remaining part of the full feature.

**Independent Test**: Open a `Feature`-labeled issue; assert it is routed to the spec pipeline, the issue gains stage notes (requirements, spec, implementation) with links, a working preview exists on a feature branch, and the issue ends labeled `agent:preview-ready`.

**Acceptance Scenarios**:

1. **Given** a `Feature`- or `Enhancement`-labeled issue, **When** the pipeline runs, **Then** the issue's intent is captured into a set of confirmed requirements and the issue notes that step.
2. **Given** requirements are confirmed, **When** the spec is drafted, **Then** a spec artifact is produced, the issue is updated with a link to it, and a note is posted.
3. **Given** a spec exists, **When** the preview is implemented and tested, **Then** a working preview exists on a feature branch, the issue links to it, and a note is posted.
4. **Given** the feature pipeline completes with a preview, **When** it finishes, **Then** the issue is labeled `agent:preview-ready` and the preview remains un-promoted, awaiting human Promote/Discard.
5. **Given** the feature pipeline needs information it cannot infer, **When** it reaches that point, **Then** the issue is labeled `agent:needs-input` and a note posts the specific question to the issue thread.

---

### User Story 6 - The issue reflects the whole lifecycle (Priority: P3)

Across all pipelines, the GitLab issue is the user-visible surface of progress: a single machine-managed workflow label (one of the five `agent:` states) plus append-only stage notes let a human read the current status of automated work directly in GitLab without opening BOS. The lifecycle is consistent regardless of category — `agent:queued` → `agent:in-progress` → a terminal state (`agent:preview-ready`, `agent:failed`, or `agent:needs-input`) — and every transition is annotated with a note.

**Why this priority**: It is the cross-cutting contract that makes US3–US5 legible in GitLab; it is independently testable by asserting the expected workflow label + notes at each stage of a driven run.

**Independent Test**: Drive one issue through a pipeline stage by stage and assert the issue's workflow label and note set match the expected sequence at each step, including a failure path.

**Acceptance Scenarios**:

1. **Given** an issue being resolved, **When** it moves between lifecycle stages, **Then** its workflow label updates to reflect the current stage and a note records the transition.
2. **Given** an issue reaches the successful terminal state, **When** the pipeline finishes, **Then** the issue carries `agent:preview-ready` and a final summary note linking the produced artifact (draft MR or preview).
3. **Given** an issue's pipeline fails or requires input, **When** it ends in that state, **Then** the issue carries `agent:failed` or `agent:needs-input` and a note that states the blocker and the required next action.

---

### Edge Cases

- **Duplicate delivery**: GitLab retries a hook; the identical payload is acknowledged but not re-emitted (framework idempotency; the handler supplies a stable idempotency key when the payload has one and relies on the body-hash fallback otherwise).
- **Token rotation**: the verification token is changed in Settings; deliveries signed with the old token are still accepted during a documented grace window, then rejected.
- **Out-of-scope issue**: an issue that carries no classification label is observed on the bus (US2) but does not start a pipeline and leaves the issue unchanged.
- **Ambiguous issue**: an issue carrying more than one classification label is rejected with a note that exactly one of `Bug`/`Feature`/`Enhancement` must be applied.
- **Issue edited or closed mid-run**: if the issue is closed while a pipeline is in flight, the pipeline stops further GitLab-mutating steps and records the abandonment on the issue; it does not open a draft MR or preview against a closed issue.
- **Concurrent issues**: multiple qualifying issues are processed independently; one issue's pipeline state never affects another's.
- **GitLab access unavailable during resolution**: if the MCP server's credential is revoked or lacks permission for the target project mid-run, the pipeline posts a note and labels the issue `agent:failed` (or `agent:needs-input` when a configuration fix would unblock it); intake (webhook → bus) is unaffected.
- **Classification label applied after creation**: an issue created without a classification label and later labeled `Bug`/`Feature`/`Enhancement` is picked up on the label-change event (the normal file-then-classify workflow); self-noise protection still holds because the system only ever writes `agent:*` workflow labels, never classification labels.
- **Self-authored noise**: issue events that BOS itself generates (e.g. notes it posts, workflow label changes it applies) do not re-trigger a pipeline for the same issue.
- **Re-opened issue**: an issue that already has a workflow label and is re-opened does not silently restart a completed pipeline; re-processing requires a new explicit trigger (out of scope for v1 — noted for follow-up).

## Requirements *(mandatory)*

### Label Model

The feature operates on two **disjoint** label namespaces in the target GitLab project. They must not collide: a classification label is never a workflow label, and vice versa.

**Classification labels** — set by a human (or the issue author) on the issue. They select the pipeline and are the opt-in trigger for resolution. **Fixed vocabulary, not configurable.** An issue carrying exactly one is in scope; none → out of scope; more than one → ambiguous (rejected with a note).

| Label | Category | Routed pipeline |
|-------|----------|-----------------|
| `Bug` | bug | Self-heal (diagnose → verify → proposed fix as a draft MR) |
| `Feature` | feature request | Build Studio spec pipeline (new feature, driven to an implemented preview) |
| `Enhancement` | feature improvement | Build Studio spec pipeline (incremental scope-add, driven to an implemented preview) |

**Workflow labels** — set and transitioned **only by the system** to report the lifecycle stage. Mutually exclusive (exactly one active at a time). Prefixed `agent:` to namespace them from human labels. The furthest the system ever applies is `agent:preview-ready`; there is deliberately **no** "promoted"/"merged" label because promotion is a human decision the system never makes.

| Label | Meaning | Stage class |
|-------|---------|-------------|
| `agent:queued` | Accepted and in scope; pipeline not yet started | non-terminal |
| `agent:in-progress` | A pipeline is actively working the issue | non-terminal |
| `agent:needs-input` | Blocked on a question posted to the issue thread | blocking |
| `agent:failed` | Ended without a preview (not reproducible, spec/preview could not be produced) | terminal |
| `agent:preview-ready` | **System terminal state** — a preview exists (draft MR, or built + tested preview on a feature branch) awaiting human Promote/Discard | terminal |

**Stage notes vs workflow labels** — the workflow label carries the *coarse* state (one of five); append-only *stage notes* carry the *fine-grained* narrative (analysis, verification, requirements, spec, implementation, preview-ready, and any blocker). A human reading only the issue (one label + the notes) can state the current status without opening BOS.

### Functional Requirements

**Integration & configuration**

- **FR-001**: The system MUST expose a GitLab integration with an **Issues** service in Settings, alongside the existing Telegram and Gmail integrations, using the existing integration-registry mechanism. The integration's connection state MUST reflect the availability of the GitLab MCP server (configured, enabled, and a validated check against the target project); when GitLab access is unavailable the integration is not connected, the Issues service cannot be enabled, and the receiver does not accept deliveries.
- **FR-002**: The Issues service configuration MUST let the user select the target **project** (project path) and enable or disable the issue webhook.
- **FR-003**: The Issues service panel MUST display the absolute receiver URL (derived from the app origin) and the shared verification token for the enabled webhook.
- **FR-004**: The system MUST publish the receiver URL and the shared verification token in a form the user can copy and register manually as a project-level webhook (bound to issue events) in GitLab's own webhook settings. The system MUST NOT create or modify provider-side webhooks — no GitLab API credential exists in BOS for that purpose.
- **FR-005**: The system MUST support disabling the webhook, which stops the receiver from accepting deliveries for the service. Removing the provider-side webhook is the user's action in GitLab (see FR-004).
- **FR-006**: The system MUST reflect live delivery state on the service panel (a "last received" indicator that updates when a delivery is processed).
- **FR-007**: The system MUST support rotation of the shared verification token, minting a new primary token while accepting the previous one for a documented grace period.

**Intake (webhook → event bus)**

- **FR-008**: The system MUST accept GitLab issue hook deliveries at the generic webhook receiver for the `gitlab` / `issues` service, reusing the existing receiver route, idempotency, and event-emission plumbing.
- **FR-009**: The system MUST authenticate each delivery by comparing the `X-Gitlab-Token` header to the stored shared token using a constant-time comparison, and reject unauthenticated deliveries.
- **FR-010**: The system MUST parse an authenticated `object_kind: issue` delivery into a typed GitLab issue event carrying, at minimum: project path, issue IID, title, description, labels, author, action (open / update / close), and the issue URL.
- **FR-011**: The system MUST publish each parsed issue event onto the event bus so that BOS surfaces and headless handlers can observe it.
- **FR-012**: The system MUST deduplicate retried deliveries so that a byte-identical hook (or one carrying the same idempotency key) is emitted at most once.
- **FR-013**: The system MUST not emit an issue event for deliveries whose `object_kind` is not an issue, or for a disabled service, a disconnected integration, or a failed verification.

**Classification & routing**

- **FR-014**: The system MUST provide a headless handler that watches the event bus for GitLab issue events (open and update) and decides whether each is in scope for automated resolution — in scope iff the issue currently carries exactly one classification label (`Bug`, `Feature`, or `Enhancement`). A classification label applied after the issue was created (the normal file-then-classify workflow) MUST be picked up on the label-change event; per-issue state MUST make repeated processing of the same issue a no-op.
- **FR-015**: The system MUST classify an in-scope issue into exactly one category, and the category MUST be determined solely from the classification label assigned to the issue — never inferred.
- **FR-016**: The system MUST use the fixed classification label vocabulary defined in the Label Model — `Bug` (→ bug pipeline), `Feature` (→ feature-request pipeline), `Enhancement` (→ feature-improvement pipeline) — and MUST treat the label as authoritative.
- **FR-017**: The system MUST NOT start a pipeline for an issue that carries none of the three classification labels (out of scope — intake only) or that carries more than one (ambiguous — a note records that exactly one of `Bug`/`Feature`/`Enhancement` must be applied).
- **FR-018**: The system MUST route a bug-classified issue to the existing self-heal / diagnose-verify-fix pipeline.
- **FR-019**: The system MUST route a feature-request- or feature-improvement-classified issue to the existing Build Studio spec pipeline.
- **FR-020**: The system MUST not start a pipeline for an out-of-scope issue, and MUST leave such an issue unchanged in GitLab.

**Resolution & issue updates**

- **FR-021**: For a bug, the pipeline MUST produce an analysis, a verification result, and — when a fix is produced — a **draft** merge request on a feature branch (the preview) pushed to the target project and created via the GitLab MCP server, and MUST link the issue to that draft MR; on success the issue is labeled `agent:preview-ready`.
- **FR-022**: For a feature request or improvement, the pipeline MUST produce a confirmed set of requirements, a specification, and a built, tested **preview on a feature branch**, each linked to the issue; on success the issue is labeled `agent:preview-ready`.
- **FR-023**: The system MUST post a note to the GitLab issue when a pipeline accepts the issue, stating that automated resolution has started.
- **FR-024**: The system MUST post a note to the GitLab issue at each significant pipeline stage (analysis, verification, requirements, spec, implementation, preview-ready, needs-input, failed) summarizing that stage's outcome.
- **FR-025**: The system MUST maintain exactly one workflow label (from the Label Model) on the GitLab issue reflecting the current lifecycle stage, transitioning it as the pipeline advances; the furthest stage the system applies is `agent:preview-ready`.
- **FR-026**: The system MUST treat every resolution outcome as a proposal: the system's terminal state is `agent:preview-ready` (a draft MR or an un-promoted preview). It MUST NOT merge, close, or promote anything, MUST NOT apply any "promoted"/"merged" label, and MUST leave promotion exclusively to a human.
- **FR-027**: The system MUST, when a pipeline requires information it cannot infer, move the issue to the `agent:needs-input` workflow label and post the specific question as a note to the issue thread.
- **FR-028**: The system MUST stop further GitLab-mutating steps and record abandonment on the issue when the issue is closed while a pipeline is in flight.

**Lifecycle & safety**

- **FR-029**: The system MUST process multiple qualifying issues concurrently and independently, with one issue's pipeline state isolated from another's.
- **FR-030**: The system MUST ensure that events generated by BOS's own writes to an issue (its own notes, workflow label changes, draft MRs) do not re-trigger a pipeline for that same issue.

**Label provisioning**

- **FR-031**: The first time the system works on an in-scope issue in a project, it MUST ensure the full label set defined in the Label Model (three classification labels + five workflow labels) exists in that project, creating any that are missing via the GitLab MCP server (idempotent), so both humans (classification) and the system (workflow state) have the labels available.

### Non-Functional Requirements

- **NFR-001 (Security)**: Webhook verification MUST use constant-time comparison; a disabled or disconnected service MUST return 404 (not 401) so an attacker cannot distinguish the two; the receiver MUST NOT echo or log the delivery body or the verification token.
- **NFR-002 (Idempotency)**: Retried deliveries MUST be acknowledged without re-emission or re-processing (see FR-012).
- **NFR-003 (Latency)**: From a valid GitLab POST to the corresponding event being on the event bus, end-to-end MUST complete in under 2 seconds (p95) under normal conditions.
- **NFR-004 (Autonomy boundary)**: The system MUST operate propose-only and stop at `agent:preview-ready`; it MUST NOT apply any "promoted"/"merged" label, and human review MUST be required before any merge, close, or promotion (enforced across FR-021, FR-022, FR-026).
- **NFR-005 (Re-entrancy)**: Concurrent issues MUST be independently processable with no shared mutable pipeline state across issues (FR-029).
- **NFR-006 (Observability)**: The full lifecycle of automated work MUST be visible in the GitLab issue (a single workflow label + stage notes) without requiring access to BOS (FR-024, FR-025, User Story 6).
- **NFR-007 (Reuse)**: The feature MUST reuse the existing webhook platform, the existing GitLab MCP server (as the sole GitLab credential holder and I/O layer for all agent-side operations), and the existing self-heal and Build Studio pipelines. The new kernel code is limited to the GitLab `WebhookHandler` + integration manifest, the classifier/router headless handler, and the per-issue resolution-state map; the kernel MUST NOT hold or use GitLab API credentials, and no new GitLab issue tools are introduced.

### Key Entities

- **GitLab Integration**: the registered integration (id `gitlab`) exposing the Issues service; its connection state is derived from GitLab access via the MCP server (validated, FR-001), and its service config holds the target project path and the webhook enabled flag. The kernel holds no GitLab credentials and no provider hook id (registration is manual, FR-004).
- **Issues Webhook Config**: per-service configuration — enabled flag, target project path, and the shared verification token (stored in the secrets store, with rotation). No provider hook id (manual registration).
- **GitLab Issue Event**: the typed event emitted to the bus from an authenticated issue hook — project path, issue IID, title, description, labels, author, action, issue URL.
- **Issue Resolution State**: the lifecycle record for one issue being worked — category (bug / feature request / feature improvement, read from its classification label), current workflow label (one of the five `agent:` states), and the produced artifacts (draft MR id / spec + preview reference). Drives the issue's workflow label and stage notes.
- **Classification Label vs Workflow Label**: two distinct, non-colliding namespaces — the user-set classification label (`Bug`/`Feature`/`Enhancement`) selects the pipeline (input), while the machine-managed workflow label (`agent:*`) reports lifecycle stage (output). A classification label is never a workflow label and vice versa.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user who already has GitLab connected can enable the issue webhook for a project and see a live delivery from GitLab within 5 minutes.
- **SC-002**: 100% of validly-signed issue deliveries are emitted to the event bus; 0% of unsigned or mis-signed deliveries are emitted.
- **SC-003**: A retried (duplicate) delivery produces no duplicate event and no duplicate pipeline run.
- **SC-004**: 100% of in-scope issues are classified by their classification label (`Bug`/`Feature`/`Enhancement`) with zero LLM inference; an issue with none of the three labels results in 0 pipeline runs, and an issue with multiple labels results in 0 pipeline runs plus an ambiguity note.
- **SC-005**: For a `Bug` issue, the issue thread contains, in order, an analysis note, a verification note, and a draft-MR link; the issue ends labeled `agent:preview-ready` with the draft MR unmerged.
- **SC-006**: For a `Feature`/`Enhancement` issue, the issue thread contains stage notes linking the requirements, the spec, and the preview, and the issue ends labeled `agent:preview-ready` with a working preview on a feature branch.
- **SC-007**: At every stage, a human reading only the GitLab issue (one workflow label + the notes) can state the current status of the automated work without opening BOS.
- **SC-008**: 0% of resolution outcomes are auto-merged, auto-closed, or auto-promoted, and the system applies no "promoted"/"merged" label; every successful outcome ends at `agent:preview-ready` as a proposal requiring human review.
- **SC-009**: Out-of-scope and ambiguous issues result in 0 pipeline runs and 0 workflow-label mutations to the issue in GitLab.

## Assumptions

- **A1 (Reachability / pure push)**: The issue webhook is **push-only** in v1 — GitLab POSTs to a receiver URL derived from the app origin (`NEXT_PUBLIC_APP_ORIGIN`), exactly the model the Telegram and Gmail webhooks already use. This requires the receiver URL to be reachable from the GitLab instance (self-hosted-on-LAN, or a public endpoint/tunnel for cloud GitLab → local BOS). A poll/pull fallback (periodic `list_issues`) is **out of scope for v1** and would be a follow-up if the user's topology has no reachable URL. This does not affect the correctness of the spec; it affects only whether the user's specific deployment can use push directly.
- **A2 (Label-driven scope)**: "BOS is notified" (US2, intake) is **unconditional** — every issue event reaches the bus. "BOS resolves it" (US3–US5) is **gated by the classification label**: an issue starts a pipeline iff it carries exactly one of `Bug`/`Feature`/`Enhancement`. That single label is both the opt-in trigger and the category selector — there is no separate gate label and no "resolve all" mode. This keeps automated work opt-in per issue.
- **A3 (Propose-only autonomy)**: The system never merges, closes, or promotes. The system's terminal state is `agent:preview-ready` — a draft MR (bug) or a built, tested preview on a feature branch (feature/enhancement). Promotion/Discard is a human decision. Enforced by NFR-004.
- **A4 (Label-only, fixed vocabulary)**: Classification is **label-only** over the fixed vocabulary `Bug`/`Feature`/`Enhancement` (Label Model); there is **no LLM inference fallback** and the vocabulary is **not configurable**. An issue with none of the three is out of scope; one with multiple is ambiguous (FR-015…FR-017).
- **A5 (Reuse over new agents)**: Resolution reuses the existing self-heal pipeline (bugs) and the Build Studio spec pipeline (features/improvements) and their existing agents; no new resolution agent is introduced. The only new kernel orchestration is the classifier/router headless handler and the per-issue resolution-state map; all GitLab I/O is delegated to the GitLab MCP server inside headless agent runs (NFR-007).
- **A6 (Watched project is buildable by the fix pipeline)**: v1 assumes the target GitLab project is a repository the fix pipeline can build in and push to (e.g. a BOS checkout hosted in the user's GitLab). The bug path's draft MR is pushed to that project and opened there; if a project is not buildable by the pipeline, the issue ends in `agent:failed`/`agent:needs-input` with an explanatory note. Arbitrary user repositories are a follow-up.
- **A7 (Project-level hooks)**: v1 supports project-level GitLab webhooks (one project per Issues service config). Group-level webhooks are out of scope for v1.
- **A8 (Issue hooks only)**: v1 handles `object_kind: issue` events (open / update / close as relevant). Merge-request, note, pipeline, and other hook types are out of scope.
- **A9 (Preview depth, not promote)**: The furthest state the system drives to is `agent:preview-ready`. For a **bug** that is a draft MR on a feature branch; for a **feature/enhancement** it is a built, tested preview on a feature branch (the spec pipeline is driven to implementation as an autonomous run). The system **never promotes** — promotion/discard of any preview is a human decision (FR-026, NFR-004).

## Dependencies

- **D1 — Existing webhook platform** (`src/lib/integrations/webhooks/`): receiver route, `WebhookHandler` interface, registry, `WebhookManager` lifecycle, secrets store, idempotency, and `emitIntegrationEvent` → event-bus bridge. This feature is a new handler + registry entry on this platform, not a parallel one.
- **D2 — GitLab MCP server** (configured with a credential authorized for the target project): the single holder of the GitLab credential and the I/O layer for all agent-side operations (issue read, notes, labels, draft MRs). The BOS kernel holds no GitLab API credentials. The integration's connection state is derived from this server's availability.
- **D3 — Event bus (spec 034)**: the canonical notification + headless-handler mechanism; the intake event and the resolution handler both live on it.
- **D4 — Self-heal pipeline (spec 031)**: the diagnose → verify → propose-fix pipeline a bug is routed into.
- **D5 — Build Studio spec pipeline (spec-kit method)**: the specify/design/plan/implement pipeline a feature/improvement is routed into, driven to a tested preview.
- **D6 — GitLab API surface via the MCP server** (`get_issue`, `create_issue_note`, `update_issue` (labels/state), `list_labels`/`create_label`, `create_merge_request` (draft)): the issue-update, label-provisioning, and draft-MR glue used by the resolving agents.
