# Tasks: Self-Healing Mechanism

**Branch**: `031-self-healing` | **Plan**: `plan.md` | **Design**: `design.md` | **Spec**: `spec.md`

**Format**: `[TaskID] [P?] [Story?] Description with file path`
**P** = parallelizable (different files, no dependency on incomplete tasks)
**Story** = user story from spec.md (US1–US7)

---

## Phase 1: Foundational — Spine Core & Usage Surfacing

*Blocking prerequisites for all user stories. No story label.*

### Usage Surfacing (ADR-5 M1)

- [ ] T001 Extend `TurnResult` in `src/lib/agent/agent-loop.ts` to carry an optional `usage?: { inputTokens, outputTokens, cacheReadTokens?, cacheWriteTokens? }` field
- [ ] T002 [P] Extend `AgentRunResult` in `src/lib/agent/subagents/types.ts` with an optional `usage?` field (same shape)
- [ ] T003 Extend `src/lib/assistant/model-turn.ts` to read `message_delta` usage (Anthropic) or `choices[0].usage` (OpenAI-compatible) from the response stream and attach it to the returned `TurnResult`
- [ ] T004 [P] Extend `src/lib/agent/subagents/claude-runner.ts` to parse `usage` / `total_tokens` from the stream-json `type:result` line (Claude) and the final event (OpenCode) and attach it to the returned `AgentRunResult`
- [ ] T005 [P] Extend `src/lib/agent/subagents/runner.ts` (`runLocalHeadless`) to accumulate per-turn usage across the loop and attach the total to the returned `AgentRunResult`
- [ ] T006 Write `tests/self-heal/usage-surface.test.ts` — a stubbed model turn returns usage; assert `TurnResult.usage` is populated; a claude-runner fixture stream-json result line is parsed correctly

### Spine Core (P1)

- [ ] T007 Create `src/lib/self-heal/types.ts` — `ScopeClass` (union: `a|b|c|d|d-bis|e`), `Ownership` (union: `bos-core|user-app|marketplace|workflow|env`), `CaseStatus` (union: `new|diagnosed|bs-pipeline|preview-ready|promoted|dismissed|failed|abandoned|suspended|env-only|patching|fixing|notified`), `HealingCase` interface, `FailureSignature` interface, `CostLedgerEntry` interface, `DedupeEntry` interface
- [ ] T008 [P] Create `src/lib/self-heal/signature.ts` — `computeFailureSignature(toolName, error)` returning `{ toolName, errorCategory, normalizedHash, dedupeKey, triggerType }`. Implement `errorCategory` mapping (timeout/not_found/permission_denied/type_mismatch/auth/unhandled_exception), message normalization (strip UUIDs, timestamps, quoted strings, numeric values; optional path-prefix strip), SHA-256 hash, and the relaxed explicit-trigger key
- [ ] T009 [P] Create `src/lib/self-heal/allowlist.ts` — `isEnvironmentalError(error)` (FR-002: network/socket, DNS, 401, 429, OOM/SIGKILL, external timeout; `permission_denied` NOT in) and `isBosOwnedLogComponent(component)` (R8: static allowlist of owned namespace prefixes)
- [ ] T010 Create `src/lib/self-heal/store.ts` — case store: `data/self-heal/index.json` (in-flight slot, dedupe map, cost ledger, slow-path queue, cost queue) + `data/self-heal/cases/<caseId>.json` (full record + timeline). Functions: `createCase`, `updateCase`, `appendTimeline`, `getCase`, `listCases`, `getDedupeEntry`, `setDedupeEntry`, `setInFlightSlot`, `clearInFlightSlot`, `getCostLedger`, `appendCostLedger`. Atomic writes via `@/os/atomic-write`
- [ ] T011 Write `tests/self-heal/store.test.ts` — atomic create/update, timeline append, dedupe-map read/write, in-flight slot set/clear, cost ledger append. Self-cleaning temp root

### Intake + Routing (P2)

- [ ] T012 Create `src/lib/self-heal/intake.ts` — `selfHealIntake(event, opts)` front door: read settings → disabled? no-op → reentrancy check → allowlist check (hard-error) → dedupe check → cost-cap check → create case → launch Diagnostician (fire-and-forget) → return ack. Plus `resolvePhaseB(caseId, report)` — scope-class router: a→close, b→consent, c→consent, d→notify, d-bis→escalate, e→escalate. Mutual exclusion check before escalation (FR-015c)
- [ ] T013 Create `src/lib/self-heal/reentrancy.ts` — `markSelfHealConversation(conversationId)`, `isSelfHealConversation(conversationId)`, `filterSelfHealEvents(events)`. The marker is stored on the conversation via `saveConversationMessages` (a meta entry or a field on the conversation record). The event filter checks `payload.selfHeal?.role`
- [ ] T014 Write `tests/self-heal/intake.test.ts` — the full front-door decision tree: disabled→no-op, reentrancy→no-op, allowlist→no-op, dedupe hit→suppressed event, cost-cap→queued, else→case created + Diagnostician launched. Phase-B routing for each scope class. Mutual exclusion: second escalation while in-flight → queued-slow. Self-cleaning temp root
- [ ] T015 Write `tests/self-heal/signature.test.ts` — error_category mapping for each exception type, normalization (UUID/timestamp/quoted/numeric stripping), SHA-256 determinism, the relaxed explicit key, path-prefix strip on/off
- [ ] T016 [P] Write `tests/self-heal/allowlist.test.ts` — env allowlist (401/429/timeout/OOM/DNS in, `permission_denied` NOT in, 403 NOT in), BOS-owned log component list

### Diagnostician (P3)

