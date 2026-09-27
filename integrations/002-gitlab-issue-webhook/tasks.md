# Tasks: GitLab Issue Webhook Integration

**Input**: `spec.md` (5 user stories, 31 FR / 7 NFR / 9 SC), `design.md` (reviewer-passed architecture), `plan.md` (Constitution gate PASS, file structure, plan-time decisions), `mockup.html` (binding UI contract).
**Branch**: `bos/002-gitlab-issue-webhook` (active). Target: **bos-core** — delegate `implement` to the Developer via `dev_delegate`.

**Tests**: **Required** (per spec's Testing strategy + project convention) — Vitest unit + Playwright e2e, all self-cleaning (temp data dirs, reset seams, no leftover cases/branches/state; the 034 convention). Fast deterministic unit tests are TDD (written red, then green). The heavy e2e (driven Bug / driven Feature) are integration-level and **gated on the R6 spike + the M5 environmental guard** — they are validation e2e, not "write-red-first" (they depend on real self-heal / Build Studio runs).

## Format: `[ID] [P?] [Story] Description`
- **[P]**: parallelizable (different files, no dependency on an incomplete task).
- **[Story]**: US1…US5 (setup/foundational/polish carry no story label).
- Every task names its exact file path.

> **Structure refinement vs plan.md (one file):** the 8-label vocabulary is extracted to `src/lib/integrations/services/gitlab/labels.ts` (plan had it inline in `issue-router.ts`) because `brief-section.ts` imports it too; `issue-router.ts` imports `labels.ts` one-way. Nothing else changes.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: create the new integration service + test locations. No logic yet.

- [ ] T001 Create empty service + test folders: `src/lib/integrations/services/gitlab/` (with `adapters/`) and `tests/integrations/gitlab/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: validate the load-bearing assumption and build the shared modules + the self-heal seam every story depends on.

**⚠️ CRITICAL**: no resolution-path work (US2/US3) can be validated until T002 (R6) is complete.

- [ ] T002 **R6 spike (runs first)** — Validate gitlab-mcp tools are reachable from a headless `runSubAgent` child: one validation run against the target project doing `get_issue` + `list_labels` + `create_label` round-trip. Record the conclusion (design.md OQ / R6). **Ground the brief mutation-whitelist's real MCP tool names** here. If it fails, stop and report — resolution is non-functional end-to-end (no kernel fallback by design).
- [ ] T003 [P] Label-vocabulary module `src/lib/integrations/services/gitlab/labels.ts` — `CLASSIFICATION_LABELS = ["Bug","Feature","Enhancement"]`, `WORKFLOW_LABELS = ["agent:queued","agent:in-progress","agent:needs-input","agent:failed","agent:preview-ready"]`, `ALL_LABELS`, predicates `isClassificationLabel`/`isWorkflowLabel`/`exactlyOneClassification`, the 5-label allowed-transition map (design §Data model; `agent:queued` provisioned-but-reserved), and the `PROPOSE_ONLY_TERMINAL = "agent:preview-ready"` constant. Unit `tests/integrations/gitlab/labels.test.ts`.
- [ ] T004 Thin per-issue resolution state `src/lib/integrations/services/gitlab/resolution-state.ts` — the real `IntegrationState` shape (top-level `connected: boolean` is the receiver gate; `services["issues"] = { enabled, config, lastSync }`; additive `config.extras.perIssue` map `issueKey="<projectPath>:<issueIid>" → { caseId? | featureBranch?, workflowLabel, updatedAt }`); serialized atomic writes via `readState`/`mutateState` (`src/lib/integrations/state/store.ts`, self-heal store precedent). **Map entry created ONLY on an in-scope dispatch or an ambiguity record; a 0-label event creates NO entry** (M4/FR-014). Unit `tests/integrations/gitlab/resolution-state.test.ts`.
- [ ] T005 GitLab-reporting brief protocol `src/lib/integrations/services/gitlab/brief-section.ts` — `renderGitLabBriefSection({ mode: "bug" | "feature" | "ambiguity" | "validate", issue, projectPath, branch? }) → string`. Per mode: the allowed-mutation MCP whitelist (tool names from T002; **no merge/close/promote**), the stage→label transitions (first GitLab mutation = `agent:in-progress` + pickup note; stage notes from the real lifecycle; terminal `agent:preview-ready`; `agent:needs-input`; `agent:failed`; the closed-mid-run `get_issue` check at each mutation), and FR-031 lazy idempotent label provisioning (first run in a project creates all 8 via `list_labels`/`create_label`). Unit `tests/integrations/gitlab/brief-section.test.ts` (asserts required instructions present + **no merge/close/promote** substring).
- [ ] T006 Self-heal brief-augmenter seam (R7c — the only edit to an existing subsystem) — NEW `src/lib/self-heal/brief-augmenters.ts` (tiny registry `{ matches(ctx) → boolean, section(ctx) → string }[]`); EDIT `src/lib/self-heal/brief.ts` (one line: append matching augmenter sections to the autonomous brief); register the gitlab bug augmenter (`matches` = `ctx.extra?.gitlab` present → `section` = `renderGitLabBriefSection` bug mode). **Regression unit**: `buildAutonomousBrief` output is byte-identical for any non-gitlab case. (Depends on T005.)

**Checkpoint**: foundation ready — R6 validated, shared modules + seam in place. US1/US2/US3 can begin.

---

## Phase 3: User Story 1 — Intake, Settings & Connection (P1) 🎯

**Goal**: a GitLab issue webhook is received + verified and published to the bus; the integration is configurable in Settings (MCP-connection state, target project, manual registration, token, last-received).

**Independent Test** (spec US1): valid signed POST → 200 + event `com.bos.gitlab.issues.issue-opened` on the bus; wrong token → 401; disabled / no handler → 404; duplicate delivery → no second event; non-issue `object_kind` → no event; Settings panel surfaces URL + token + manual-registration steps + connection state (matches `mockup.html`).

- [ ] T007 [P] Integration manifest `src/lib/integrations/services/gitlab/manifest.ts` — `IntegrationManifest` id `gitlab`, `services: [ issues ]` with `configSchema { projectPath }`, `oauthConfig` = a no-flow descriptor (the kernel holds no GitLab credential; registration is manual). No provider-side hook fields.
- [ ] T008 [P] Webhook handler `src/lib/integrations/services/gitlab/adapters/issue-webhook.ts` — `verify` = **local** `timingSafeStringEqual` of `X-Gitlab-Token` against `[config.extras?.token, secrets.primary, secrets.previous]`; missing header OR empty candidate set ⇒ `false` (the deliberate stricter-than-Telegram divergence, m2). `receive` = parse `object_kind === "issue"` → `{ type: "issue_opened" | "issue_updated", data: { projectPath, issueIid, title, description, labels, author, action, issueUrl } }` with `service: "gitlab/issues"` + `timestamp`; non-issue ⇒ `{ events: [] }`. No `onEnable`/`onDisable` (platform mints/rotates the secret). Unit `tests/integrations/gitlab/issue-webhook.test.ts`.
- [ ] T009 Barrel + registries — `src/lib/integrations/services/gitlab/index.ts` (`registerIntegration(GITLAB_MANIFEST)` + `registerWebhookHandler("gitlab","issues",issueWebhookHandler)`; spine registration is added in T013); EDIT `src/lib/integrations/index.ts` (`import "./services/gitlab";`); EDIT `src/lib/integrations/webhooks/registry.ts` (static `HANDLERS.gitlab = { issues: issueWebhookHandler }`). (Depends on T007, T008.)
- [ ] T010 Connection-state derivation + validation run (FR-001) — a validate-mode sub-agent run (`renderGitLabBriefSection` validate) does a `get_issue`/`list_labels` round-trip for the target project and reports back; on success `mutateState` flips top-level `state.connected = true` (and `false` on failure/unavailable). The receiver gate reads top-level `connected` (404 when false). Unit for the state flip. (Depends on T004, T005, T002.)
- [ ] T011 Settings section `src/components/apps/settings/integrations/GitLabIssuesSection.tsx` (MCP-connection card, target-project field, manual-registration steps w/ copy, label-provisioning status, last-received; **reuses `WebhookSection`** for URL + token reveal/copy/rotate + enable) + EDIT `ServiceConfigView.tsx` (render `<GitLabIssuesSection>` for `gitlab/issues`). Per `mockup.html`. (Depends on T010.)
- [ ] T012 e2e (webhook→bus) — append to `e2e/002-gitlab-issue-webhook.spec.ts`: valid signed POST → 200 + `com.bos.gitlab.issues.issue-opened` on the bus; wrong token → 401; disabled → 404; duplicate delivery (idempotency) → no second event; non-issue `object_kind` → no event. M5 guard (gitlab-mcp preconditions) + self-cleaning (temp data dir, reset seams). (Depends on T008, T009, T010.)

**Checkpoint**: US1 fully functional — an issue webhook lands on the bus and the integration is configurable. **First increment (intake-only; no resolver value yet).**

---

## Phase 4: User Story 2 — Bug → self-heal → draft MR (P1)

**Goal**: an issue labeled `Bug` is dispatched to the self-heal pipeline; the fix is pushed to the watched project + a **draft** MR opened + linked; the issue reaches `agent:preview-ready`.

**Independent Test** (spec US2): POST a `Bug` issue → a self-heal case is created → analysis + verification notes posted → fix pushed to the project + draft MR created (`draft: true`) → issue labeled `agent:preview-ready` and linked to the MR; **no merge/close/promote**.

- [ ] T013 Issue router — classify + Bug arm `src/lib/integrations/services/gitlab/issue-router.ts` — core headless executor on `com.bos.gitlab.issues.*` (034 spine, `registerSelfHealSpine`-pattern): read classification labels **from the event payload**; exactly one → route; **Bug** → emit `com.bos.self-heal.trigger` with `TriggerContext { trigger:"explicit", description: title+body, toolName: "gitlab-issue:<iid>", extra.gitlab: { projectPath, issueIid, issueUrl } }`, **no `selfHeal.role` marker** (genuine external trigger); create the `issueKey` map entry via `resolution-state`. Complete the barrel: EDIT `index.ts` to `registerGitLabIssueSpine`. Unit: `Bug` routes; 0-label is a no-op creating no entry (FR-020); >1-label is deferred to the US5 arm (stub now). (Depends on T004, T006.)
- [ ] T014 Bug path wiring + case observer — compose the end-to-end bug run: T013 trigger → spine → Diagnostician → `escalateCase` → `buildAutonomousBrief` (T006 seam appends the T005 bug-mode section, which carries the **delivery tail** (R1): `create_branch` → `push_files`/`create_or_update_file` the diff → `create_merge_request { draft:true }` → link issue↔MR → `agent:preview-ready` + summary; on needs-input → question + `agent:needs-input` + stop; on failure → `agent:failed` + note; closed-mid-run → `agent:failed` + abandonment note). Reuse the **case-bound** `createRunObserver` for the bug path (it has a case) to mirror needs-input/failed (key off the case store, m6 — no `case_failed` event). Unit: the bug brief text contains the delivery-tail steps + no merge/close. (Depends on T013, T005, T006.)
- [ ] T015 e2e (driven Bug) — append to `e2e/002-gitlab-issue-webhook.spec.ts`: POST a `Bug` issue → assert self-heal case created, analysis + verification notes posted, fix branch pushed to the project, **draft** MR created, issue labeled `agent:preview-ready` + linked, **no merge/close/promote**. M5 guard (requires a buildable watched project — A6 — and R6). Self-cleaning (delete case/branch/MR). (Depends on T014.)

**Checkpoint**: US1 + US2 — a Bug goes webhook → bus → self-heal → draft MR → `agent:preview-ready`. **First end-to-end increment that delivers value.**

---

## Phase 5: User Story 3 — Feature / Enhancement → Build Studio → preview (P1)

**Goal**: an issue labeled `Feature`/`Enhancement` drives the Build Studio spec pipeline autonomously (specify → design → plan → tasks → implement → preview) on a feature branch; the issue reaches `agent:preview-ready` with preview/branch links.

**Independent Test** (spec US3): POST a `Feature` issue → a feature-intent run starts (branch pre-set) → spec artifacts created → implementation → preview built + tested → issue labeled `agent:preview-ready` with branch/preview links + stage notes; **no draft MR, no promote**.

- [ ] T016 Router Feature/Enhancement arm — extend `issue-router.ts`: `Feature`/`Enhancement` → **the router pre-sets the conversation's active feature branch** (`bos/gitlab-<project>-<iid>`) BEFORE `runSubAgent` (self-heal precedent); builds the feature-intent brief (BS per-conversation autonomous pre-authorization text + `renderGitLabBriefSection` feature mode; `Enhancement` carries "which existing feature" for the **run** to resolve per D-OQ3); creates the map entry; calls `runSubAgent({ contentOnly:false, featureBranch })`. **The run's first step creates the project** (M1 — the router does NOT call `create_project`). Unit: `Feature` routes to the feature arm, branch pre-set, no `create_project` call in the router. (Depends on T013.)
- [ ] T017 Feature-path observer — **NEW thin code** (not `createRunObserver`, which is case-bound): wraps the case-agnostic `StuckDetector` (`src/lib/self-heal/stuck-detector.ts`) + the run's `onEvent`; posts/mirrors `agent:needs-input`/`agent:failed` for the feature run (no case store → key off run-end + StuckDetector). Unit. (Depends on T016.)
- [ ] T018 e2e (driven Feature) — append to `e2e/002-gitlab-issue-webhook.spec.ts`: POST a `Feature` issue → assert feature branch created, spec artifacts on that branch (project + feature created **by the run**), implementation, preview built + tested, issue labeled `agent:preview-ready` with branch/preview links + stage notes, **no draft MR, no promote**. M5 guard; D-R2 (no slot; bounded by `issueKey` + platform per-run limits + StuckDetector). Self-cleaning. (Depends on T017.)

**Checkpoint**: US1 + US2 + US3 — all three P1 stories work independently.

---

## Phase 6: User Story 4 — Issue Updated Throughout (P2)

**Goal**: an in-progress run reflects each stage on the issue (the `agent:*` label + a note); the workflow-label state machine and the propose-only boundary hold.

**Independent Test** (spec US4): given an in-progress run (bug or feature), when a stage completes, the issue carries the corresponding `agent:*` label + a note; workflow labels stay mutually exclusive; **no merge/close/promote ever**.

- [ ] T019 Label-state-machine enforcement + propose-only guard — ensure the run brief (T005) + observers (T014 bug, T017 feature) apply **only** the 5 workflow labels per the allowed-transition map (mutual exclusion), never a merge/close/promote MCP call; add a static assertion/test that `brief-section.ts` output and both observers contain no merge/close/promote. (Mostly verification + guard test; the mechanism is foundational.) Unit. (Depends on T005, T014, T017.)
- [ ] T020 e2e (label/note lifecycle) — append to `e2e/002-gitlab-issue-webhook.spec.ts`: drive a run (reuse the US2 bug or US3 feature harness) and assert the full label/note **sequence** — `agent:in-progress` + pickup → (stage notes) → `agent:preview-ready` — each workflow label applied in order and mutually exclusive, and no merge/close/promote occurred. M5 guard. Self-cleaning. (Depends on T019, T015 or T018.)

**Checkpoint**: US4 — the issue transparently reflects the run's progress.

---

## Phase 7: User Story 5 — Out-of-scope & Ambiguity (P3)

**Goal**: an issue with no classification label is left **unchanged** (no note, no map entry, no run); one with multiple classification labels gets an ambiguity note and no pipeline.

**Independent Test** (spec US5): POST a 0-label issue → no note, no map entry, no run, issue unchanged; POST a `Bug`+`Feature` issue → an ambiguity note + no pipeline; a later single `Bug` label applied to the 0-label issue **is** picked up (FR-014).

- [ ] T021 Router 0-label no-op + ambiguity arm — extend `issue-router.ts`: **0 classification labels → pure no-op, NO map entry, NO GitLab write** (FR-020/SC-009); **>1 → ambiguity** per design R4 (the thin ambiguity sub-agent run via `renderGitLabBriefSection` ambiguity mode posting exactly one note; no pipeline dispatch). Ensure a later single-label event on a previously-0-label issue **dispatches** (M4 — no blocking entry was created). Unit: 0-label → no entry + later label dispatches; >1 → ambiguity note, no case, no feature run. (Depends on T013, T005.)
- [ ] T022 e2e (out-of-scope + ambiguity + FR-014) — append to `e2e/002-gitlab-issue-webhook.spec.ts`: POST 0-label issue → assert no note, no map entry, issue unchanged; POST 2-label issue → assert ambiguity note + no self-heal case + no feature run; then apply a single `Bug` label to the 0-label issue → assert it now dispatches (FR-014 label-after-open). M5 guard. Self-cleaning. (Depends on T021.)

**Checkpoint**: all five user stories independently functional.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T023 [P] Documentation (Constitution VI) — `docs/usage` (end user: configuring gitlab-mcp, validating the connection, enabling the webhook, the manual project-webhook registration steps (URL + token, Issues events only), the label model (classification vs workflow, the 5-state lifecycle), the propose-only boundary) + `docs/dev` (developer: the handler + registry entry pattern, the I/O-free `issue-router.ts` + thin `resolution-state.ts`, the self-heal **brief-augmenter seam** + its no-op guarantee, the GitLab-reporting brief protocol in `brief-section.ts`).
- [ ] T024 [P] Security hardening pass — assert **no body/token logging** in handler/router/observers (NFR-001); 404-not-401; constant-time verify; the stricter-than-Telegram divergence is documented (m2). Add a focused unit if not already covered.
- [ ] T025 Final full e2e run — `buildstudio_run_tests` on `user-specs/integrations/002-gitlab-issue-webhook`: all cases green; confirm self-cleaning (no leftover cases/branches/state); confirm the M5 guard **fails fast (not hang)** when gitlab-mcp is absent.
- [ ] T026 Self-heal regression (R7c) — confirm the self-heal pipeline is unaffected for non-gitlab cases: run the T006 no-op regression unit + a self-heal e2e smoke. The `brief.ts` edit is the only touch to an existing subsystem.

---

## Dependencies & Execution Order

### Phase Dependencies
- **Setup (P1)**: none — start immediately.
- **Foundational (P2)**: depends on Setup. **T002 (R6) gates T005, T006, T010, and all of US2/US3** — it runs first. T003 ∥ T004.
- **US1 (P3)**: after Foundational.
- **US2 (P4)**: after US1 (shares the router file `issue-router.ts` — build classify + Bug arm first) + Foundational.
- **US3 (P5)**: after US2 (extends `issue-router.ts` with the feature arm).
- **US4 (P6)**: after US2 + US3 (its e2e reuses a driven run; the guard test spans both observers).
- **US5 (P7)**: after US2 (extends `issue-router.ts` with the ambiguity arm).
- **Polish (P8)**: after all desired stories.

### User Story Dependencies
- **US1 (P1)**: needs Foundational. No other story.
- **US2 (P1)**: needs US1's intake (the bus event) + the router base. 
- **US3 (P1)**: needs US2's router base (classify). Otherwise independent of US2's delivery.
- **US4 (P2)**: needs US2 + US3 (observers + a driven run to verify).
- **US5 (P3)**: needs US2's router base (classify). Independent of US2/US3 delivery.

> **Same-file serialization**: `issue-router.ts` is built across US2 (T013) → US3 (T016) → US5 (T021); `e2e/002-gitlab-issue-webhook.spec.ts` is extended across T012 → T015 → T018 → T020 → T022. These are sequential, not parallel.

### Within Each Story
- Fast deterministic unit tests are TDD (red → green).
- Model/state modules (T003, T004) before the consumers (router, brief).
- Handler (T008) before receiver e2e (T012).
- Brief protocol (T005) + seam (T006) before any resolution path.

### Parallel Opportunities
- **Foundational**: T003 ∥ T004 (different files; both after T002).
- **US1**: T007 ∥ T008 (manifest vs handler, different files).
- **Polish**: T023 ∥ T024.
- The three P1 stories' **e2e** (T015, T018) cannot run in parallel — they share `e2e/002-gitlab-issue-webhook.spec.ts` and both drive real pipelines (serial, per the 039 `test.describe.configure({mode:"serial"})` precedent for shared-state e2e).

---

## Parallel Example: US1

```text
Launch together (different files, no mutual dependency):
  T007  manifest.ts
  T008  adapters/issue-webhook.ts
Then (both needed):
  T009  barrel + registries  →  T010  connection  →  T011  settings  →  T012  e2e
```

---

## Implementation Strategy

### MVP First
1. Phase 1 (Setup) → Phase 2 (Foundational, **T002 R6 spike first**).
2. **Phase 3 (US1)** → intake lands on the bus, integration configurable. *(intake-only)*
3. **Phase 4 (US2)** → a Bug goes end-to-end to a draft MR. **← first increment that delivers value; STOP and validate here.**
4. *(Optional stop — US1+US2 is a shippable MVP.)*

### Incremental Delivery
5. **Phase 5 (US3)** → Feature/Enhancement to a preview. (Completes the P1 set.)
6. **Phase 6 (US4)** → the "throughout" transparency is asserted.
7. **Phase 7 (US5)** → out-of-scope/ambiguity guards.
8. **Phase 8 (Polish)** → docs, security pass, full e2e, self-heal regression.

Each story is independently testable; the same-file router/e2e tasks serialize in the order above.

### Notes
- **R6 is the gate.** If T002 fails, the resolution half (US2/US3/US4/US5 e2e) is non-functional — do not proceed past US1 without resolving it (that would be a design change, surfaced to the user).
- **A6 + M5**: the driven-pipeline e2e (T015, T018, T020) require a buildable watched project (the user's `browseros` GitLab, repo 42) and a configured gitlab-mcp; the M5 guard makes them fail fast, not hang, when absent.
- **Propose-only (NFR-004)** is enforced by the absence of any merge/close/promote MCP call in the brief (T005), the observers (T014/T017), and the guard test (T019).
- **Commit after each task or logical group.** Stop at each checkpoint and validate the story independently.
