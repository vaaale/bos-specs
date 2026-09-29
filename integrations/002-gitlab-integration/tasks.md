# Tasks: GitLab Integration (Webhook → Triage)

**Input**: Design documents from `/Specs/user-specs/integrations/002-gitlab-integration/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (user stories), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: Included — the feature spec + plan explicitly request tests (Vitest unit + Playwright e2e, incl. a regression guard for the v5 blocker). Story phases are **TDD**: the story's tests are written and FAIL before the story's implementation.

**Organization**: Tasks grouped by user story (spec.md US1–US5) for independent implementation/testing. The two platform additions (authless, mintSecret) and the integration skeleton are **Foundational** (block all stories). US1–US4 are the promoted webhook/R6/R7 base; **US5 (R8)** is an additive increment on top (it modifies the promoted dispatcher/agent/skill/labels, does not replace them).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: which user story (US1…US4); Setup/Foundational/Polish have no story label
- Exact file paths included in every task

---

## Phase 1: Setup

**Purpose**: Scaffold the integration module so story tasks have a home.

- [ ] T001 Create the gitlab integration module structure per [plan.md](./plan.md) at `src/lib/integrations/services/gitlab/` (folders: `adapters/`, `triage/`; empty placeholder files to be filled by later tasks: `manifest.ts`, `index.ts`, `labels.ts`)

**Checkpoint**: directory exists; `npx tsc --noEmit` still clean.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The two platform additions + integration skeleton that **every** user story depends on. ⚠️ No story work until this phase is complete.

- [ ] T002 [P] Add the **connected-by-default for authless integrations** platform addition: `authless?: boolean` on `IntegrationManifest` in `src/lib/integrations/types.ts` + coerce `connected = true` in `readState` (derive on read, don't persist) in `src/lib/integrations/state/store.ts` (design ADR-1)
- [ ] T003 [P] Add the **manual-webhook enable doesn't auto-mint** platform addition: `mintSecret?: boolean` (default `true`) on `enableWebhook` in `src/lib/integrations/webhooks/manager.ts` + forward `body.mintSecret` in `src/app/api/integrations/[id]/services/[serviceId]/webhook/route.ts` (design ADR-1b)
- [ ] T004 [P] Add the gitlab **event-type mapping** in `src/lib/events/legacy-mapping.ts`: `KNOWN_TYPES` entries `gitlab/webhooks:issue-opened` → `gitlab.issue.opened` and `gitlab/webhooks:mr-opened` → `gitlab.mr.opened`; `KNOWN_NAMES.gitlab = "GitLab"`; a `legacyFriendlySummary` branch (`"issue #<iid> <title>"` / `"MR !<iid> <title>"`) (design ADR-2; contract `contracts/bos-event.md`)
- [ ] T005 [P] Define the **label constants** — exactly 3 classification (`Bug`/`Feature`/`Improvement`) + 8 status (`Triage`/`Analyzing`/`Verifying`/`Fix Proposed`/`Spec Drafted`/`Needs Review`/`Needs Input`/`Failed`, with `Needs Review` marked terminal) in `src/lib/integrations/services/gitlab/labels.ts` (shared by the skill, panel, and tests; data-model §7)
- [ ] T006 Create `GITLAB_MANIFEST` — `id:"gitlab"`, `authless:true`, single service `webhooks`, **inert placeholder `oauthConfig`** (no real OAuth; must stay inert in `IntegrationListView`) in `src/lib/integrations/services/gitlab/manifest.ts` (depends T002) (design ADR-1/ADR-6)
- [ ] T007 Register the integration: add `import "./services/gitlab"` to `src/lib/integrations/index.ts`; create the **zero-method actions adapter** with `capabilities: { webhook: true }` in `src/lib/integrations/services/gitlab/adapters/webhooks-adapter.ts`; wire `registerIntegration(GITLAB_MANIFEST)` + `registerAdapter` in `src/lib/integrations/services/gitlab/index.ts` (depends T006) (design ADR-8 — the two-registry requirement)

**Checkpoint**: Foundation ready — the `gitlab` integration is registered, authless-connected, and its event types map. Unit gates green. User story implementation can begin.

---

## Phase 3: User Story 1 — Enable the webhook & receive a new-item event (Priority: P1) 🎯 MVP

**Goal**: The webhook-only integration is enableable in the panel (URL + generate/rotate token), receives a verified GitLab POST, dedups, and emits `gitlab.issue.opened` / `gitlab.mr.opened` into the event kernel. No handling yet — the event is the deliverable.

**Independent Test**: In the panel, Enable + Generate token. POST a well-formed new-issue payload with a correct `X-Gitlab-Token` → `200 {ok,emitted:1}` and `gitlab.issue.opened` appears in the Event Viewer. Wrong/missing token → `401`. Webhook disabled → `404`. A push/note/update POST → `200 {emitted:0}`. (quickstart Part 1.3–1.4, 1.9; SC-002.)

### Tests for User Story 1 (write first, ensure they FAIL) ⚠️

- [ ] T008 [P] [US1] Unit test `GitLabWebhookHandler.verify` — correct token → true; wrong/missing → false; **no candidates present → false** (never accept-all); rotation grace (previous accepted) in `src/lib/integrations/services/gitlab/__tests__/webhook.verify.test.ts` (contract `contracts/gitlab-webhook.md`)
- [ ] T009 [P] [US1] Unit test `GitLabWebhookHandler.receive` — issue+`open` → 1 `issue-opened` with correct projection; MR+`open` → 1 `mr-opened` (with branches); push / issue-`update` / note → `[]`; malformed JSON → `[]`; `description` truncated; `labels` normalized in `src/lib/integrations/services/gitlab/__tests__/webhook.receive.test.ts`

### Implementation for User Story 1

- [ ] T010 [US1] Implement `GitLabWebhookHandler` — `verify` (plain timing-safe match of `X-Gitlab-Token` against `[previous, primary]`, reject when no candidates) + `receive` (parse, filter `object_kind ∈ {issue,merge_request}` **and** `action==="open"`, project payload; everything else `[]`) in `src/lib/integrations/services/gitlab/adapters/webhook.ts` (depends T007; make T008/T009 pass)
- [ ] T011 [US1] Register `GitLabWebhookHandler` as `gitlab: { webhooks: … }` in `src/lib/integrations/webhooks/registry.ts` (depends T010)
- [ ] T012 [P] [US1] Create `GitLabLabelsReference` (classification + status chip groups, sourced from `labels.ts`) in `src/components/apps/settings/integrations/GitLabLabelsReference.tsx` (depends T005; mockup Labels section)
- [ ] T013 [US1] Create `GitLabDetailView` — header card (name + `Webhook` badge + "webhook only — handling is a separate, configurable step") + the reused **`WebhookSection`** (serviceId `webhooks`, `mintSecret:false`) + `GitLabLabelsReference`; **no** OAuth card, **no** MCP status in `src/components/apps/settings/integrations/GitLabDetailView.tsx` (depends T012; mockup states A/B/C; design ADR-6/R4)
- [ ] T014 [US1] Route `gitlab` → `GitLabDetailView` and update the generic-view fallback to `id !== "telegram" && id !== "gitlab"` in `src/components/apps/settings/IntegrationsTab.tsx` (depends T013) (design §3.9 / §9 F2 — must not also render the generic view's git-provider OAuth panel)

**Checkpoint**: US1 independently functional — the webhook emits the event and the panel is usable. (MVP increment 1: signal + config. No handling yet.)

---

## Phase 4: User Story 2 — Automatic triage & resolution (Priority: P2)

**Goal**: A default **core headless thin-dispatcher** handler (`gitlab-issue-triage`, enabled by default, toggleable) forwards each new issue to the configured **agent** (seeded **Gitlab Issue Agent** + **`gitlab-issue-triage` skill**), which classifies, runs the workflow, and updates the issue via the agent's GitLab MCP server, ending at `Needs Review`. The panel gains the **Workflow / agent-picker** card.

**Independent Test**: With a GitLab MCP server configured on the agent (manual) — or a stubbed agent/MCP (e2e): open a new `Bug` issue → `gitlab.issue.opened` → dispatched to the agent → issue gets classification comment + status labels advancing `Triage→Analyzing→Verifying→Fix Proposed→Needs Review` with a proposed fix posted. With **no** GitLab MCP reachable → `com.bos.gitlab.triage.failed` (`no_gitlab_mcp`), no crash, issue unmodified. (quickstart Part 1.2, 1.6, 1.8; SC-003, SC-007.)

### Tests for User Story 2 (write first, ensure they FAIL) ⚠️

- [ ] T015 [P] [US2] Unit test the dispatcher's **post-run, outcome-based failure detection** — agent missing → `agent_not_found`; `run.error`/`endedReason==="error"` → `run_error`; clean run with **no successful `mcp_tool_call`** → `no_gitlab_mcp`; clean run with **≥1 successful `mcp_tool_call`** → no failure event. **This test MUST FAIL on a `try/catch`-only implementation** (the v5 blocker: the gateway is in-band and the agent loop never dies of tools, so a clean run returns `completed` with no throw) in `src/lib/integrations/services/gitlab/__tests__/dispatch.test.ts` (contract `contracts/dispatcher-task.md` §4; design ADR-4)
- [ ] T016 [P] [US2] Unit test the label sets from `labels.ts` — exact 3 classification + 8 status labels, `Needs Review` is the terminal, single-active-status constant grouping in `src/lib/integrations/services/gitlab/__tests__/labels.test.ts`
- [x] T016a [P] [US2] Unit test that the skill's **references are reachable at run time** — drive the real `skill_load` / `skill_read_file` tools against the seeded skill (every `references/*.md` listed and readable), assert `skill_read_file` is in the `gitlab-issue-agent` gate **and survives `headlessGate`** (the dispatcher runs headless), and assert the router links exactly the shipped references, in `tests/agent/gitlab-triage-skill-references.test.ts` (guards design §3.8: a router whose references cannot be opened silently degrades to its own one-line summaries)

### Implementation for User Story 2

- [ ] T017 [US2] Write the **`gitlab-issue-triage` skill** (the pipeline driver, model-followed guidance) — read issue via GitLab MCP → classify (labels first, inference fallback + comment) → resume/idempotency (existing status label ⇒ resume) → run the matching pipeline applying the transient status label → terminal `Needs Review`; Q2 boundary (never close/commit); single-active-status swap. **Multi-file**: `seed/skills/gitlab-issue-triage/SKILL.md` is the ROUTER, with per-stage depth in `references/{status-labels,classify,bug-analyzing,bug-verifying,bug-fix-proposed,spec-drafted,finalize}.md` that the agent opens with `skill_read_file` as it enters each stage (design §3.8/ADR-12; data-model §6–§7)
- [ ] T018 [US2] Create the **Gitlab Issue Agent** — `type: local`, frontmatter `skills: ["gitlab-issue-triage"]`, `mcp` allowlist default empty (= all), tool scope (MCP + text/skill — `skill_read_file` included, without which the `references/*.md` of T017 are unreachable; no source/VFS-spec-write), system prompt in `seed/agents/gitlab-issue-agent/AGENT.md` (depends T017; design ADR-5)
- [ ] T019 [US2] Implement the **thin dispatcher** — resolve `triageAgentId` (default `gitlab-issue-agent`) from `state.services["webhooks"].config`, build the issue-context task (names the skill + restates Q2), `runSubAgent(agent, task, {contentOnly:true, onEvent})` collecting `tool_result`s, then the **post-run outcome-based** failure detection emitting `com.bos.gitlab.triage.failed` in `src/lib/integrations/services/gitlab/triage/dispatch.ts` (depends T015, T018; make T015 pass) (contract `contracts/dispatcher-task.md`; design §3.7)
- [ ] T020 [US2] Implement the **core headless handler** — `registerCoreExecutor("gitlab-issue-triage", …)` prompt-ack + `void dispatchToAgent(record)`; `api.register({ handlerId:"gitlab-issue-triage", eventType:"gitlab.issue.opened", mode:"headless", ownerId:"core", declaredBy:"core", displayName:"GitLab issue triage" })` **omitting `enabled`** (so disable persists); export `registerGitLabTriageHandler()` in `src/lib/integrations/services/gitlab/triage/handler.ts` (depends T019; design ADR-3/ADR-12, §9 F1)
- [ ] T021 [US2] Register the handler at boot — call `registerGitLabTriageHandler()` after `startEventKernel()` in `src/instrumentation.ts` (depends T020)
- [ ] T022 [US2] Create `GitLabWorkflowSection` — the **Workflow** card: agent picker (`<select>` over agents, default `gitlab Issue Agent`, "Manage →" to Settings → Agents) persisting `config.triageAgentId`; the skill note; the Events handler-toggle pointer with an "Open Events" link → Event Viewer (id `event-viewer`) in `src/components/apps/settings/integrations/GitLabWorkflowSection.tsx` (depends T013, T020; mockup Workflow card; design ADR-5/ADR-7)
- [ ] T023 [US2] Add `GitLabWorkflowSection` to `GitLabDetailView` (between the Webhook section and the labels reference) in `src/components/apps/settings/integrations/GitLabDetailView.tsx` (depends T022)

**Checkpoint**: US1 + US2 both work independently — a new issue now flows end-to-end to `Needs Review`. (MVP increment 2: the full loop, the genuinely useful MVP.)

---

## Phase 5: User Story 3 — Route the event elsewhere (Priority: P2)

**Goal**: The webhook and the handler are **independent switches** — disable the default handler and the event is still emitted (no agent dispatch); another consumer (e.g. the Workflow app) can take the same event.

**Independent Test**: Toggle `gitlab-issue-triage` **off** in Event Viewer → Configuration; open a new issue → `gitlab.issue.opened` is still emitted/visible but **no agent dispatches**; re-enable → dispatch resumes. (quickstart Part 1.7; FR-018, SC-004.)

### Tests for User Story 3 (write first, ensure they FAIL) ⚠️

- [ ] T024 [P] [US3] e2e — with the handler disabled, a new issue still emits `gitlab.issue.opened` but dispatches no agent; with it enabled, the dispatch occurs (covers FR-018 independence + `declaredBy:"core"` toggle authority) in `e2e/002-gitlab-integration.spec.ts` (depends T021)

### Implementation for User Story 3

- [ ] T025 [US3] Verify the "Open Events" pointer in `GitLabWorkflowSection` navigates to the Event Viewer handler toggle (the authoritative surface — no Settings→Events tab invented) and that multi-handler coexistence on `gitlab.issue.opened` is unobstructed in `src/components/apps/settings/integrations/GitLabWorkflowSection.tsx` (depends T022; design ADR-7)

**Checkpoint**: The event/handler split is demonstrable — routing elsewhere works without touching the webhook.

---

## Phase 6: User Story 4 — Merge request events (Priority: P3)

**Goal**: A new (action `open`) MR emits `gitlab.mr.opened`; the default handler does **not** dispatch for it (issue-only, Q1=A) — the MR event exists for the user to route elsewhere.

**Independent Test**: Open a new MR in a hooked project → `gitlab.mr.opened` is emitted; the `gitlab-issue-triage` handler does not run for it (it is registered for `gitlab.issue.opened` only). (quickstart Part 1.4 MR trigger; FR-019.)

### Tests for User Story 4 (write first, ensure they FAIL) ⚠️

- [ ] T026 [P] [US4] e2e — a new MR emits `gitlab.mr.opened` (projection incl. `sourceBranch`/`targetBranch`) and the default handler does not dispatch for it (Q1) in `e2e/002-gitlab-integration.spec.ts` (depends T011, T021)

**Checkpoint**: MR coverage complete (emit-only). All user stories independently functional.

---

## Phase 8: User Story 5 — R8: the agent runs the spec pipeline itself (Priority: P2)

**Goal**: On the user's approval (a GitLab comment at a human-waiting gate), the agent — on its **per-issue conversation** — creates an **ordinary** feature branch via a new `gitlab_start_feature` server tool, writes the **real** spec-kit artifacts to the branch headless (`spec.md`; + `mockup.html` via `file_write` + `design.md` for UI features), attaches them to the issue (FR-031), and ends at `Design Complete`. On the design's approval it continues **itself** (`plan.md` → `tasks.md` → `dev_delegate` TDD, same branch) to `Needs Review`. The agent never promotes/merges (FR-028); v1 = bos-core features only (FR-029). (design v6 ADR-14/15/16; contracts/gitlab-start-feature.md.)

**Depends on the promoted US2**: the dispatcher (`triage/dispatch.ts`), the core handler (`triage/handler.ts`), the agent (`seed/agents/gitlab-issue-agent/AGENT.md`), the skill (`seed/skills/gitlab-issue-triage/`), and the labels (`labels.ts`) are all **modified** by US5, not recreated.

**Independent Test**: (quickstart Part 2; SC-009/SC-008/SC-003) A `Feature` issue at `Spec Drafted` + an approval comment → the agent calls `gitlab_start_feature` → an **ordinary** `bos/<NNN>-<slug>` (next-number, never self-heal) is bound mid-run → `spec.md` (+ `mockup.html` + `design.md`) written to the branch → the issue is at `Design Complete` with an FR-031 post naming the branch + spec path → the branch's artifacts are listed in Build Studio's spec tree. A second approval at `Design Complete` → `plan.md`/`tasks.md` → `dev_delegate` → `Needs Review` with a healthy preview. A **feedback** comment → revise + re-post, stay at the gate, **no** branch allocated. A **no-op** (the agent's own note at a gate) → no comment, no label, no branch, no failure.

### Tests for User Story 5 (write first, ensure they FAIL) ⚠️

- [ ] T032 [P] [US5] Extend the label tests — the R8 status sets are exact: work stages `{ Triage, Analyzing, Verifying, Mockup, Design, Plan, Tasks, Implementing }`; human-waiting gates `{ Spec Drafted, Fix Proposed, Design Complete, Needs Review }`; `Needs Review` is the **promotion** gate (implementation-complete), not draft-complete; the single-active invariant holds across the extended set in `src/lib/integrations/services/gitlab/__tests__/labels.test.ts` (data-model §7)
- [ ] T033 [P] [US5] Unit test **`gitlab_start_feature`** — computes the **ordinary** `bos/<NNN>-<slug>` (NNN = max(existing)+1 in the chosen `user-specs` project; **never** self-heal naming, SC-009); a **mid-run bind** (conversation pre-existing with an empty `activeFeatureBranch`) is visible to a subsequent `file_write` on the same conversation (**the ADR-14 linchpin**); write-time re-check-and-bump on a concurrent collision (Risk 11); **idempotent** re-approval reuses the already-bound branch (no second NNN); an invalid slug returns an error (no ref written) in `src/lib/assistant/tools/server/__tests__/gitlab-feature-branch.test.ts` (contract `contracts/gitlab-start-feature.md`)
- [ ] T034 [P] [US5] Unit test **`pipeline.ts`** — `pipelineConversationId(projectId, iid)` = `c-gitlab-issue-<projectId>-<iid>` and is **stable across every note** on the issue; the ghost seed (`saveConversationMessages`) is idempotent (the same conversation across re-dispatch) in `src/lib/integrations/services/gitlab/__tests__/pipeline.test.ts` (design ADR-14)
- [ ] T035 [P] [US5] Unit test the **R8 dispatcher change** — a `gitlab.issue.noted` run is **conversation-bound** (seeds the per-issue ghost conversation + passes `conversationId`) and **omits `contentOnly`**; a `gitlab.issue.opened` run is **unchanged** (headless, `contentOnly: true`, no `conversationId`); the resume task names `references/approve-spec.md`/`approve-design.md` + `gitlab_start_feature` + "never promote or merge" but carries **no branch name**; a **no-op resume at a gate** (own note) is a **success** with no branch allocated in `src/lib/integrations/services/gitlab/__tests__/dispatch.test.ts` (contract `contracts/dispatcher-task.md`; design §3.7)
- [ ] T036 [P] [US5] Extend the **skill-references test** — the router's human-waiting states now point at `references/approve-spec.md` + `references/approve-design.md`; those references name `gitlab_start_feature`, `dev_delegate`, and the **`file_write`-of-`mockup.html`** (assert they do **not** name `ui_preview_*` — ADR-16); the approval-vs-feedback (FR-027), FR-031 attach, and "never promote/merge" clauses are present and both new references are readable through `skill_read_file` in `tests/agent/gitlab-triage-skill-references.test.ts`

### Implementation for User Story 5

- [ ] T037 [US5] Extend **`labels.ts`** to the R8 set (work stages + gate reclassifications + the human-waiting set + `Needs Review` = implementation-complete) in `src/lib/integrations/services/gitlab/labels.ts` (depends T032; make T032 pass) (FR-014, data-model §7)
- [ ] T038 [US5] Implement the **`gitlab_start_feature`** server tool (`execution: "server"`, cycle-free — imports only pure/spec/conversation helpers) — resolve the chosen `user-specs` project, scan its feature dirs for next `NNN`, compose + `isValidFeatureBranch`-validate `bos/<NNN>-<slug>`, **write-time re-check-and-bump**, **reuse** the conversation's already-bound branch on re-approval, bind via `setConversationActiveFeatureBranch(ctx.conversationId, branch)` (git ref lazy on first `dev_delegate`), return `{ branch, featurePath, store, featureId }`; **register** it in `src/lib/assistant/registry.ts` (static import — present from the first build) in `src/lib/assistant/tools/server/gitlab-feature-branch.ts` (depends T033; make T033 pass) (FR-023/FR-024; contract `contracts/gitlab-start-feature.md`)
- [ ] T039 [US5] Implement **`triage/pipeline.ts`** — `pipelineConversationId(projectId, iid)` + the idempotent **ghost seed** (`conversationStore.saveConversationMessages(conversationId, agentId, [task])`, the self-heal intake pattern keyed on the issue) + the conversation-bound run wrapper in `src/lib/integrations/services/gitlab/triage/pipeline.ts` (depends T034; make T034 pass) (design ADR-14; data-model §8)
- [ ] T040 [US5] Extend the **note projection** to carry the issue's current `labels[]` (from the top-level `issue.labels[].title`, the `issue.iid` fallback already present) in `src/lib/integrations/services/gitlab/adapters/webhook.ts` (depends T039) (FR-022; contracts/bos-event.md)
- [ ] T041 [US5] Extend the **dispatcher** — the `gitlab.issue.noted` run seeds the per-issue ghost conversation + runs **conversation-bound** (omit `contentOnly`); the `gitlab.issue.opened` run is **unchanged**; the resume task gains the approval-path clauses (create branch with `gitlab_start_feature` on approval; write real artifacts; `dev_delegate` after design approval; FR-031 attach; "never promote or merge"; do NOT create a branch on feedback) in `src/lib/integrations/services/gitlab/triage/dispatch.ts` (depends T038, T039, T040; make T035 pass) (FR-027; contract `contracts/dispatcher-task.md`)
- [ ] T042 [US5] Widen the **Gitlab Issue Agent** gate — `tools` from the v5 MCP+text/skill set to the **all-server pipeline gate** (`mcp_*`, `skill_*` incl. `skill_read_file`, `web_search`/`web_fetch`, `scratchpad_*`, `memory_*`, `file_*`, `gitlab_start_feature`, `dev_delegate`); `skills` gains `spec-kit-driver` + `ui-designer-craft`; `mcp: []` (all servers) in `seed/agents/gitlab-issue-agent/AGENT.md` (depends T038) (FR-028; design §3.11/ADR-16)
- [ ] T043 [US5] Extend the **`gitlab-issue-triage` skill** — new `references/approve-spec.md` (approval-1: judge approval → `gitlab_start_feature` → `skill_load` driver → `specify` (write `spec.md`) → if UI surface, `file_write` `mockup.html` (NOT `ui_preview_*`) + architect `design.md` → labels `Mockup`→`Design` → FR-031 attach → `Design Complete`; no `dev_delegate`) + `references/approve-design.md` (approval-2: judge approval → reuse bound branch → `plan.md` → `tasks.md` → `dev_delegate` (TDD, plan's file list = scope) → labels `Plan`→`Tasks`→`Implementing` → FR-031 attach → `Needs Review`); extend `references/notes.md` (approval-vs-feedback, FR-027) + `references/status-labels.md` (R8 work stages + gate reclass) + `references/finalize.md` (FR-031 attach; `Needs Review` = implementation complete) in `seed/skills/gitlab-issue-triage/` (depends T037, T042; make T036 pass) (design §3.12)
- [ ] T044 [P] [US5] Extend **`GitLabLabelsReference`** — the status-label chip group grows with the R8 labels (work stages + the two gate reclassifications + `Design Complete`), still sourced from `labels.ts` in `src/components/apps/settings/integrations/GitLabLabelsReference.tsx` (depends T037; mockup Labels section is the binding shape)
- [ ] T045 [US5] Add the **R8 e2e** (the `@@e2e` script pattern — every fixture drives the run via a script line in the issue/note, because an unscripted headless run hits the real LLM) — approval → ordinary-named branch + `spec.md`/`mockup.html`/`design.md` on the branch + FR-031 post + `Design Complete`; design approval → `plan.md`/`tasks.md` + `dev_delegate` → `Needs Review`; the branch's artifacts are listed in Build Studio's spec tree (SC-008); a feedback comment → revise + stay + no branch; a no-op at a gate → no comment/label/branch/failure in `e2e/002-gitlab-integration.spec.ts` (depends T041, T043; SC-009/SC-008/SC-003, FR-023…FR-031)

**Checkpoint**: US5 complete — an approved GitLab issue runs the spec pipeline **itself** to `Needs Review` with a healthy preview, ordinary branch naming, artifacts on the branch + attached to the issue, and the agent never promotes. (R8 increment: the full "drive it from GitLab" value.)

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T027 [P] Write end-user docs — enable webhook, generate token, register in GitLab, configure the agent/MCP, the label model — in `docs/usage/integrations/gitlab.md`
- [ ] T028 [P] Write developer docs — the webhook-only/`authless` pattern, the thin-dispatcher pattern, and **post-run outcome-based failure detection** (with the why — the in-band gateway / never-dies-of-tools contract) — in `docs/dev/integrations/gitlab-triage.md`
- [ ] T029 [P] Additional unit edge-case tests — unrecognized label falls to inference, oversized `description` truncation, dedup ring duplicate → `emitted:0` — in `src/lib/integrations/services/gitlab/__tests__/`
- [ ] T030 [P] Add the **loop-prevention** e2e — a follow-up `note` (comment) and `issue`/`action:"update"` (label) POST for the same issue does **not** emit a new `gitlab.issue.opened` (object_kind+action filter) in `e2e/002-gitlab-integration.spec.ts` (depends T011) (FR-020, SC-006, design ADR-9)
- [ ] T031 Run the full validation — [quickstart.md](./quickstart.md) manual path (SC-001) + all e2e (`e2e/002-gitlab-integration.spec.ts`) + gates `npx tsc --noEmit` and `npm run lint`; record results to `test-results.md`

**Checkpoint**: Feature complete, documented, and validated. Ready for Analyze/Converge then implement.

---

## Dependencies & Execution Order

### Phase dependencies

- **Setup (Phase 1)**: none.
- **Foundational (Phase 2)**: depends on Setup — **blocks all stories**. T002/T003/T004/T005 are parallel; T006 ← T002; T007 ← T006.
- **US1 (Phase 3)**: depends on Foundational. Tests T008/T009 parallel; T010 ← T007; T011 ← T010; T012 ← T005 (parallel); T013 ← T012; T014 ← T013.
- **US2 (Phase 4)**: depends on Foundational + US1 (T013 detail view, T011 webhook). Tests T015/T016 parallel; T017 → T018 → T019 → T020 → T021; T022 ← T013,T020; T023 ← T022.
- **US3 (Phase 5)**: depends on US2 (T021/T022).
- **US4 (Phase 6)**: depends on US1 (T011) + US2 (T021, to confirm issue-only registration).
- **US5/R8 (Phase 8)**: depends on the **promoted US2** (T017 skill, T018 agent, T019 dispatcher, T020 handler) — it **modifies** them. T032/T033/T034/T035/T036 (tests) parallel; T037 → T038 → T039 → T040 → T041; T042 ← T038; T043 ← T037,T042; T044 ← T037; T045 ← T041,T043.
- **Polish (Phase 7)**: depends on all desired stories. US5's docs (T043's docs are in Phase 8; the Phase-7 T027/T028 base docs are the v5/R7 set) are additive — R8 docs extend, not rewrite, them.

### User story independence

- **US1 (P1)**: standalone after Foundational — the event pipeline + panel.
- **US2 (P2)**: standalone after US1 — the handling (agent + skill + dispatcher). The useful MVP is US1+US2.
- **US3 (P2)**: depends on US2 (the toggle target) but is mostly verification of the existing kernel `setEnabled`.
- **US4 (P3)**: depends on US1's shared `receive` (MR branch) + US2's issue-only registration.
- **US5/R8 (P2)**: depends on the promoted US2 (it extends the dispatcher/agent/skill/labels). Standalone after US2 is promoted — the useful R8 increment is US2 + US5.

### Within each story

Tests are written and **FAIL before** implementation (TDD). The US2 dispatcher test (T015) is the regression guard for the v5 blocker and must fail on the pre-fix `try/catch`-only implementation.

### Parallel opportunities

- **Foundational**: T002 ∥ T003 ∥ T004 ∥ T005.
- **US1**: T008 ∥ T009 (tests); T012 ∥ (with T010/T011 on the server side).
- **US2**: T015 ∥ T016 (tests); T017 ∥ T015/T016 (skill is independent of the handler tests).
- **Polish**: T027 ∥ T028 ∥ T029 ∥ T030.

---

## Parallel Example: User Story 1

```text
# Server + UI sides are independent — launch together:
Task T010  "Implement GitLabWebhookHandler (adapters/webhook.ts)"
Task T012  "Create GitLabLabelsReference (GitLabLabelsReference.tsx)"
# then:
Task T011  "Register handler (webhooks/registry.ts)"          ← T010
Task T013  "Create GitLabDetailView (GitLabDetailView.tsx)"   ← T012
Task T014  "Route in IntegrationsTab"                         ← T013
```

---

## Implementation Strategy

### MVP First

1. Setup (T001) + Foundational (T002–T007) — foundation ready.
2. US1 (T008–T014) — **STOP and validate**: the webhook emits the event, the panel works. (Signal + config increment.)
3. US2 (T015–T023) — **STOP and validate**: the full loop (new `Bug` issue → `Needs Review`). This is the **useful MVP**.

### Incremental Delivery

4. US3 (T024–T025) — validate handler independence (FR-018).
5. US4 (T026) — validate MR emit-only (Q1).
6. Polish (T027–T031) — docs, edge tests, loop-prevention e2e, full quickstart + gates. (US1–US4 are the **promoted** base.)
7. **US5/R8 (T032–T045)** — **STOP and validate**: the approval-driven spec pipeline (approved issue → ordinary branch → real artifacts on the branch + attached to the issue → `Design Complete` → plan/tasks/implement → `Needs Review`). This is the R8 increment.
8. R8 docs + full validation (the R8 sections of T027/T028 docs; quickstart Part 2; the R8 e2e T045; gates).

Each increment is independently testable without breaking the previous. US2's T015 is the load-bearing regression test for the v5 blocker — do not skip it. US5's **T033 (the mid-run bind linchpin)** is the load-bearing regression test for the R8 architecture — do not skip it.

---

## Notes

- [P] = different files, no dependency on an incomplete task.
- [Story] maps each task to a spec.md user story for traceability (FR coverage: US1→FR-001…FR-008; US2→FR-009…FR-017,FR-020,FR-021; US3→FR-018; US4→FR-019; US5/R8→FR-023…FR-031 + the R8 rewording of FR-014/FR-022/SC-003/SC-009).
- The e2e (T024/T026/T030) **stubs the agent + GitLab MCP** and drives the **real** webhook receiver with recorded payloads; the real-credentials path is quickstart Part 1.
- Commit after each task or logical group; stop at any checkpoint to validate the story independently.
- Open low-stakes items (from plan.md), **resolved during the R8 plan** (no longer open): (c) the pre-approval draft is a **comment**; the post-approval **real** artifacts are **attached to the issue** (FR-031) — confirmed; (d) the seed `Gitlab Issue Agent` defaults to `mcp: []` (all servers) — confirmed. `WebhookSection` stock chrome (event-type filter + save button; "Rotate secret" label) is **accepted** as-is (low stakes; functionally identical to the mockup) — revisit only if it reads as a defect in review.
- **E2E gotcha (standing)**: any e2e that triggers a headless agent run hits the **real LLM provider** (the unit network guard does not cover e2e). Every R8 fixture (T045) must carry an `@@e2e` script line in the issue/note, matched by `e2e-provider.ts` (any line) — the same pattern R7's e2e used, extended to drive the approval gates + `gitlab_start_feature` + the branch writes + the FR-031 post.