- [ ] T017 Create `src/lib/self-heal/diagnostician.ts` — `runDiagnostician(failureSignature, caseId)`: build the prompt (failure signature + raw data + instructions for Mode 2), call `runSubAgent` with `agent: "conversation-reviewer"`, `headless: true`, `tools: ["bos_source_search", "bos_source_read", "app_list", "query_events", "get_event", "submit_diagnostics_report"]`, `conversationId` (a fresh self-heal conversation, marked via `markSelfHealConversation`). Parse the returned report, write it to VFS `/Documents/BOS Improvements/<caseId>.md`, update the case with the report path + scope class + ownership + proposed surface. Return the parsed fields
- [ ] T018 [P] Create `src/lib/assistant/tools/server/diagnostics.ts` — `submit_diagnostics_report` server tool: accepts `{ caseId, scopeClass, ownership, proposedSurface, reportMarkdown }`. Writes the markdown to VFS (via `saveConversationMessages` or the fs client), returns confirmation. This is the Diagnostician's ONLY write
- [ ] T019 Modify `seed/agents/conversation-reviewer/AGENT.md` — add Mode 2 (gap diagnosis from failure signatures) to the description and system prompt. Add tools to the tool set: `app_list`, `query_events`, `get_event`. Add `submit_diagnostics_report` to the tool set. The unified tool set per C7. Keep Mode 1 (behavioral review) unchanged
- [ ] T020 [P] Extend the `agent-behavior-review` skill — add a "Mode 2: Gap Diagnosis" section to `SKILL.md` with the investigation method (reproduce → search specs/docs → verify in source → classify scope → write report). The report format: markdown with YAML frontmatter (scopeClass, ownership, proposedSurface, caseId, triggeredAt). The classification rules (a/b/c/d/d-bis/e) and the ownership predicate (data/user-apps/items/ check)
- [ ] T021 Write `tests/self-heal/diagnostician.test.ts` — `runDiagnostician` with a mocked `runSubAgent` returns a valid report; the VFS file is written; the case is updated with scope class. Self-cleaning temp root

### Consent Application (analyze fix F1 — FR-010, FR-011, FR-022c)

- [ ] T071 Create `src/lib/self-heal/consent.ts` — `applyApprovedEdit(caseId, edit)`: for scope class `b` (FR-010) apply the proposed skill patch via the existing skill-management mechanism; for class `c` (FR-011) apply the proposed workflow definition edit. MUST refuse to apply unless the case is in the consented `patching`/`fixing` state (consent-gated). On success move the case to a terminal closed state and emit the lifecycle event. Wired into intake Phase-B routing (T012) and the API route `?op=consent` (T032)
- [ ] T073 [P] Write `tests/self-heal/consent.test.ts` — `applyApprovedEdit` for class `b` actually changes the skill file (assert the diff applied); for class `c` changes the workflow def; a NOT-consented case results in NO change (the gate holds). Self-cleaning temp root

### Cost + Queues (P4)

- [ ] T022 Create `src/lib/self-heal/cost.ts` — `recordCaseCost(caseId, usage)`, `capExhaustedForToday()`, `getDayKey()` (UTC midnight), `estimateCost(task)` (fallback when usage is undefined). Ledger is in `data/self-heal/index.json`. Reset at midnight UTC
- [ ] T023 [P] Create `src/lib/self-heal/queue.ts` — `enqueueCost(caseId)`, `dequeueCost()` (FIFO, max 100, 7-day TTL, emit `cost_cap_evicted` on eviction), `enqueueSlow(caseId)`, `dequeueSlow()` (FIFO, no dedupe re-check on re-entry), `sweepSuspendedTimeouts()` (emit `abandoned` for cases suspended > N days). Bounded queues in `data/self-heal/index.json`
- [ ] T024 Write `tests/self-heal/cost.test.ts` — ledger append, capExhaustedForToday, midnight-UTC reset, undefined-usage estimate fallback
- [ ] T025 [P] Write `tests/self-heal/queue.test.ts` — bounded FIFO (max 100, FIFO eviction), 7-day TTL eviction + `cost_cap_evicted` event, slow-path dequeue-on-slot-free (re-entry, no dedupe re-check), suspended-timeout sweep → abandoned

### Spine Handler + Boot Wiring

- [ ] T026 Create `src/lib/self-heal/spine-handler.ts` — `registerSelfHealSpine()`: calls `registerCoreExecutor("self_heal.request", "core:self-heal", handler)` at boot. The handler calls `selfHealIntake(event)`, returns a minimal ack immediately (Diagnostician is fire-and-forget). Also registers handlers for `self_heal.decision_resolved` (resume) and `self_heal.fix_ready` (notification)
- [ ] T027 [P] Modify `src/instrumentation.ts` — at boot, after the event kernel and Supervisor are up: (a) call `registerSelfHealSpine()`, (b) register the `self-heal` plugin, (c) register the scheduled Diagnostician job (if `selfHeal.diagnostician.scheduled` is true), (d) run boot-reconcile (R9: single-owner via scheduler daemon lock, recover in-flight cases)

---

## Phase 2: US1 — Explicit Report → Autonomous Fix → `fix_ready` (P1)

*The headline end-to-end loop. Independent test: SC-001.*

