# Feature Specification: Self-Healing Mechanism

**Feature Branch**: `031-self-healing`

**Created**: 2026-09-07

**Status**: Draft

**App Target**: `bos-core`

**Input**: User description: "A self-healing mechanism that detects errors/failures in BOS, autonomously diagnoses whether they represent genuine gaps (vs. agent misuse or environmental issues), and drives the full fix pipeline (specify → implement → preview) without stopping at step boundaries. The user is notified when a fix is ready for review. The mechanism extends the existing Conversation Reviewer agent into a Diagnostician, integrates with the Build Studio pipeline, the 005 self-modification preview system, and the 034 event system."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Explicit problem report drives an autonomous fix (Priority: P1)

The user (or an agent) reports a problem via the `self_heal.request` tool. The Diagnostician investigates, writes a markdown diagnostics report, classifies the scope. If the scope is a genuine BOS-core gap (class e) or a user-owned app bug (class d-bis), the system drives the full Build Studio pipeline autonomously — specify through implement — and the developer produces a fix with TDD on a feature branch. When the preview is ready, the user receives a `fix_ready` event and can promote.

**Why this priority**: This is the headline capability. It proves the end-to-end loop: detect → diagnose → classify → implement → notify → human-gated promote. It's the path that fixes the `bos_app_launch` params gap (the acceptance test case).

**Independent Test**: Fire a `self_heal.request` event with the known `bos_app_launch`-missing-params signature. Verify: Diagnostician produces a markdown report classifying it `e`, the autonomous pipeline runs to completion, a preview is built with the `params` fix + a passing regression test, and a `fix_ready` event is emitted.

**Acceptance Scenarios**:

1. **Given** the user calls `self_heal.request` with "the agent couldn't open the portfolio report in the Editor because `bos_app_launch` doesn't accept a file parameter", **When** the Diagnostician investigates, **Then** it produces a markdown report with YAML frontmatter `scopeClass: e`, `ownership: bos-core`, `proposedSurface: "bos_app_launch tool schema + handler in FrontendToolsV2.tsx"`, and a narrative citing `os-store.ts:11` (params supported) vs `FrontendToolsV2.tsx:22` (params dropped).
2. **Given** the Diagnostician's report classifies the scope as `e`, **When** the fast spine emits `case_escalated`, **Then** a Build Studio conversation is seeded with the report as the spec input, and the pipeline runs autonomously from `specify` through `implement` without stopping at step boundaries.
3. **Given** the developer has implemented the fix on the feature branch, **When** the preview passes typecheck/build/health and the test suite reports ≥95% coverage on modified files, **Then** the system emits `self_heal.fix_ready` with the preview branch name, a fix summary, and a link.
4. **Given** the user receives the `fix_ready` event, **When** they click it in the Event Viewer, **Then** the preview is pinned to their session and the Topbar shows the normal Promote/Discard controls.

---

### User Story 2 - Automatic detection of tool failures triggers diagnosis (Priority: P1)

BOS detects that an agent's tool call has thrown a non-environmental error, or that N consecutive calls to the same tool have failed with the same signature. The trigger fires, the Diagnostician runs, and the case is resolved by scope class.

**Why this priority**: This is the primary *automatic* trigger (alongside explicit). Without it, the user must manually report every problem. With it, the system notices issues the user hasn't even noticed yet.

**Independent Test**: Simulate 3 consecutive `file_read` calls failing with the same "permission denied" error signature (non-environmental). Verify: the repeated-failure trigger fires, the Diagnostician runs, produces a report, and the case resolves correctly (e.g., class b — the skill mis-teaches the tool, or class e — a real permission bug).

**Acceptance Scenarios**:

1. **Given** an agent's tool call throws an exception that is NOT in the environmental allowlist (network, DNS, auth, rate-limit, OOM, external timeout), **When** the hard-error trigger is enabled, **Then** the self-heal mechanism is invoked with the error signature as input.
2. **Given** 3 consecutive calls to the same tool fail with the same error signature within 300 seconds, **When** the repeated-failure trigger is enabled, **Then** the mechanism is invoked with the correlated failure pattern as input.
3. **Given** a tool call fails with a network timeout error, **When** the hard-error trigger is enabled, **Then** the mechanism is NOT invoked (environmental allowlist filter).
4. **Given** the same failure signature was already diagnosed and closed 2 hours ago, **When** the trigger fires again, **Then** no new case is created (dedupe window, default 86400s).

