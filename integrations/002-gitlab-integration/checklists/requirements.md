# Specification Quality Checklist: GitLab Integration (Webhook → Workflow)

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-09-27
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs) — references to "GitLab MCP server", "event kernel", "X-Gitlab-Token" are named platform mechanisms / the external contract, not implementation choices
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous — Q1 and Q2 resolved; FR-015 and US5 now concrete (MR = event only, no built-in workflow)
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- Both clarifications resolved in Session 2026-09-27: **Q1 → A** (new MRs emit `gitlab.mr.opened`, no built-in workflow — triage/resolution is issue-only in v1; MR-review is a planned follow-up) and **Q2 → A** (default handler comments + relabels only; it proposes, never closes the item or opens an MR).
- **Revision (2026-09-27, user review)**: R1 no-OAuth (GitLab integration is webhook-only; all GitLab read/write via the GitLab MCP server; handler fails gracefully when MCP absent — SC-007), R2 classification labels fixed to `Bug`/`Feature`/`Improvement`, R3 handler-applied status labels defined. Spec + mockup fully rewritten.
- **One open design decision to confirm at `design`**: the generic webhook receiver's existing `state.connected` gate assumes a connected integration; for a webhook-only gitlab integration the design must treat it as available-by-default (only the webhook-enable flag + valid token gate inbound). Flagged in Assumptions.
- **Design step revised to v5** (2026-09-27, R6 handling-model change + re-review): handler is a **thin dispatcher**; the **Gitlab Issue Agent + `gitlab-issue-triage` skill** are the unit of capability; the deterministic MCP pre-flight is **removed** (failure detected **post-run, outcome-based**, per revised ADR-4); the panel gains a **Workflow / agent-picker** card; self-heal confirmed **fully separate**. R6 re-review (architect-reviewer): **one [BLOCKER] + 3 [NOTE]s, all resolved** (see design.md §9). The blocker was the failure-observability contract (the MCP gateway is in-band + the agent loop never dies of tools, so a try/catch never fires for the no-MCP case); fixed with post-run outcome detection (agent-missing / `run.error` / no successful `mcp_tool_call`), keeping R6's no-pre-flight model.
- Earlier v3 review: `design.md` passed a one-round architect-reviewer check — **Approve-with-notes, no blockers**. Two correctness fixes applied (F1: drop `enabled:true` from handler registration so disable persists across restarts; F2: `IntegrationsTab` fallback must exclude `gitlab`). Three precision improvements applied (F3 placeholder `oauthConfig` inert in list view; F4 `legacyFriendlySummary` branch; F5 `WebhookSection` chrome noted). Open for `plan`: one low-stakes UI decision (F5: accept stock `WebhookSection` labels/chrome vs. add a label prop for exact mockup parity).
- All content items pass. **Design re-review complete — Ready for `plan` → `tasks` → `implement`.** (Open low-stakes items remain: WebhookSection label parity; confirm "draft spec" is posted as a GitLab comment; seed agent `mcp: []` default.)

## R8 (Session 2026-09-28 — the agent runs the spec pipeline itself)

- **R8 added** (spec revision 4): the GitLab agent runs the spec-kit pipeline **itself** (no Build Studio agent delegation — it is the label/state owner); two human-waiting gates (approval-1 `Spec Drafted`/`Fix Proposed` → branch + real artifacts → `Design Complete`; approval-2 → plan/tasks/`dev_delegate` → `Needs Review`); **ordinary** branch naming (`bos/NNN-slug`, never self-heal — FR-023); **server-side** branch creation (FR-024, the self-heal conversation+branch mechanism, non-interactive `gitlab_start_feature` tool); the agent's tool gate expands to the all-server set (FR-028); **FR-031** attaches the real artifacts to the issue post-approval; v1 = bos-core features only (FR-029). **US5** added.
- **R8 content quality**: no `[NEEDS CLARIFICATION]` markers; R8 FRs (FR-023…FR-031) testable + unambiguous; the two open plan questions **resolved** (c: draft=comment, real artifacts attached per FR-031; d: seed agent `mcp: []`); SC-003 reworded + SC-009 added.
- **Design v6** (architect, source-verified): ADR-14 (mid-run, agent-driven branch creation — the ghost conversation + `gitlab_start_feature`, grounded in `getConversationActiveFeatureBranch` being an uncached per-call read + `runLocalHeadless` resolving from `opts.conversationId`), ADR-15 (ordinary naming, server-computed NNN, idempotent reuse), ADR-16 (all-server headless gate; the mockup is a `file_write` of `mockup.html`, **not** the live UI-Preview surface — the **spec-affecting finding** applied to FR-025/FR-022).
- **Plan artifacts (v6, R8)**: `plan.md` (Constitution Check **PASS** — TDD, no-workarounds, YAGNI, blast-radius, VFS-is-not-source all hold; the `gitlab_start_feature` tool is a genuine platform mechanism closing a verified gap, not a hack), `data-model.md` (ghost conversation §8, feature branch §9, FR-031 post §10, extended label model §7), `contracts/gitlab-start-feature.md` (new) + `bos-event.md` (`issueLabels`) + `dispatcher-task.md` (conversation-bound noted run, `contentOnly` omitted, approval-path clauses) + `quickstart.md` (R8 Parts 2a–2d + R8 automated checks).
- **Tasks (R8)**: **US5** = T032–T045 (Phase 8) on top of the promoted T001–T031 — labels, `gitlab_start_feature` (+ the mid-run-bind linchpin test T033), `pipeline.ts`, note projection, dispatcher, agent gate, skill extension, labels reference, R8 e2e (T045, `@@e2e` script pattern).
- **Ready for `implement`** — **but implementation is gated on explicit user authorization** (the spec pipeline's `implement` is a hard gate). The promoted base is US1–US4 (T001–T031); US5/R8 (T032–T045) is the additive increment.