- [ ] T028 [US1] Create `src/lib/assistant/tools/server/self-heal.ts` — three server tools: `self_heal.request` (FR-001: accepts problem description + optional context, calls `selfHealIntake`), `self_heal.request_decision` (FR-016: accepts caseId + question, emits `decision_needed`, suspends the case), `self_heal.complete_fix` (FR-017: accepts caseId + branch/appId + summary, checks Supervisor preview state, emits `fix_ready`, clears in-flight slot)
- [ ] T029 [US1] Implement branch pre-creation in `src/lib/self-heal/intake.ts` (FR-015b): before the BS conversation's first token, set the BS conversation's `activeFeatureBranch` to the deterministic name `bos/self-heal-<caseId-lowercase>` — do NOT eagerly create the git ref; it materializes lazily in `supervisorBegin` at the first `dev_delegate` (design.md ADR-3, reviewer-verified). The Diagnostician's report is passed as **user intent** — the BS agent's `specify` step writes a proper `spec.md` from it
- [ ] T030 [US1] Implement the autonomous brief in `src/lib/self-heal/intake.ts`: construct the `runSubAgent` task for the BS agent that includes (a) the pre-authorization instruction ("The user has pre-authorized this fix. Run the full pipeline autonomously. Stop only if you encounter a decision you cannot resolve autonomously."), (b) the FR-015a classification-verification instruction (one re-diagnosis; a second disagreement is never LLM-arbitrated — emit `decision_needed`), (c) the FR-014 mandate in the developer delegation brief — TDD (failing test first) + ≥95% coverage on modified files + the `plan` step's file list as a **hard scope constraint** (modify only those files; emit `decision_needed` for any addition; the `converge` step verifies modified ⊆ plan list), (d) the FR-015 **commit-before-advance** rule (persist each user-confirmed decision to the current artifact before advancing), (e) the Diagnostician's report as the `specify` input
- [ ] T031 [US1] Implement boot-reconcile in `src/lib/self-heal/intake.ts` (R9): on boot (single-owner via scheduler daemon lock), find cases in `bs-pipeline` state with no live conversation → check Supervisor for a `ready` preview on the case's branch → if found, re-emit `fix_ready` (idempotent); if not, mark case `failed`
- [ ] T032 [P] [US1] Create `src/app/api/self-heal/route.ts` — API route: `GET` (list cases, optional `?caseId` for detail), `POST ?op=report` (fire `self_heal.request`), `POST ?op=consent` (approve/reject class-b/c patch), `POST ?op=answer` (submit `decision_resolved`), `POST ?op=dismiss` (close case). All server-only; the client fetches over HTTP
- [ ] T033 [P] [US1] Create `src/apps/build-studio/selfheal/useSelfHealCases.ts` — React hook that fetches from `/api/self-heal` and subscribes to `self_heal.*` events (via the 034 stream) for live updates
- [ ] T034 [US1] Create `src/apps/build-studio/selfheal/SelfHealPane.tsx` — the top-level pane (peer of the conflict pane), with a "Report a problem" button (C1: text input + optional conversation picker) and the case list
- [ ] T035 [US1] Create `src/apps/build-studio/selfheal/CaseList.tsx` — table with columns: Status | Case | Trigger | Scope Class (badge). Active/in-flight first. Click a row → detail view
- [ ] T036 [US1] Create `src/apps/build-studio/selfheal/CaseDetail.tsx` — diagnostics report (markdown rendered), header strip (scope class, ownership, proposed surface, status), state-transition timeline (vertical, timestamped), and the scope-class-dependent action area
- [ ] T037 [US1] Create `src/apps/build-studio/selfheal/PreviewStatus.tsx` — the class-e/d-bis action card: build state (building/ready/failed), branch name or app id, "Pin & open preview" button, build log tail, note: "Promote stays in Topbar (FR-024)"
- [ ] T072 [US1] Create `src/apps/build-studio/selfheal/ConsentCard.tsx` — the class-b/c action card (FR-022c): the proposed edit rendered as a before/after diff, with **Approve edit** (calls `/api/self-heal?op=consent` → `applyApprovedEdit`, T071) and **Dismiss** buttons. Rendered from `CaseDetail` (T036) when the case is in the consented/`patching` state
- [ ] T038 [US1] Modify `src/apps/build-studio/index.tsx` — add "Self-Heal" to the nav (peer of the conflict/git-merge page). Render `SelfHealPane` when selected
- [ ] T039 [P] [US1] Modify `src/apps/build-studio/manifest.ts` — add `eventHandler` entries for `self_heal.fix_ready` (deep-link to the case, show a toast) and `self_heal.decision_needed` (highlight the suspended case)
- [ ] T040 [US1] Write `e2e/031-self-healing.spec.ts` — **SC-001 acceptance test**: fire `self_heal.request` with the `bos_app_launch`-missing-params signature → assert Diagnostician writes a markdown report classifying `e` with a source citation → case escalates and a BS conversation is seeded with `activeFeatureBranch` = `bos/self-heal-<id>` → (mock the developer step to a committed `params` fix on the branch) → preview health passes → `fix_ready` event emitted with the branch name → the Event Viewer shows it. **Regression test that fails on base**: a test asserting `bos_app_launch` accepts a `file` param must fail before the fix and pass after. **Consent (SC-009)**: a class-b case renders `ConsentCard`; approving it applies the skill edit (assert the file changes) and the case closes. `test.describe.configure({mode:'serial'})`

---

## Phase 3: US2 — Automatic Detection of Tool Failures (P1)

*Independent test: SC-002, SC-003, SC-007.*