---

### User Story 3 - Workflow timeout triggers diagnosis with correct scoping (Priority: P2)

A workflow run or long-running operation times out. The trigger fires, the Diagnostician determines whether the cause is a workflow definition problem (class c — fix the workflow) or an app bug (class d/d-bis — notify or fix the app).

**Why this priority**: Workflows are a major failure surface (the user has hit timeout bugs before). Correctly scoping the fix (workflow vs. app) prevents the mechanism from modifying the wrong artifact.

**Independent Test**: Create a workflow with a node that will inevitably timeout (e.g., a research node with an impossible query and a 10s timeout). Run it. Verify: the timeout trigger fires, the Diagnostician classifies it `c` (workflow definition issue), and proposes a workflow edit (not an app change).

**Acceptance Scenarios**:

1. **Given** a workflow run exceeds its configured timeout, **When** the workflow-timeout trigger is enabled, **Then** the mechanism is invoked with the timeout record (workflow id, node that timed out, configured vs. actual duration) as input.
2. **Given** the Diagnostician determines the timeout is caused by a misconfigured workflow node (e.g., an impossibly short timeout for the node type), **When** it writes the report, **Then** the scope class is `c` and the proposed surface is the workflow definition file.
3. **Given** the Diagnostician determines the timeout is caused by a bug in the app the workflow calls, **When** it writes the report, **Then** the scope class is `d` or `d-bis` (depending on ownership) and the proposed surface is the app, not the workflow.

---

### User Story 4 - User-owned marketplace app bugs are fixable (Priority: P2)

A bug is detected in a marketplace app that the user maintains (it exists in `data/user-apps/items/<id>/`). The Diagnostician classifies it `d-bis`, and the system drives the fix through the Build Studio pipeline with `app_build` as the delivery mechanism.

**Why this priority**: The user maintains their own marketplace items and expects the self-healer to be able to fix them, not just notify about them. This closes the ownership boundary correctly.

**Independent Test**: Introduce a known bug into a user-owned item (e.g., a broken tool handler in the OKF KB service). Trigger the self-heal mechanism. Verify: the Diagnostician detects the item in `data/user-apps/items/okf-knowledge-base/`, classifies it `d-bis`, and the pipeline produces an `app_build` preview with the fix.

**Acceptance Scenarios**:

1. **Given** a failure signature points to a marketplace app, **When** the Diagnostician checks `data/user-apps/items/<id>/`, **Then** it finds the item and classifies the scope as `d-bis` (user-owned, fixable).
2. **Given** a failure signature points to a marketplace app, **When** the Diagnostician checks `data/user-apps/items/<id>/` and finds nothing, **Then** it classifies the scope as `d` (not owned, notify only).
3. **Given** the scope is `d-bis`, **When** the pipeline completes, **Then** the fix is delivered as an `app_build` preview (not a BOS-source feature branch), and `fix_ready` is emitted with the app id.

---

### User Story 5 - Self-healing behavior is configurable in Settings (Priority: P2)

The user can enable/disable the global mechanism, toggle each trigger type independently, configure the Diagnostician's scheduling, set the TDD coverage target, the cost cap, and the dedupe window — all from a Settings → Self Improvement page.

**Why this priority**: Without configuration, the user can't dial the mechanism up or down as they trust it. A fresh install should be conservative (only explicit trigger on); a trusted setup can enable all automatic triggers.

**Independent Test**: Open Settings → Self Improvement. Toggle the global kill switch off. Verify: no triggers fire, no scheduled execution runs, no LLM tokens are consumed by the mechanism. Toggle it back on, enable the hard-error trigger, simulate a non-env error. Verify: the mechanism fires.

**Acceptance Scenarios**:

1. **Given** `selfHeal.enabled` is false, **When** any trigger condition is met, **Then** the mechanism does NOT fire, no case is created, and no LLM tokens are consumed.
2. **Given** `selfHeal.enabled` is true but `selfHeal.triggers.hardError` is false, **When** a non-environmental tool error occurs, **Then** the hard-error trigger does NOT fire (but the explicit trigger still would).
3. **Given** `selfHeal.diagnostician.scheduled` is true and `selfHeal.diagnostician.idleThresholdSec` is 300, **When** a conversation has been idle for 300 seconds, **Then** the Diagnostician runs a review pass over it (Mode 1) and over the pending failure queue (Mode 2).
4. **Given** `selfHeal.costCapPerDay` is set to a value below the day's accumulated usage, **When** a new trigger fires, **Then** the case is queued (not dropped) and processed the following day.

