# Plan: GitLab Integration (Webhook → Triage) — R8

**Spec**: [`spec.md`](./spec.md) · **Design**: [`design.md`](./design.md) (v6) · **Data model**: [`data-model.md`](./data-model.md) · **Contracts**: [`contracts/`](./contracts/) · **Quickstart**: [`quickstart.md`](./quickstart.md) · **Mockup**: [`mockup.html`](./mockup.html)
**Feature Branch**: `002-gitlab-integration` (R8 work lands on `bos/002-gitlab-spec-pipeline`) · **Status**: Draft (v6 — R8: the GitLab agent runs the spec-kit pipeline *itself* on a server-side, per-issue feature branch)
**App Target**: `bos-core`

This plan covers the **R8 layer** on top of the already-promoted webhook/R6/R7 base. It is grounded in [`design.md`](./design.md) v6 (ADR-14/15/16 + §3.7/§3.10/§3.11/§3.12); it does not re-derive the promoted decisions.

---

## Constitution Check

Gated against the BrowserOS Constitution (`/Specs/bos-system-specs/.specify/memory/constitution.md`). R8-relevant principles:

| Principle | Status |
|---|---|
| **I. Spec-Driven — Software As A Prompt** | ✅ R8 is specified (`spec.md` R8/US5/FR-023…FR-031), designed (design v6 ADR-14/15/16), and planned here before any code. The agent runs the **spec-kit method headless** (`skill_load` the driver, write real artifacts to the branch) — it does not invent a parallel pipeline. The seed skill is a spec'd behavior contract. |
| **II. Server Authority & SSR Boundary** | ✅ The new `gitlab_start_feature` is a **server tool** (`execution: "server"`); `setConversationActiveFeatureBranch` / `saveConversationMessages` / the `NNN` scan are all server-side. No secret or Node API reaches the client. No GitLab credential is stored (R1/R4) — the tool binds a branch, it holds no token. |
| **III. Always Delegate; Claude Codes** | ✅ The GitLab agent (a `local` agent) runs the *method* headless, and delegates the **implementation** to the Developer via `dev_delegate` (`type: claude`) — the same path self-heal's Build Studio run uses. Claude codes; the local agent orchestrates. |
| **IV. Minimize Blast Radius** | ✅ All `src/` changes land on a feature branch via `dev_delegate`. R8 additions are **additive**: one new server tool, one new `triage/pipeline.ts`, label constants, skill references, agent-gate widening. No promoted R6/R7 mechanism is altered (only the noted run's `conversationId` + task wording + the tool gate). The per-issue "ghost" conversation reuses the self-heal conversation+branch pattern by **import-free convention** — self-heal is not modified. |
| **V. The VFS Is Not the Source** | ✅ The pre-approval deliverable stays a GitLab **comment**. Post-approval, the agent writes **spec artifacts** (`spec.md`/`mockup.html`/`design.md`/`plan.md`/`tasks.md`) to the branch-coupled `user-specs` store — spec content, not source, on the feature branch, never base. Source is edited only by the Developer in its isolated worktree (via `dev_delegate`). |
| **VI. Specs & Docs Stay in Sync** | ✅ `docs/usage/integrations/gitlab.md` is updated for R8 (the two-gate flow, the per-issue branch, FR-031 attachment) and a `docs/dev/` note covers the `gitlab_start_feature` tool + ghost-conversation pattern in the same change. |
| **VII. Respect Boundaries** | ✅ No `package.json`/lockfile/build changes, no new external dependency. Gates: `npx tsc --noEmit` + `npm run lint`. |
| **TDD** (Development Workflow) | ✅ Every R8 behavior has a failing test first — the test list below maps each SC to unit + e2e tests (Phase 2–5). The load-bearing regression is the **ordinary-naming** test (SC-009) and the **mid-run branch bind** test (ADR-14's linchpin). |
| **No workarounds / YAGNI** | ✅ `gitlab_start_feature` is a genuine platform mechanism (non-interactive counterpart to `dev_branch_request`, reusing `setConversationActiveFeatureBranch` + the ordinary branch-number source), not a special-case hack. v1 is scoped to **bos-core features** (`user-specs`; FR-029) — no marketplace-item path, no MR workflow (Q1), no self-heal coupling (R6). The NNN-contention store-level reservation is **not** built speculatively (ADR-15/Risk 11) — it's a recorded future fix. |

**Complexity Tracking (justified, not speculative):** the `gitlab_start_feature` tool exists because `dev_branch_request` is a *frontend elicitation* a headless run cannot answer (verified: `runLocalHeadless`'s `awaitFrontendResult` → `{kind:"timeout"}`); it is the minimum new surface that lets a headless agent create an *ordinary* branch. The ghost conversation reuses self-heal's conversation-store, not a new store. Both are justified by a verified gap, not preference.

**Conflict flagged:** none.

---

## R8 decisions in implementation terms

Carried from design v6 (ADR-14/15/16, §3.7/§3.10/§3.11/§3.12):

- **Mid-run, agent-driven branch creation (ADR-14).** The branch is read **fresh from the conversation file on every tool call** (`getConversationActiveFeatureBranch` is an uncached `vfs.readText`; `runLocalHeadless` resolves it from `opts.conversationId`, ignoring the `featureBranch` option for `local` agents). So a branch bound *while the agent is already running on that conversation* is picked up by its next `file_*` write / `dev_delegate` — no re-rooting. The dispatcher seeds a **stable per-issue conversation** (`c-gitlab-issue-<projectId>-<iid>`) on every `gitlab.issue.noted` and runs the agent **conversation-bound**; the agent, on a **judged approval**, calls `gitlab_start_feature`, which binds `bos/<NNN-slug>` to the run's own conversation. A **feedback** note never calls the tool — the branch never materializes.
- **Dispatcher knows the gate, not the intent.** The note hook carries `issue.labels[]` (verified against GitLab's `note_builder.rb`/`data_builder/note.rb`), so the dispatcher *can* see the issue is at a human-waiting gate — but approval-vs-feedback is a **semantic** read (FR-027) that belongs to the model. Binding on every gate-note would burn a number per feedback comment and collide on the later approval. The agent owns the bind.
- **Ordinary naming, server-computed (ADR-15, FR-023).** `gitlab_start_feature` resolves the chosen `user-specs` project, scans its feature dirs for `NNN = max(existing)+1` (the spec-kit numbering rule), composes `NNN-slug`, validates via `isValidFeatureBranch`, and returns `{ branch, featurePath, store }`. **Write-time re-check-and-bump** on collision (bounded retry) — Risk 11. Never self-heal naming.
- **The mockup is a `file_write` of `mockup.html`, not the live UI-Preview surface (ADR-16, spec-affecting, FR-025).** `ui_preview_generate`/`patch`/`web_view` are **frontend-execution** and stripped by `headlessGate` (no browser in a headless run). The static `mockup.html` — the spec-kit `ui-designer` role's own `bos-core` artifact — is written with `file_write`, routes to the branch via `withFeatureScope({conversationId})`, and is visible in Build Studio's artifact viewer (FR-030) with no browser.
- **FR-031 — attach the real artifacts to the issue.** Post-approval, the agent makes the produced artifacts visible on the **GitLab issue** (MCP upload/attachment where the server provides one, else a comment carrying the artifact), naming the **branch + spec path**. The branch artifacts remain the source of truth (FR-030); the issue post is the human-facing view. MCP-driven (the skill discovers the server's tools, same as comment/label tools).
- **`contentOnly` omitted on the noted path.** Inert for the agent's own `local` run, but **load-bearing** for the nested Developer: a `contentOnly:true` delegation takes the standalone-content path and *refuses* a bos-core source edit (`claude-runner.ts`). Approval-2 has dev work and must not pass it.
- **All-server tool gate (FR-028, ADR-16).** The agent's effective headless gate is `baseGate.allow` filtered to `execution: "server"` — all of `file_*`, `gitlab_start_feature`, `dev_delegate`, `skill_*`, `web_search`/`web_fetch`, `scratchpad_*`, `memory_*`, `mcp_*` survive. **No promote/merge tool exists** in the gate, so the "never promote/merge" boundary holds by construction (restated in the skill).

---

## Project structure (R8 file set, grounded)

**New:**
- `src/lib/integrations/services/gitlab/triage/pipeline.ts` — the per-issue conversation: `pipelineConversationId(projectId, iid)` → `c-gitlab-issue-<projectId>-<iid>`; the **ghost seed** (`conversationStore.saveConversationMessages(conversationId, agentId, [task])`, idempotent, mirrors `self-heal/intake.ts:515`); the conversation-bound run wrapper. Reuses the conversation-store server helpers self-heal uses (import-free reuse — self-heal is not modified).
- `src/lib/assistant/tools/server/gitlab-feature-branch.ts` — `gitlab_start_feature` (a `serverTool`, `execution: "server"`): resolve project + next `NNN` + slug (scan via `lib/specs/projects` + `lib/dev/spec-fs` for the project's feature dirs; validate via `lib/agent/feature-branch` `isValidFeatureBranch`), **write-time re-check-and-bump** on collision, bind `bos/<NNN-slug>` to `ctx.conversationId` via `setConversationActiveFeatureBranch` (`lib/assistant/conversation-store.ts`), return `{ branch, featurePath, store }`. **Cycle-free:** imports only pure/spec/conversation helpers — never the integration's `dispatch.ts`/`runner` (which import the tool registry).

**Modified:**
- `src/lib/assistant/registry.ts` — import + spread `gitlabStartFeatureTools()` into `assistantTools()`'s static map (the registry cache is built once; a static import is present from the first build — no invalidation needed). The only registration edit for the new tool.
- `src/lib/integrations/services/gitlab/adapters/webhook.ts` — the **note projection gains the issue's current `labels[]`** (from the top-level `issue` object — §3.4 of design; `object_attributes` describes the *comment*, the labels are on `issue`), so the `gitlab.issue.noted` event carries the gate state. (The iid is `issue.iid`, with a defensive `object_attributes.noteable_iid` fallback — the note hook has no `noteable_iid`.)
- `src/lib/integrations/services/gitlab/triage/dispatch.ts` — the **noted** run is conversation-bound: for `gitlab.issue.noted`, seed the per-issue conversation + pass `conversationId` to `runSubAgent` (omit `contentOnly`); the **fresh** (`gitlab.issue.opened`) run is unchanged (genuinely headless, no `conversationId`, `contentOnly: true`). The resume task text gains the approval-path clauses + "never promote or merge."
- `src/lib/integrations/services/gitlab/labels.ts` — R8 label set: work stages `Mockup`, `Design`, `Plan`, `Tasks`, `Implementing`; gate reclassifications (`Design Complete`; `Needs Review` refined to *implementation complete, preview healthy*); the human-waiting set gains `Design Complete`. Single-active-status invariant unchanged.
- `seed/agents/gitlab-issue-agent/AGENT.md` — `tools` widened from the v5 MCP+text/skill set to the full all-server pipeline gate (§3.11); `skills` gains `spec-kit-driver` + `ui-designer-craft`. `mcp` stays `[]` (all servers) — open question (d), resolved.
- `seed/skills/gitlab-issue-triage/SKILL.md` + `references/` — new `references/approve-spec.md` (approval-1) + `references/approve-design.md` (approval-2); extend `references/notes.md` (approval-vs-feedback, FR-027) + `references/status-labels.md` (R8 work stages + gate reclassifications) + `references/finalize.md` (FR-031 attachment, `Needs Review` = implementation complete).
- `src/components/apps/settings/integrations/GitLabLabelsReference.tsx` — the status-label chip group grows with the R8 labels (sourced from `labels.ts`; the mockup is the binding shape).
- `docs/usage/integrations/gitlab.md` + a `docs/dev/integrations/` note — R8 two-gate flow, per-issue branch, FR-031; the `gitlab_start_feature` + ghost-conversation pattern (Principle VI).

**Not modified:** the webhook receiver route; the self-heal module (R6 — separate; R8 reuses its conversation+branch pattern by convention).

---

## Design notes (implementation-relevant)

- **The fresh-read linchpin is a regression target.** The whole R8 design turns on `getConversationActiveFeatureBranch` being an *uncached per-call* read. A unit test must assert a branch bound **mid-run** (after the conversation already exists with an empty `activeFeatureBranch`) is visible to a subsequent `file_write` on the same conversation — this is what makes mid-run binding possible and guards against a future caching change silently breaking it.
- **The ghost conversation is inert.** Its transcript is ignored by `runLocalHeadless` (`io.loadMessages` returns `[]`); it exists only to (i) carry the branch and (ii) be the FR-030 visibility home. It is **not** a user Assistant conversation (Assumption A; the trajectory is the headless run transcript, not attached).
- **FR-030 needs no new code.** Build Studio's spec tree + artifact viewer read from the branch-coupled store root (the Supervisor worktree for `user-specs` on `bos/<NNN-slug>`). Once the agent writes the artifacts to that branch, they are visible with zero integration-specific code. The per-issue conversation's only FR-030 role is that it is *the conversation on that branch*.
- **The NNN scan is a spec-store fact, not an integration fact** — it lives in `gitlab_start_feature` (the server tool), not the dispatcher (ADR-15). The agent passes *intent* (project + title); the tool produces the *authoritative* name.

---

## Phase breakdown (becomes the tasks grouping)

**Phase 1 — Setup**
- Extend `labels.ts` (R8 label set; the human-waiting set; the single-active invariant).
- Add `gitlab_start_feature` to the tool registry (skeleton, no behavior yet).
- Scaffold the test files for Phase 2–5 (failing).

**Phase 2 — `gitlab_start_feature` + per-issue conversation**
- `triage/pipeline.ts`: `pipelineConversationId`, ghost seed, conversation-bound run wrapper.
- `tools/gitlab-feature-branch.ts`: resolve project + next `NNN` + slug, write-time re-check-and-bump, bind via `setConversationActiveFeatureBranch(ctx.conversationId)`, return `{branch, featurePath, store}`; **idempotency** (re-approval of the same issue reuses the existing bound branch — the ghost conversation already carries it — and does not allocate a new number).
- Unit tests: ordinary naming (SC-009, next number in the project, `bos/` prefix, never self-heal); the mid-run fresh-read linchpin; write-time re-check-and-bump (Risk 11); idempotent re-approval; validation of the composed branch.

**Phase 3 — Dispatcher approval path + note projection**
- `webhook.ts`: note projection gains `issue.labels[]` (and the `issue.iid` fallback). Unit test: a note payload whose `issue.labels` carries a gate label; a payload where the two ids deliberately differ (pin `noteable_id` ≠ `issue.iid`).
- `dispatch.ts`: the noted run seeds the per-issue conversation + runs conversation-bound (omit `contentOnly`); the fresh run unchanged; the resume task text gains the approval-path + "never promote/merge" clauses. Unit tests: fresh vs noted conversation-bound choice; the no-op resume is a success (no successful-MCP-call ⇒ not a success); the resume task names `references/approve-spec.md`/`approve-design.md` but carries no branch-creation instruction.

**Phase 4 — Skill extension + agent gate**
- `seed/agents/gitlab-issue-agent/AGENT.md`: widen `tools` to the all-server gate; add `spec-kit-driver` + `ui-designer-craft` skills; `mcp: []`.
- `seed/skills/gitlab-issue-triage/`: new `references/approve-spec.md` (approval-1: judge approval → `gitlab_start_feature` → `skill_load` driver → `specify` (write `spec.md`) → if UI surface, `file_write` `mockup.html` (NOT `ui_preview_*`) + architect `design.md` → labels `Mockup`→`Design` → FR-031 attach → `Design Complete`; no `dev_delegate`) + `references/approve-design.md` (approval-2: judge approval → reuse bound branch → `plan.md` → `tasks.md` → `dev_delegate` (TDD, plan's file list = scope) → labels `Plan`→`Tasks`→`Implementing` → FR-031 attach → `Needs Review`); extend `notes.md` (approval-vs-feedback + the self-terminating own-note branch), `status-labels.md` (R8 work stages + gate reclass), `finalize.md` (FR-031 attachment).
- Test (the `gitlab-triage-skill-references.test.ts` pattern): the router's human-waiting states now point at the new references; the references name `gitlab_start_feature`, `dev_delegate`, and the `file_write`-of-`mockup.html` (asserting it does **not** name `ui_preview_*`); the approval-vs-feedback + FR-031 + "never promote/merge" clauses are present.

**Phase 5 — FR-031 attachment + e2e**
- FR-031 attachment is **skill-driven** (the agent uses the MCP server's upload/attachment or a comment) — the e2e asserts the issue gains a post/comment naming the branch + spec path after each approval.
- E2E (the `@@e2e` script pattern from R7 — every fixture carries an `@@e2e` script in the issue description riding the provider, because an unscripted headless run hits the real LLM):
  - **SC-009 e2e:** a `Feature` issue → `Spec Drafted` → approval comment → agent creates `bos/<NNN-slug>` (assert ordinary naming, next number), writes `spec.md` + `mockup.html` + `design.md` to the branch, posts the FR-031 artifact, ends at `Design Complete`.
  - **SC-009/SC-008 e2e:** `Design Complete` → approval comment → agent writes `plan.md`/`tasks.md`, `dev_delegate` implements on the same branch, ends at `Needs Review`; the branch artifacts are openable in Build Studio (SC-008 — assert the spec tree lists them).
  - **SC-003 e2e:** a `Bug` issue → `Fix Proposed`; a `Feature`/`Improvement` → `Spec Drafted` (the reworded human-waiting states).
  - **Feedback e2e (FR-027):** a feedback comment on a gate → agent revises + re-posts, **stays** at the same gate, **no branch** allocated (assert no new `bos/NNN` feature dir).
  - **FR-016 regression (no-op):** a note that is the agent's own → no comment, no label, no branch, run succeeds.

**Phase 6 — Polish**
- `docs/usage/integrations/gitlab.md` (R8 two-gate flow, per-issue branch, FR-031) + `docs/dev/` note (the `gitlab_start_feature` + ghost-conversation pattern) — same change (Principle VI).
- `quickstart.md` R8 parts (below).
- `npx tsc --noEmit` + `npm run lint` clean.

---

## Risks & test notes

Carried from design v6 (Risks 11–14): **11** NNN contention across concurrent approvals (write-time re-check-and-bump; store-level reservation is a future fix, not built); **12** multi-minute headless method pipeline (bounded by `getMaxAgentSteps()` + `toolTimeoutMs: 600_000`; a `max_steps` exhaustion returns non-`error`, so a half-finished pipeline reports as acted — mitigated by the skill's resumable/idempotent stages, the next note resumes from the current label; the branch + partial artifacts survive); **13** ghost-conversation proliferation (bounded by distinct issues, not notes; inert; no GC in v1); **14** user switches to Build Studio mid-pipeline (branch artifacts are the source of truth; the GitLab labels go quiet past the handoff; the agent re-reads the live issue *and* branch artifacts on re-dispatch and reconciles — soft guidance).

**E2E gotcha (standing, from memory):** any e2e that triggers a headless agent run hits the **real LLM provider** (the unit network guard does not cover e2e). Every fixture must carry an `@@e2e` script in the issue description, matched by `e2e-provider.ts` (any line, not just offset 0) — the same pattern R7's e2e used. The R8 e2e fixtures extend that: the `@@e2e` script drives classify → label → the approval gate → `gitlab_start_feature` → the branch writes → the FR-031 post → `Design Complete` → (design approval) → `plan`/`tasks`/`dev_delegate` → `Needs Review`.

**Decisions closed during this plan (no longer open):**
- **(c)** The pre-approval draft is a **comment**; the post-approval **real** artifacts are **attached to the issue** (FR-031) *and* written to the branch (FR-030). — confirmed by the user.
- **(d)** The seed `Gitlab Issue Agent` defaults to `mcp: []` (all servers) for naming robustness. — confirmed.
- **(b)** `WebhookSection` stock chrome (event-type filter + save button; "Rotate secret" label) is **accepted** as-is (low stakes; functionally identical to the mockup). Revisit only if it reads as a defect in review.