- [ ] T041 [US2] Create `src/plugins/self-heal/index.ts` and `src/plugins/self-heal/init.ts` — the tool-error capture plugin: `afterToolCall` hook (capture successful calls for the rolling window), `onError` hook (capture tool errors). For hard-error trigger: check `isEnvironmentalError` → if not env, compute `FailureSignature`, call `selfHealIntake`. For repeated-failure trigger: maintain a 300s/3-count rolling window per tool+signature; on threshold, call `selfHealIntake` with the correlated pattern
- [ ] T042 [US2] Register the `self-heal` plugin in the plugin system (via `init.ts` export, wired in `src/instrumentation.ts` — T027 already covers the wiring; this task is the plugin's own registration logic)
- [ ] T043 [P] [US2] Modify `src/lib/agent/capabilities-registry.ts` — add capabilities for `self_heal.request`, `self_heal.request_decision`, `self_heal.complete_fix`, `submit_diagnostics_report`
- [ ] T044 [P] [US2] Modify `src/lib/agent/tool-manifest.ts` — mirror the four new server tools (FR-001, FR-016, FR-017, FR-028)
- [ ] T045 [US2] Add to `e2e/031-self-healing.spec.ts` — **SC-003**: simulate a network-timeout tool error → assert zero cases, zero `case_created`. **SC-007**: fire the same non-env signature twice within the window → one case, one `dedupe_suppressed` event. **SC-006**: `selfHeal.enabled=false` → no trigger fires, zero cases

---

## Phase 4: US3 — Workflow Timeout + Log Events → Correct Scoping (P2)

*Independent test: US3 acceptance scenarios (workflow timeout → class c or d/d-bis).*

- [ ] T046 [US3] Implement the workflow-timeout trigger in `src/lib/self-heal/spine-handler.ts` — subscribe to 002's timeout event (tolerant of a missing event per R7: if 002 doesn't emit, the trigger stays inert). On receipt: extract workflow id, node, configured-vs-actual duration, compute `FailureSignature`, call `selfHealIntake`
- [ ] T047 [P] [US3] Implement the log-events trigger in `src/lib/self-heal/spine-handler.ts` — subscribe to 017's error-level log events. Filter: `isBosOwnedLogComponent(component)` (R8 allowlist). Compute `FailureSignature`, call `selfHealIntake`
- [ ] T048 [US3] Implement scope-class `c` routing in `src/lib/self-heal/intake.ts` (FR-011): the Diagnostician's report proposes a specific workflow definition edit. Application is consent-gated: the case enters `patching` state, the BS Self-Heal page shows the proposed edit with Approve/Dismiss buttons
- [ ] T049 [P] [US3] Implement scope-class `d` routing in `src/lib/self-heal/intake.ts` (FR-012): emit a notification event to the user describing the suspected bug and the app's identity. Case closes as `notified`. No modification
- [ ] T050 [US3] Add to `e2e/031-self-healing.spec.ts` — workflow timeout: create a workflow with an inevitably-timeout node, run it, assert the timeout trigger fires, the Diagnostician classifies `c`, and a workflow edit is proposed (not an app change)

---

## Phase 5: US4 — User-Owned Marketplace App Bugs (P2)

*Independent test: SC-004, SC-005.*

- [ ] T051 [US4] Implement the ownership predicate in `src/lib/self-heal/diagnostician.ts` — before classification, check `data/user-apps/items/<id>/` (via `listInstalledItems` or the fs client). Present → `d-bis` (user-owned, fixable). Absent → `d` (not owned, notify). The Diagnostician's prompt includes this check as a required step
- [ ] T052 [US4] Implement scope-class `d-bis` routing in `src/lib/self-heal/intake.ts` (FR-013): same autonomous BS pipeline as class `e`, but the delivery mechanism is `app_build` (not a BOS-source feature branch). The branch pre-creation (T029) is skipped for `d-bis` — no feature branch needed; the fix lands in the item's staging directory
- [ ] T053 [US4] Add to `e2e/031-self-healing.spec.ts` — **SC-004**: a failure signature pointing at an item present in `data/user-apps/items/` → `d-bis` → `app_build` preview (not a BOS-source branch). **SC-005**: the same signature with no local item → `d` → notification only, no modification

---

## Phase 6: US5 — Settings → Self Improvement (P2)

*Independent test: SC-006 (kill switch), US5 acceptance scenarios.*

- [ ] T054 [US5] Create `src/lib/self-heal/config.ts` — the `selfHeal` `ConfigRegistration` (FR-027): `enabled` (bool, default true), `triggers.explicit` (bool, default true), `triggers.hardError` (bool, default false), `triggers.repeatedFailure` (bool, default false), `triggers.workflowTimeout` (bool, default false), `triggers.logEvents` (bool, default false), `diagnostician.scheduled` (bool, default false), `diagnostician.idleThresholdSec` (number, default 300), `autonomousImplement` (bool, default true), `tdd.required` (bool, default true), `tdd.targetCoverage` (number, default 95), `costCapPerDay` (number, default 1_000_000), `dedupeWindowSec` (number, default 86_400), `suspendedTimeoutDays` (number, default 7), `costQueueMax` (number, default 100), `costQueueTtlDays` (number, default 7)
- [ ] T055 [P] [US5] Create `src/components/apps/settings/SelfImprovementTab.tsx` — the Settings → Self Improvement tab (FR-027). Two-column layout per the mockup: left = Global (master toggle) + Triggers (5 toggles with one-line descriptions); right = Autonomy (2 toggles + coverage number) + Limits (cost cap, dedupe window, suspended timeout, queue max/TTL). Dim all controls when `enabled` is false
- [ ] T056 [US5] Modify `src/apps/settings/index.tsx` — add "self-improvement" → `SelfImprovementTab` to `CUSTOM_TABS`
- [ ] T057 [US5] Add to `e2e/031-self-healing.spec.ts` — **SC-006**: `selfHeal.enabled=false` → no trigger fires, zero cases, zero LLM tokens. Toggle back on, enable hard-error, simulate a non-env error → mechanism fires. Cost cap: set `costCapPerDay` below the day's usage → new triggers are queued, not dropped

---

## Phase 7: US6 — Scheduled Diagnostician (P3)

*Independent test: US6 acceptance scenarios (scheduled review, both modes).*

- [ ] T058 [US6] Implement the scheduled Diagnostician job in `src/lib/self-heal/diagnostician.ts` — a recurring scheduler job (`owner: "self-heal"`, registered at boot in T027) that, when `selfHeal.diagnostician.scheduled` is true: (a) enumerates conversations idle past `idleThresholdSec` (from the conversation store, not a raw VFS glob — R6), (b) runs Mode 1 (behavioral review) on each, (c) runs Mode 2 (gap detection) over the pending failure/event queue, (d) writes reports, creates cases for actionable findings. Both modes in a single pass (FR-021)
- [ ] T059 [US6] Add to `e2e/031-self-healing.spec.ts` — scheduled review: enable `selfHeal.diagnostician.scheduled`, wait for the idle threshold, assert the Diagnostician runs, produces markdown reports, and creates cases for actionable findings. "No issues found" → no case created

---

## Phase 8: US7 — Exception Stop → Suspend → Resume (P3)

*Independent test: SC-009 (consent/decision), US7 acceptance scenarios.*