---

### User Story 6 - The Diagnostician runs on a schedule for proactive review (Priority: P3)

The Diagnostician is scheduled (via Settings) to periodically review idle conversations (Mode 1: behavioral issues) and the pending failure queue (Mode 2: gap detection). This provides proactive, not just reactive, self-healing.

**Why this priority**: Reactive triggers catch problems as they happen. Scheduled review catches problems that didn't trigger a hard error but represent a pattern (e.g., an agent consistently taking a suboptimal approach that doesn't error but wastes tokens).

**Independent Test**: Enable `selfHeal.diagnostician.scheduled`. Wait for the idle threshold. Verify: the Diagnostician runs, produces markdown reports for any conversations with behavioral issues (Mode 1) and for any queued failure signatures (Mode 2), and cases are created for actionable findings.

**Acceptance Scenarios**:

1. **Given** the Diagnostician is scheduled and a conversation has been idle past the threshold, **When** the scheduler fires, **Then** the Diagnostician reviews the conversation (Mode 1) and writes a markdown report to `/Documents/BOS Improvements/`.
2. **Given** the Diagnostician is scheduled and there are pending failure signatures in the event queue, **When** the scheduler fires, **Then** the Diagnostician processes each signature (Mode 2) and writes a diagnostics report for each.
3. **Given** the Diagnostician's review finds no actionable issues, **When** it completes, **Then** a brief "no issues found" report is written and no case is created.

---

### User Story 7 - Unresolvable decisions suspend the pipeline and await user input (Priority: P3)

During the autonomous Build Studio pipeline, the Diagnostician (or the BS agent) encounters a decision it cannot resolve autonomously (e.g., a scope conflict that changes the fix surface, or missing information no research can fill). The pipeline emits a `decision_needed` event with the question, suspends, and resumes when the user answers.

**Why this priority**: This is the safety valve for the autonomous pipeline. Without it, the system would either guess (risky for code changes) or block forever. With it, the system asks only when it genuinely must.

**Independent Test**: Create a scenario where the Diagnostician's report is ambiguous (e.g., the gap could be fixed in tool A or tool B, and both are valid). Verify: the pipeline emits `decision_needed` with the question, the run suspends, the user answers via the BS Self-Heal page, the pipeline resumes and completes.

**Acceptance Scenarios**:

1. **Given** the autonomous pipeline encounters an unresolvable decision, **When** it cannot proceed, **Then** it emits `self_heal.decision_needed` with the question, context, and the case id, and the run transitions to suspended.
2. **Given** the run is suspended, **When** the user provides an answer via the BS Self-Heal page (which emits `self_heal.decision_resolved`), **Then** the run resumes from the suspension point with the answer injected.
3. **Given** the run is suspended, **When** the suspended timeout expires (default 7 days), **Then** the system emits `self_heal.abandoned`, the case closes, and the run terminates.
4. **Given** the autonomous pipeline does NOT encounter an unresolvable decision, **When** it runs, **Then** it completes without suspending (the common case — most gaps are unambiguous once diagnosed).

---

### Edge Cases