- [ ] T060 [US7] Implement the suspension state in `src/lib/self-heal/store.ts` — add `suspended` to `CaseStatus`. The case record gains `suspendedAt` (timestamp) and `pendingQuestion` (the question markdown). The suspended-timeout sweep (T023) reads these
- [ ] T061 [US7] Implement the resume mechanism in `src/lib/self-heal/intake.ts` — on `decision_resolved` event (subscribed in T026): find the suspended case by id, append the answer to the pending question, re-enter the BS conversation with the answer (a new `runSubAgent` with the case context + the answer), the BS agent picks up from the last artifact on disk (cold-restart recovery per FR-015 commit-before-advance)
- [ ] T062 [P] [US7] Create `src/apps/build-studio/selfheal/SuspendedCard.tsx` — the suspended action card: amber border + pulse, "Waiting on you" header, the pending question (markdown), a free-text answer input, "Submit answer" button (emits `decision_resolved` via `/api/self-heal?op=answer`)
- [ ] T063 [US7] Add the suspended case to `CaseDetail.tsx` (T036) — when the case status is `suspended`, render `SuspendedCard` instead of the preview/consent cards
- [ ] T064 [US7] Add to `e2e/031-self-healing.spec.ts` — **SC-009**: a suspended case shows the decision card on the BS page; submitting an answer emits `decision_resolved` and resumes the pipeline. Suspended timeout: a case suspended > `suspendedTimeoutDays` → `abandoned` event emitted, case closed

---

## Phase 9: Polish & Cross-Cutting Concerns

- [ ] T065 [P] Write `docs/dev/self-healing/self-healing.md` — how the mechanism works for maintainers: the fast spine, the case store, the scope classes, the Diagnostician, the slow path, the integration points (034, 005, 003, 002), the boot-reconcile, the re-entrancy guard. Reference `design.md` ADRs
- [ ] T066 [P] Write `docs/usage/self-healing.md` — user-facing: what the mechanism does, the 5 triggers, the BS Self-Heal page, Settings → Self Improvement, how to promote/discard a fix, the `self_heal.request` tool
- [ ] T067 Write `tests/self-heal/reentrancy.test.ts` — the Diagnostician's own execution is excluded from the trigger set: a failing Diagnostician run does NOT create a new self-heal case (FR-025). The self-heal conversation marker is set and read correctly. The event-payload filter works
- [ ] T068 Write `tests/self-heal/integration-spine-handoff.test.ts` — the FR-015b precondition: seed a BS conversation + set `activeFeatureBranch` → assert the field is set BEFORE any agent turn. `self_heal.complete_fix` reads a `ready` Supervisor preview state and emits `fix_ready`
- [ ] T069 Run the full e2e suite (`e2e/031-self-healing.spec.ts`) — all 10 SCs (SC-001 through SC-010) pass. `test.describe.configure({mode:'serial'})`. All tests self-cleaning
- [ ] T070 Run coverage on `src/lib/self-heal/**` — assert ≥95% line+branch. If below, add targeted tests for the uncovered branches

---

## Dependencies

```
T001–T006 (usage)  ──┐
T007–T011 (core)     ──┤
T012–T016 (intake)   ──┤
T017–T021 (diag)     ──┤
T022–T025 (cost)     ──┤
T026–T027 (boot)     ──┤
T071+T073 (consent)  ──┘──► T028–T040 (US1) ──► T041–T045 (US2) ──► T046–T050 (US3)
                               + T072 (US1)        │
                                                    ├──► T051–T053 (US4)
                                                    ├──► T054–T057 (US5)
                                                    ├──► T058–T059 (US6)
                                                    ├──► T060–T064 (US7)
                                                    └──► T065–T070 (Polish)
```

**Foundational (T001–T027, plus the consent application T071/T073) MUST complete before any user story.**
**US1 (T028–T040) MUST complete before US2–US7** (the BS page + API route are shared infrastructure).
**US2–US7 are independent of each other** (they can be developed in parallel once US1 is done).
**Polish (T065–T070) is last.**

## Parallel Opportunities

- **T001 + T002**: different files (`agent-loop.ts` vs `types.ts`), no dependency
- **T004 + T005**: `claude-runner.ts` vs `runner.ts`, independent
- **T008 + T009**: `signature.ts` vs `allowlist.ts`, independent
- **T016**: `allowlist.test.ts` is independent of T015 (`signature.test.ts`)
- **T025**: `queue.test.ts` is independent of T024 (`cost.test.ts`)
- **T027**: `instrumentation.ts` wiring is independent of T026 (the handler logic)
- **T032 + T033**: API route vs React hook, different files
- **T039 + T043 + T044**: manifest, capabilities, tool-manifest — three independent edits
- **T047**: log-events trigger is independent of T046 (workflow-timeout)
- **T049**: class-d routing is independent of T048 (class-c)
- **T055**: Settings tab is independent of T054 (config registration)
- **T062**: SuspendedCard is independent of T061 (resume logic)
- **T065 + T066**: two docs, independent

## Implementation Strategy

**MVP = Foundational (T001–T027 + T071/T073) + US1 (T028–T040 + T072).** This delivers the headline capability: the explicit `self_heal.request` → Diagnostician → autonomous BS pipeline → TDD fix → `fix_ready` event, with the BS Self-Heal page, the class-b/c consent flow (approve a skill/workflow edit), and the `bos_app_launch` acceptance test. The user can report a problem and get a fix on a preview with one click.

---

# Scope-Add (2026-09-08): Headless Transcription + Stuck-Run Stop/Start

**On top of the base feature (T001–T073, already implemented). Plan `plan.md` §"Scope-Add" phases P11–P16; `design.md` §9 (ADR-10–14). Spec US8/9/10, FR-029–FR-035.**
**Story** labels now include US8/US9/US10. TDD-first throughout.

### P11 — Platform Transcription (FR-029/FR-030) [US8]

- [ ] T074 [US8] Create `src/lib/agent/subagents/transcript.ts` — `TranscriptionWriter`: `open(agentId, runId, taskInput)`, `appendAssistantText(text)`, `appendReasoning(text)`, `appendToolCall(name, input)`, `appendToolResult(name, result, ok)`, `finalize({endedAt, aborted})`. Writes to `data/agent-transcripts/<agentId>/<runId>.md` (via `dataDir()`). YAML frontmatter (agentId, runId, startedAt, endedAt, aborted, caseId?) + markdown body. Append-per-turn (fs.appendFile), atomic frontmatter on open. Config-gated: `transcriptEnabled()` no-ops the whole writer when `agentRuns.transcriptions.enabled` is false
- [ ] T075 [US8] Extend `AgentRunResult` in `src/lib/agent/subagents/types.ts` — add `runId: string` (required, generated by each runner), `endedReason?: "completed" | "cancelled" | "error" | "max_steps"`, `aborted?: boolean`. Extend `SubAgentEvent` — add `{ type: "run_started", runId: string, agentId: string, startedAt: string, parentRunId?: string }` (the first event a headless run emits). **Coordinate with existing T002 `usage?` — additive, same file**
- [ ] T076 [US8] Modify `src/lib/agent/subagents/runner.ts` (`runLocalHeadless`) — (1) accept `opts.runId` else generate `headless-<agentId>-<Date.now()>`; (2) emit `run_started` first (via both the internal `emit` AND `opts.onEvent`); (3) instantiate a `TranscriptionWriter` and wire it into the existing `emit` handler (append assistant text / reasoning / tool calls / results as they arrive; finalize in the existing `finally`); (4) surface the loop's end reason as `result.endedReason` on the non-error return (M1: `completed`/`cancelled`/`max_steps` — today only `error` is special-cased); (5) set `result.aborted = (reason === "cancelled")`; (6) return `runId` on the result. **Behavior for every existing caller is unchanged except the new additive fields**
- [ ] T077 [US8] Modify `src/lib/agent/subagents/claude-runner.ts` — generate `runId` at the TOP of `runClaudeAgent` (before all early-return/refusal sites) and thread it into `runClaudeCli`/`runOpenCodeCli`/`runViaMcp` (each refusal return gains the now-required `runId` — S2). Emit `run_started`. Wire a `TranscriptionWriter` into the existing stdout `tool_use`/`tool_result` parsing (append tool calls + results + final text; finalize on close). Set `result.endedReason` (`completed`/`error` — no step loop, so never `max_steps`) + `result.aborted` on the kill path. Return `runId`
- [ ] T078 [US8] Add `agentRuns` config namespace in `src/lib/config/registry.ts` — `agentRuns.transcriptions.enabled` (boolean, default true) + the `transcriptEnabled()` reader (used by `transcript.ts`)
- [ ] T079 [P] [US8] Create `src/app/api/agent-transcripts/route.ts` — read-only API: `GET ?runId` (return the markdown file as text), `GET ?agentId` (list `<runId>.md` filenames for the agent), `GET ?caseId` (delegate to the self-heal case store's `runs[]` to list the case's transcripts). No writes
- [ ] T080 [US8] Write `tests/agent/transcript.test.ts` — a stubbed local run (direct `TranscriptionWriter` call sequence) produces the documented markdown format with correct frontmatter + body; disabled config → no file created; an aborted finalize → file marked `aborted` with a partial body. Self-cleaning temp root

### P12 — Platform Abort (FR-032) [US10]

- [ ] T081 [US10] Create `src/lib/agent/subagents/run-registry.ts` — module-level `Map<runId, { abort: () => void, parentRunId?: string }>`. `registerRun(runId, {abort, parentRunId?})`, `unregisterRun(runId)`, `abortHeadlessRun(runId): boolean` — looks up the run, if it has a `parentRunId` chain (nested child), aborts the parent first (cascades to all children); calls `abort()`. Returns true if the run existed and was aborted, false if unknown. `abortAllChildren(parentRunId)` for the cascade
- [ ] T082 [US10] Modify `src/lib/agent/subagents/runner.ts` (`runLocalHeadless`) — create the `AbortController` BEFORE the loop (currently created inline); register it in `run-registry.ts` on start (`registerRun(runId, { abort: () => controller.abort() })`); clear in the existing `finally` (`unregisterRun(runId)`) alongside the existing `setInRunAgent(null)`
- [ ] T083 [US10] Modify `src/lib/agent/subagents/claude-runner.ts` — after spawning the `child`, register it in `run-registry.ts` (`registerRun(runId, { abort: () => { child.kill("SIGTERM"); /* SIGKILL escalation timer */ }, parentRunId: opts.parentRunId })`); clear on `child.on("close")`. Add a 5s SIGKILL escalation: if the child hasn't exited within 5s of SIGTERM, send SIGKILL
- [ ] T084 [US10] Modify `src/lib/agent/capabilities-registry.ts` + `src/lib/agent/tool-manifest.ts` — add the `abort_headless_run` capability entry (it is a server-side platform function, not a tool exposed to agents in v1 — but it MUST be callable from the self-heal API route)
- [ ] T085 [US10] Write `tests/agent/run-registry.test.ts` — register a run with a mock abort; `abortHeadlessRun` calls the abort and returns true; aborting an unknown runId returns false (no-op); a parent with a registered child: `abortHeadlessRun(parentId)` cascades to the child (both abort functions called); `unregisterRun` removes the entry (no leak). Self-cleaning

### P13 — `endedReason` Surfacing (M1 — FR-033(b) enablement) [US10]

- [ ] T086 [US10] Modify `src/lib/self-heal/intake.ts` — `PipelineRunOutput` gains `endedReason?: "completed"|"cancelled"|"error"|"max_steps"` + `aborted?: boolean`; `DEFAULT_HOOKS.runPipeline` threads `result.endedReason` + `result.aborted` through from the `runSubAgent` call's returned `AgentRunResult`
- [ ] T087 [US10] Modify `src/lib/self-heal/diagnostician.ts` — `runDiagnostician` captures the `AgentRunResult` from `runSubAgent`; passes `endedReason` + `aborted` to the case-update / run-end handler (so a max-steps Diagnostician run is also handled, though the primary case is the pipeline run)