- **Stale diagnosis**: The Diagnostician classifies a gap that was already fixed in a prior promote. The pipeline's `specify` step finds the existing spec/fix and the Diagnostician's report notes "already addressed" — no new case is created, the existing one is referenced.
- **Preview build failure**: The developer's fix doesn't pass typecheck/build/health. The case transitions to `failed` status; the user is notified via the case timeline; no auto-retry (the user can re-trigger manually).
- **User discards a preview**: The case closes as `dismissed`; the failure signature enters the dedupe window (won't re-trigger within the window).
- **Simultaneous triggers for the same root cause**: The dedupe (FR-019) ensures the first case wins; subsequent triggers within the window are suppressed and logged as duplicates.
- **Diagnostician's own run fails**: The re-entrancy guard (FR-025) prevents the failure from triggering a new self-heal case.
- **Cost cap reached mid-pipeline**: The pipeline completes its current case (it's already in progress); NEW triggers are queued for the next day.
- **The fix requires a change to the Supervisor**: The Diagnostician MUST classify this as unfixable-by-self-heal (the Supervisor is off-limits per 005 FR-001/FR-010) and emit a notification to the user instead.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST expose a `self_heal.request` tool (available to agents, apps, and the user) that accepts a problem description and optional context (conversation id, event id, tool name + error, file path) and creates a Healing Case, invoking the Diagnostician.

- **FR-002**: The system MUST detect agent hard errors as a self-heal trigger. A hard error is a thrown exception from a tool call or agent run. The system MUST maintain an environmental allowlist (network errors, DNS failures, authentication errors, rate-limit responses, OOM kills, external service timeouts) and MUST NOT trigger for errors matching the allowlist.

- **FR-003**: The system MUST detect repeated tool-call failures as a self-heal trigger: when a configurable count (default 3) of consecutive calls to the same tool or tool class fail with the same error signature within a configurable time window (default 300s), the mechanism MUST be invoked with the correlated pattern.

- **FR-004**: The system MUST detect workflow and long-operation timeouts as a self-heal trigger: when a workflow run or long-running operation exceeds its configured timeout, the mechanism MUST be invoked with the timeout record (workflow id, node, configured vs. actual duration).

- **FR-005**: The system MUST detect error-level log events in BOS-owned component namespaces as a self-heal trigger. Events from external or third-party components MUST NOT trigger the mechanism.

- **FR-006**: Each trigger type (explicit, hard-error, repeated-failure, workflow-timeout, log-events) MUST be independently toggleable via the `selfHeal.triggers.*` settings. A global `selfHeal.enabled` kill switch MUST disable all triggers and scheduled execution simultaneously.

- **FR-007**: Upon trigger, the system MUST invoke the Diagnostician agent with the failure signature as input. The Diagnostician MUST produce a markdown diagnostics report containing: (a) YAML frontmatter with `scopeClass` (one of: `a`-env, `b`-skill, `c`-workflow, `d`-app-not-owned, `d-bis`-app-owned, `e`-bos-core), `ownership` (one of: `bos-core`, `user-app`, `marketplace`, `workflow`, `env`), and `proposedSurface` (the specific file/tool/skill/workflow to modify); (b) a human-readable investigation narrative with at least one source citation (file path + line, or spec reference) for every claim about BOS's current behavior; (c) a verdict: "genuine gap" (with the exact missing surface) or "usage/agent error" (with the correct invocation).

- **FR-008**: The Diagnostician MUST be the existing `conversation-reviewer` agent, extended with a second operational mode. Mode 1 (behavioral review of past conversations) and Mode 2 (gap diagnosis from failure signatures) share the same agent definition and the `agent-behavior-review` skill, selected by the input type. The Diagnostician MUST NOT write source code or delegate to other agents; its only write is the diagnostics report file.

- **FR-009**: For scope class `a` (environmental/transient), the system MUST close the case with status `env-only` and MUST NOT make any durable change. An optional user-facing suggestion MAY be included in the case record.

- **FR-010**: For scope class `b` (agent misuse / skill gap), the system MUST propose a specific skill patch or memory lesson as part of the diagnostics report. Application of the proposed change MUST be consent-gated: the user approves the specific edit before it is applied.

- **FR-011**: For scope class `c` (workflow/data gap), the system MUST propose a specific workflow definition edit. Application MUST be consent-gated.

- **FR-012**: For scope class `d` (marketplace app, not owned by the user — no local copy in `data/user-apps/items/`), the system MUST NOT modify the app. It MUST emit a notification event to the user describing the suspected bug and the app's identity.

- **FR-013**: For scope class `d-bis` (user-owned marketplace app — exists in `data/user-apps/items/<id>/`), the system MUST drive the fix through the Build Studio pipeline with the same autonomous execution as class `e`. The delivery mechanism MUST be `app_build` (not a BOS-source feature branch).

- **FR-014**: For scope class `e` (BOS core gap), the system MUST drive the fix through the Build Studio pipeline on a feature branch. The developer delegation MUST instruct test-driven development (write the failing test first, then implement) and target ≥95% line/branch coverage on the files modified. The test suite MUST pass before the `fix_ready` event is emitted.

- **FR-015**: The Build Studio pipeline for self-heal fixes MUST run autonomously from `specify` through `implement` without stopping at step boundaries. The escalation MUST carry an explicit pre-authorization instruction: "The user has pre-authorized this fix. Run the full pipeline autonomously. Stop only if you encounter a decision you cannot resolve autonomously."

- **FR-016**: When the autonomous pipeline encounters an unresolvable decision, it MUST emit a `self_heal.decision_needed` event (with the question, context, and case id) and the run MUST transition to suspended state. The run MUST resume when a matching `self_heal.decision_resolved` event arrives with the user's answer.

- **FR-017**: When a fix is complete and its preview (or `app_build` result) passes health checks and the test suite, the system MUST emit a `self_heal.fix_ready` event containing: the case id, the preview branch name (class e) or app id (class d-bis), a summary of the fix, and a link to the preview. The user MUST be able to act on this event (promote or discard).

- **FR-018**: The system MUST maintain a Healing Case record for each self-heal invocation. The case MUST track: id, trigger type, failure signature, status (state machine: `new → diagnosed → [scope-class-specific states] → terminal`), diagnostics report path, scope class, ownership, proposed surface, linked preview/app id, and a timeline of state transitions with timestamps.

- **FR-019**: The system MUST deduplicate triggers by failure signature within a configurable time window (`selfHeal.dedupeWindowSec`, default 86400s). A signature that already has an open or recently-closed case MUST NOT create a new case; the duplicate MUST be logged and linked to the original.

- **FR-020**: The system MUST enforce a daily cost cap (`selfHeal.costCapPerDay`): the total LLM token usage across all self-heal cases in a calendar day MUST NOT exceed the configured limit. When the cap is reached, new triggers MUST be queued (not dropped) and processed the following day.

- **FR-021**: The Diagnostician MUST be schedulable for periodic execution via `selfHeal.diagnostician.scheduled` (boolean) and `selfHeal.diagnostician.idleThresholdSec` (number, default 300). When scheduled, it MUST run both Mode 1 (behavioral review of conversations idle past the threshold) and Mode 2 (gap detection over the pending failure/event queue) in a single pass.

- **FR-022**: The Build Studio app MUST include a "Self-Heal" content page (accessible from the main BS navigation, alongside the git-merge page) that displays: (a) a list of all Healing Cases with their current status and scope class; (b) a detail view for each case showing the full diagnostics report, the proposed change, and the state-transition timeline; (c) consent controls for class b/c patches (approve specific edit / dismiss); (d) for class e/d-bis cases, a link to the preview and the current build status; (e) for suspended cases, the pending question and an answer input.

- **FR-023**: The self-heal mechanism MUST NOT modify the Supervisor (per 005 FR-001/FR-010) or the base version. All code fixes MUST land on feature branches as previews (class e) or as `app_build` results (class d-bis).

- **FR-024**: The self-heal mechanism MUST NOT auto-promote any fix. Promotion is always an explicit user action via the Topbar or the BS Self-Heal page.

- **FR-025**: The Diagnostician's own execution MUST be excluded from the trigger set (re-entrancy guard). A failing Diagnostician run MUST NOT create a new self-heal case.

- **FR-026**: The system MUST emit lifecycle events for all case state transitions via the 034 event system: `self_heal.case_created`, `self_heal.case_escalated`, `self_heal.decision_needed`, `self_heal.decision_resolved`, `self_heal.fix_ready`, `self_heal.abandoned`. These events MUST be queryable and provide a complete audit trail.

- **FR-027**: A `selfHeal` configuration namespace MUST exist in Settings, exposing: global enabled toggle, per-trigger toggles, Diagnostician scheduling (enabled + idle threshold), autonomous implement toggle, TDD required toggle, TDD target coverage (percentage), cost cap (tokens/day), dedupe window (seconds), and suspended timeout (days, for the HITL node).

- **FR-028**: The diagnostics report file MUST be written to `/Documents/BOS Improvements/` (the same directory as Mode 1 reports) with a filename derived from the case id. The report format is markdown with YAML frontmatter.

### Key Entities

- **Healing Case**: The durable record of one self-heal invocation. Tracks the full lifecycle from trigger through resolution. Fields: id, trigger type, failure signature, status (state machine), scope class, ownership, proposed surface, diagnostics report path, linked preview/app id, timeline.

- **Diagnostician**: The extended `conversation-reviewer` agent operating in Mode 2. Input: failure signature. Output: markdown diagnostics report + scope classification. Read-only except for the report file.

- **Diagnostics Report**: A markdown file with YAML frontmatter (scopeClass, ownership, proposedSurface, caseId, triggeredAt) and a human-readable body (investigation narrative, source citations, verdict, proposed fix description).

- **Failure Signature**: A normalized identifier for a failure pattern, used for deduplication. Derived from: tool name + error type + key error parameters (e.g., `bos_app_launch:missing-params`, `file_read:permission-denied:/path`).

- **Scope Class**: The classification of a diagnosed problem into one of six fix surfaces: `a` (env — no change), `b` (skill/agent — patch skill), `c` (workflow — edit workflow), `d` (app not owned — notify), `d-bis` (app owned — fix via app_build), `e` (BOS core — fix via feature branch).

- **Fast Spine**: The deterministic, fast-path workflow (trigger → diagnose → scope → resolve/escalate) that completes in minutes and ends without waiting for the slow path.

- **Slow Path**: The autonomous Build Studio pipeline (specify → implement → preview) that runs over hours/days for class e/d-bis fixes.

### Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The `bos_app_launch` params gap (the acceptance test case) is diagnosed, fixed, and delivered as a preview with a passing regression test and ≥95% coverage on the modified file, from trigger to `fix_ready` event, without any user intervention between the explicit trigger and the `fix_ready` emission.

- **SC-002**: A non-environmental tool error (e.g., a permission bug in `file_read`) is detected automatically (no explicit report), diagnosed, and resolved (or escalated) within 5 minutes of the error occurring.

- **SC-003**: An environmental error (e.g., a network timeout) does NOT trigger the self-heal mechanism — zero cases created, zero LLM tokens consumed.

- **SC-004**: A user-owned marketplace app bug is detected, classified `d-bis`, and delivered as an `app_build` preview (not a BOS-source branch).

- **SC-005**: A marketplace app bug where the app is NOT in `data/user-apps/` is classified `d` and results in a notification only — no modification attempt.

- **SC-006**: With `selfHeal.enabled` false, the mechanism consumes zero LLM tokens and creates zero cases regardless of trigger conditions.

- **SC-007**: The same failure signature does not create more than one case within the dedupe window (default 24h).

- **SC-008**: The autonomous pipeline completes without stopping (no `decision_needed` event) for an unambiguous class-e gap like the `bos_app_launch` case.

- **SC-009**: The BS Self-Heal page displays all active cases with their current status, and the user can approve a class-b skill patch, answer a suspended question, or link to a preview — all from that page.

- **SC-010**: No self-heal fix is ever promoted without an explicit user action. The system emits `fix_ready`; only the user promotes.

## Assumptions

- The 034 event-notification system is implemented and available (it is — converged 2026-08-24). The self-heal mechanism rides on its emit/query/get/ack infrastructure.
- The 005 self-modification pipeline (Supervisor, base + preview, promote/discard) is implemented and available (it is — the user has been using it for feature development).
- The `conversation-reviewer` agent and its `agent-behavior-review` skill exist and are functional (they are — the user built them). The extension adds Mode 2 without modifying Mode 1's behavior.
- The Build Studio pipeline (specify → clarify → design → plan → tasks → implement → converge) is available as a delegated agent (`build-studio`) that can be invoked programmatically with a pre-authorization instruction.
- The Workflow Manager service (with event-triggered runs, per spec 002) is available for the fast-spine workflow.
- The `data/user-apps/items/` directory is the authoritative location for user-owned marketplace items. An item's presence there (vs. only in the public marketplace) is the ownership predicate.
- The HITL node (suspended workflow state) is a v2 enhancement. In v1, the exception stop uses terminate-and-retrigger: the workflow ends when `decision_needed` is emitted, and a new workflow is triggered when `decision_resolved` arrives with the case context.
- The environmental allowlist for trigger filtering is a static, code-defined list (not user-configurable in v1). It covers: network/socket errors, DNS resolution failures, 401/403 auth errors, 429 rate-limit responses, OOM/SIGKILL, and external service timeouts (errors originating from outside BOS).
- The Diagnostician's report is the single source of truth for the scope classification. The pipeline does not re-classify; it trusts the report's `scopeClass` field.
- TDD and the 95% coverage target are instructions in the developer delegation brief, not a global change to the Developer agent's behavior. Other (non-self-heal) developer delegations are unaffected.