### P14 — Self-Heal Stuck Detector + Stop/Start + `stopped` (FR-033/FR-034/FR-035) [US10]

- [ ] T088 [US10] Create `src/lib/self-heal/stuck-detector.ts` — `StuckDetector` class (one instance per in-flight self-heal run): `onToolCall(tool: string, input: unknown)` — normalizes input using the `signature.ts` normalization (reuse: strip IDs/timestamps/quoted strings/numerics); tracks `(normalizedToolInput) → count`; `isStuck(): { stuck: boolean, reason: "repeat-calls"|"max-steps", repeatedCall?: string, count?: number }` (returns repeat-calls when count ≥ `selfHeal.stuckDetector.repeatCalls` with no intervening distinct call); `onRunEnd(endedReason)` — returns stuck if `endedReason === "max_steps"`. Pure, deterministic, no LLM
- [ ] T089 [US10] Modify `src/lib/self-heal/types.ts` — `CaseStatus` union: add `"stopped"`. `HealingCase`: add `runs: CaseRun[]` (one per linked headless run: `{ runId, agentId, role: "diagnostician"|"pipeline", status: "in-flight"|"completed"|"aborted"|"stopped", startedAt, endedAt? }`) and `stuckSignature?: { tool: string, normalizedInput: string, count: number, runId: string }` (set when the detector fires)
- [ ] T090 [US10] Modify `src/lib/self-heal/store.ts` — `appendRun(caseId, run: CaseRun)`; `updateRunStatus(caseId, runId, status)`; `setStuckSignature(caseId, sig)`; `clearInFlightSlot` is now also called on `stopped` (not just terminal states) — add an explicit `releaseInFlightSlot(caseId)` that the Stop handler calls
- [ ] T091 [US10] Modify `src/lib/self-heal/intake.ts` — `stopRun(caseId)`: find the in-flight run (from `case.runs`), call `abortHeadlessRun(runId)`, `updateRunStatus(caseId, runId, "stopped")`, `releaseInFlightSlot(caseId)`, emit `self_heal.run_aborted`, update case status to `stopped` in the timeline. `startRun(caseId)`: re-acquire in-flight slot, update case status back to `bs-pipeline`, launch a fresh `runSubAgent` for the pipeline run (same task/brief as the original — the commit-before-advance rule means the last artifact on disk is the resume point), record the new run in `case.runs`, emit `self_heal.run_restarted`. `onPipelineRunEvent(caseId, ev)` (new hook): when `ev.type === "run_started"`, record the run; when `ev.type === "tool_call"`, feed it to the `StuckDetector`; if the detector fires, `setStuckSignature` + emit `self_heal.run_stuck` + flag the case. Wire `onPipelineRunEvent` into `DEFAULT_HOOKS.runPipeline`'s `onEvent` callback. `onDiagnosticianRunEvent` (new hook): same pattern for the Diagnostician run. Thread `endedReason === "max_steps"` from the run-end handler → stuck (max-steps) → `stopped` + `run_stuck` (M1: this is the FR-033(b) path — no abort needed, the run already ended, transcript is complete)
- [ ] T092 [US10] Modify `src/lib/self-heal/diagnostician.ts` — pass `onEvent` through to `runSubAgent` (the diagnostician seam — ADR-12); on `run_started`, record the run in `case.runs`; feed `tool_call` events to the `StuckDetector`; on run-end with `endedReason === "max_steps"`, flag stuck + `stopped`
- [ ] T093 [US10] Modify `src/app/api/self-heal/route.ts` — add `POST ?op=stop` (body: `{caseId}`) → calls `stopRun(caseId)`; `POST ?op=start` (body: `{caseId}`) → calls `startRun(caseId)`. `GET ?caseId` response now includes `runs[]` + `stuckSignature` if set
- [ ] T094 [P] [US10] Add to `src/lib/config/registry.ts` — `selfHeal.stuckDetector.enabled` (boolean, default true) + `selfHeal.stuckDetector.repeatCalls` (number, default 5) to the `SELF_HEAL_CONFIG_KEYS` list + shape
- [ ] T095 [US10] Write `tests/self-heal/stuck-detector.test.ts` — same `(tool, normalizedInput)` called 5× (default `repeatCalls`) with no intervening distinct call → `isStuck()` returns `{stuck: true, reason: "repeat-calls"}`; distinct calls → not stuck; `onRunEnd("max_steps")` → `{stuck: true, reason: "max-steps"}`; `onRunEnd("completed")` → not stuck. Deterministic — no LLM mock needed
- [ ] T096 [US10] Write `tests/self-heal/stopped-state.test.ts` — `stopRun` on an in-flight case: abort is called, run status → `stopped`, in-flight slot released, `run_aborted` event emitted, case status → `stopped`; `startRun` on a `stopped` case: slot re-acquired, fresh run launched, run recorded, `run_restarted` emitted, case → `bs-pipeline`; a stuck run does NOT create a new case (FR-025/FR-035); `stopped` is not terminal: boot-reconcile leaves a `stopped` case alone. Self-cleaning
- [ ] T097 [US10] Write `tests/self-heal/integration-partial-build.test.ts` — (R-SA5 / S3): simulate a pipeline run that was aborted mid-`dev_delegate` (the candidate is staged but the build was interrupted). Assert `previewStateFor(branch) !== "ready"` — the partial build does NOT mark the preview ready, so an aborted→`stopped` case cannot emit a false `fix_ready`. Self-cleaning

### P15 — BS Pane + Manifest + Settings (US9/US10 UI, FR-022(f)/(g)) [US9/US10]

- [ ] T098 [US9] Create `src/apps/build-studio/selfheal/TranscriptsSection.tsx` — the "Transcripts" section for the case-detail view: lists `case.runs` (each: agent name, run id, role label ("Diagnostician"/"Pipeline"), status chip, a "View transcript" button); a `● live` badge on any `in-flight` run; clicking a run fetches `/api/agent-transcripts?runId=<id>` and renders the markdown in a scrollable `<pre>` panel (monospace, the default run is the most-recent in-flight or most-recent completed); "no transcription (disabled)" empty state when the file is absent
- [ ] T099 [US10] Create `src/apps/build-studio/selfheal/RunControl.tsx` — the run-control area: when `case.stuckSignature` is set, show the amber stuck indicator ("⚠ Stuck — repeated [tool] ×[count] with no progress"); when the case is in-flight (`bs-pipeline`): show the Stop button (amber, `■ Stop`, note: "Kills the run; case → stopped"); when the case is `stopped`: show the Start button (green, `▶ Start`, note: "Relaunches a fresh run from the last committed artifact"). Stop calls `/api/self-heal?op=stop`; Start calls `/api/self-heal?op=start`
- [ ] T100 [US9] Modify `src/apps/build-studio/selfheal/CaseDetail.tsx` — wire in `TranscriptsSection` (below the timeline) and `RunControl` (in the action area, above the scope-class-specific cards). `RunControl` is visible when `case.status` is `bs-pipeline` or `stopped`; `TranscriptsSection` is always visible (shows runs that have started)
- [ ] T101 [US10] Modify `src/apps/build-studio/selfheal/CaseList.tsx` — add the `stopped` status chip (muted gray-amber, distinct from amber `suspended` and red `failed`). The status-to-chip map: `stopped` → neutral/amber-gray, `suspended` → amber (existing), `failed` → red (existing)
- [ ] T102 [P] [US9/US10] Modify `src/apps/build-studio/selfheal/useSelfHealCases.ts` — subscribe to `self_heal.run_stuck`, `self_heal.run_aborted`, `self_heal.run_restarted` events; on any of these, refetch the affected case's detail (the runs + stuckSignature + status may have changed)
- [ ] T103 [P] [US10] Modify `src/apps/build-studio/manifest.ts` — add `eventHandler` entries for `self_heal.run_stuck` (toast: "Self-heal run [caseId] appears stuck"), `self_heal.run_aborted` (toast: "Self-heal run stopped"), `self_heal.run_restarted` (toast: "Self-heal run restarted")
- [ ] T104 [US10] Modify `src/components/apps/settings/SelfImprovementTab.tsx` — add a "Stuck Detection" settings group: `selfHeal.stuckDetector.enabled` toggle + `selfHeal.stuckDetector.repeatCalls` number input (min 3, max 50, default 5). Disable the number input when the toggle is off

### P16 — Docs + E2E + Coverage (scope-add) [US8/US9/US10]

- [ ] T105 [P] [US8/US10] Modify `docs/dev/self-healing/self-healing.md` — add sections: "Headless Run Transcription" (the format, where files live, the read API, how it is wired into both runners, the config gate), "Platform Abort" (the registry, the cascade for nested CLI runs, SIGKILL escalation), "Stuck-Run Detection" (the detector, the two conditions, `endedReason` threading, the `stopped` state, Stop/Start semantics, FR-035 re-entrancy). Reference `design.md` ADR-10–14
- [ ] T106 [P] [US9/US10] Modify `docs/usage/self-healing.md` — add: "Reading a run's transcript" (the BS Self-Heal page → case → Transcripts section); "Stopping a stuck run" (the amber stuck indicator, Stop/Start buttons, what `stopped` means, that Start launches a fresh run); the `agentRuns.transcriptions.enabled` setting
- [ ] T107 [US8/US9/US10] Add to `e2e/031-self-healing.spec.ts` — **SC-011**: trigger a self-heal case (explicit `self_heal.request`) → assert a file exists at `data/agent-transcripts/conversation-reviewer/<runId>.md` after the Diagnostician run completes; assert no file under `/Documents/Chats`. **SC-012**: open the BS Self-Heal page → the case → the Transcripts section → assert the Diagnostician run appears with status "Completed"; click it → assert the transcript panel renders the task input + at least one tool call. **SC-013** (the `file_grep` incident): set up a self-heal case where the pipeline run's agent repeatedly calls the same tool (mock the agent to emit the same `tool_call` 5×); assert the stuck detector fires (`run_stuck` event emitted); the case shows the amber stuck indicator; click Stop → assert the run is aborted, case → `stopped`, partial transcript written; click Start → assert a fresh run is launched, case → `bs-pipeline`. **SC-014**: the stuck case (from SC-013 setup) did NOT create a new self-heal case (only one case in the store). **R-SA5**: the aborted pipeline build's branch: `previewStateFor !== "ready"` (no false `fix_ready`). `test.describe.configure({mode:'serial'})`
- [ ] T108 [US8/US10] Run coverage — assert ≥95% line+branch on `src/lib/self-heal/**` AND `src/lib/agent/subagents/transcript.ts` AND `src/lib/agent/subagents/run-registry.ts`. If below, add targeted tests for uncovered branches (in particular: the disabled-config no-op path in `transcript.ts`, the unknown-runId no-op in `run-registry.ts`, the max-steps branch in the run-end handler)

**Increment 2 = US2 (T041–T045).** Automatic detection. The system starts noticing problems the user hasn't reported.

**Increment 3 = US3 + US4 (T046–T053).** Workflow-timeout and log-event triggers, plus the ownership predicate and `d-bis` routing. The mechanism covers all failure surfaces.

**Increment 4 = US5 (T054–T057).** Settings. The user can dial the mechanism up/down.

**Increment 5 = US6 + US7 (T058–T064).** Scheduled Diagnostician + exception stop. Proactive review and the safety valve.

**Final = Polish (T065–T070).** Docs, full e2e, coverage gate.
