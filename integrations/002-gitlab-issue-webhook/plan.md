# Implementation Plan: GitLab Issue Webhook Integration

**Branch**: `bos/002-gitlab-issue-webhook` (active) | **Date**: 2026-09-27 | **Spec**: [`spec.md`](./spec.md) | **Design**: [`design.md`](./design.md) | **UI contract**: [`mockup.html`](./mockup.html)

**Input**: Feature specification from `spec.md`; the authoritative architecture lives in `design.md` (reviewer-passed, 2026-09-27). This plan sets the technical context, runs the Constitution gate, lays out the concrete file structure, and resolves the three decisions `design.md` deferred to plan. It references `design.md` rather than repeating it.

## Summary

Add GitLab as the **third integration** on the existing kernel webhook platform (`src/lib/integrations/webhooks/`): a `gitlab` / `issues` `WebhookHandler` (constant-time `X-Gitlab-Token` verify, payload parse) that emits a typed event onto the 034 event bus, plus a headless classifier/router on the bus that reads the issue's **classification label** (`Bug`/`Feature`/`Enhancement`) and dispatches to an **existing** resolution pipeline — self-heal (bugs) or the Build Studio spec pipeline (features/improvements) — driving the issue's GitLab side (notes, `agent:*` workflow labels, draft MR / preview) through the **GitLab MCP server**, which is the sole GitLab credential holder and I/O layer. The kernel holds no GitLab credential, registers no provider-side webhook (registration is manual), and the "issue-update glue" is a brief section, not a kernel module. Propose-only: everything stops at `agent:preview-ready`.

The single load-bearing assumption (design R6) is that **gitlab-mcp tools are exposed to headless `runSubAgent` children**. The first implementation spike verifies it before any other run code is written.

## Technical Context

**Language/Version**: TypeScript, within the existing BOS Next.js (App Router) codebase.

**Primary Dependencies**: **none new.** Reuses, by reference: the webhook platform (`src/lib/integrations/webhooks/` — route, `WebhookHandler`, registry, manager, secrets store, idempotency, `emitIntegrationEvent`); the 034 event bus (`src/lib/events/`); the self-heal pipeline (spec 031, `src/lib/self-heal/` — trigger + autonomous slow path + `brief.ts`); the spec-kit Build Studio method; and the GitLab MCP server (the credential + I/O layer, D2). No new npm packages, no `package.json`/lockfile change (Constitution VII).

**Storage**: files. Non-sensitive per-issue resolution state + connection flag in `data/integrations/gitlab/state.json` (the real `IntegrationState` shape, via `readState`/`mutateState` — `src/lib/integrations/state/store.ts`). The webhook verification token in the SecretsStore (`webhook:gitlab/issues` key, with primary/previous rotation grace — `src/lib/integrations/webhooks/store.ts`). Deep state in the self-heal case store (bugs) and the spec artifacts + feature branch (features). No new database, no migrations.

**Testing**: Vitest unit (BOS convention) + Playwright e2e `e2e/002-gitlab-issue-webhook.spec.ts`, mirroring the 034/031 e2e conventions (temp data dirs, reset seams, self-cleaning teardown, the M5 environmental guard). See `design.md` §Testing strategy.

**Target Platform**: the Next.js app process (server-side handlers, route, headless executor) + one Settings UI section (React).

**Project Type**: web-service (Next.js) — a new integration service under `src/lib/integrations/services/gitlab/` plus a `ServiceConfigView` section.

**Performance Goals**: NFR-003 — valid POST → event on the bus in < 2 s (p95). The receiver path is pure parse + emit (no GitLab I/O, no MCP call in the request path), so this is inherited from the existing platform, not at risk.

**Constraints**: propose-only (NFR-004); no kernel GitLab credential / no kernel GitLab I/O (NFR-007); constant-time verify, 404-not-401, no body/token logging (NFR-001); concurrent issues isolated, no shared mutable pipeline state (NFR-005 / FR-029).

**Scale/Scope**: 1 new integration (`gitlab` / `issues`), 2 reused pipelines, a fixed 8-label vocabulary, 1 Settings section, 1 headless router + 1 thin per-issue state map. The genuinely new runtime surface is the bug-path delivery tail (design R1).

No `NEEDS CLARIFICATION` items remain — every technical detail is grounded in verified source (see `design.md` §Context, §R0).

## Constitution Check

*GATE: must pass before implementation. Re-checked post-design; still passes.*

| Principle | Status | Notes |
|---|---|---|
| I. Spec-Driven (NON-NEGOTIABLE) | ✅ | `spec.md` is the source of truth; `design.md` and this plan reference it. Code will be the derivative. Pipeline followed: specify → clarify → design → plan → tasks → implement. |
| II. Server Authority & SSR Boundary | ✅ | All new state is server-side: the token in the SecretsStore, the per-issue map in `data/integrations/gitlab/state.json`, the router as a server-side headless handler, the receiver as a `route.ts`. The client sees only the masked/reveal token + last-received (existing `WebhookSection` pattern). **New domain types follow the existing integration type home** (`src/lib/integrations/services/gitlab/`), consistent with how `telegram`/`gsuite` define their handler types — not `src/os/types.ts`, matching the established integration-platform convention (`src/lib/integrations/types.ts`). |
| III. Always Delegate; Claude Codes | ✅ | `implement` delegates to the Developer (Claude) via `dev_delegate` (bos-core target). Build Studio writes no source. |
| IV. Minimize Blast Radius (NON-NEGOTIABLE) | ✅ | All changes on `bos/002-gitlab-issue-webhook`; Developer edits an isolated worktree; candidate self-tests (Playwright) before promote. |
| V. The VFS Is Not the Source | ✅ | Spec/plan/design are VFS artifacts (correct). Code changes go through the Developer editing `src/`. Runtime state persists under `data/` (gitignored). |
| VI. Specs & Docs Stay in Sync (NON-NEGOTIABLE) | ✅ (obligation added) | Implementation **must** update docs in the same change — see *Documentation Obligation* below. This is the one gate that turns a "pass" into a deliverable list. |
| VII. Respect Boundaries | ✅ | No secrets/`package.json`/lockfile/build-config changes (no new dependencies). The only edit to an existing subsystem file outside the new service is `src/lib/self-heal/brief.ts` (the augmenter seam), which **must be a no-op for all non-gitlab cases** (regression-tested, design R7c). Quality gates (`tsc --noEmit`, lint) must pass. |

**Technology constraints**: within the existing stack (Next.js App Router + React + Zustand; OpenAI-compatible AI providers; self-describing apps). No new app; the new Settings section is a React component in the existing Integrations view.

**Gate result: PASS.** No Complexity Tracking required (no principle violations to justify).

## Project Structure

### Documentation (this feature)

```text
/Specs/user-specs/integrations/002-gitlab-issue-webhook/
├── spec.md              # binding contract (31 FR / 7 NFR / 9 SC)
├── design.md            # binding architecture (reviewer-passed) — data model, contracts, flows
├── plan.md              # this file
├── mockup.html          # binding UI contract
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase: tasks (NOT created by plan)
```

No separate `research.md` / `data-model.md` / `contracts/` / `quickstart.md` — see *Design Notes → Companion artifacts* for the rationale (the design already carries each).

### Source Code (repository root)

```text
src/lib/integrations/services/gitlab/
├── manifest.ts                  # IntegrationManifest id "gitlab" — services:[issues], oauthConfig = no-flow descriptor
├── index.ts                     # barrel: registerIntegration + registerGitLabIssueSpine + brief-augmenter registration
├── adapters/
│   └── issue-webhook.ts         # WebhookHandler: verify (local constant-time token) + receive (parse object_kind:issue); NO onEnable/onDisable
├── issue-router.ts              # core headless executor on com.bos.gitlab.issues.* — I/O-free; 8-label vocab constants; feature-intent brief builder
├── resolution-state.ts          # thin per-issue map + connection + lastReceived in state.json (serialized writes)
└── brief-section.ts             # renderGitLabBriefSection({mode: bug|feature|ambiguity|validate}) — the GitLab-reporting protocol as brief text

src/lib/self-heal/
├── brief-augmenters.ts          # NEW: tiny registry; buildAutonomousBrief appends matching augmenter sections
└── brief.ts                     # EDIT: one-line call into the augmenter registry (no-op for non-gitlab cases)

src/components/apps/settings/integrations/
├── GitLabIssuesSection.tsx      # NEW: MCP-connection card, target project, manual-registration steps, label status, last-received (reuses WebhookSection)
└── ServiceConfigView.tsx        # EDIT: render <GitLabIssuesSection> for gitlab/issues

src/lib/integrations/index.ts                  # EDIT: import "./services/gitlab"; (barrel pattern of gsuite/telegram)
src/lib/integrations/webhooks/registry.ts      # EDIT: static HANDLERS entry gitlab: { issues: issueWebhookHandler }

tests/integrations/gitlab/                     # unit: verify, receive, router, resolution-state, brief-section, label-vocabulary
e2e/002-gitlab-issue-webhook.spec.ts           # e2e: webhook→bus, driven Bug→agent:preview-ready (draft MR), driven Feature→agent:preview-ready, self-noise, closed-mid-run
```

**Structure Decision**: a single new integration service folder under the existing `src/lib/integrations/services/` (mirroring `telegram`/`gsuite`), one new Settings section under the existing Integrations view, and exactly two surgical edits into existing files (`integrations/index.ts` barrel, `webhooks/registry.ts` entry) plus one tightly-scoped self-heal edit (the augmenter seam, R7c-regression-tested). This is the minimal-blast-radius shape the platform is built for ("adding a handler = one entry + the impl" — `webhooks/registry.ts`). No new app, no new top-level package, no new runtime process.

## Plan-Time Decisions

`design.md` deferred three decisions to plan. Resolved here, each grounded in the project's standing architectural principles (generic platform mechanisms over special-cases; no workarounds; the kernel stays I/O-free; fuzzy work is agent-owned).

**D-R2 — Feature-run cost bound → rely on the platform's per-run limits; do NOT add a bespoke cap.**
Feature runs bypass `selfHealIntake`, so self-heal's daily cost cap / single in-flight slot / case-bound observer do not apply (design §Boundedness). The trigger volume is already tightly bounded by the router's `issueKey` gate — **one run per issue, ever** — which is far stricter than self-heal's event-stream trigger. The platform's per-run limits (max steps / tool timeout, `src/lib/agent/subagents/runner.ts`) are the generic bound every `runSubAgent` inherits. A bespoke feature-run dollar cap would be a special-case mechanism the platform has for no other run — rejected on that basis. If the platform later adds a generic run cost cap, feature runs inherit it for free. No slot is reintroduced (FR-029/NFR-005 require concurrency).

**D-OQ1 — Abandonment (FR-028) → `agent:failed` + an explicit abandonment note; no sixth label.**
The Label Model is a fixed 5-label set; adding `agent:abandoned` would change the user-approved label contract. An issue closed mid-run *is* a failure to deliver a preview, so `agent:failed` is the honest terminal. The distinction is carried by the note ("issue closed mid-run — reopened work requires a new trigger"), so SC-007 (a human can state status from label + notes) holds: `agent:failed` + that note is legible as "abandoned," not "the fix was wrong."

**D-OQ3 — `Enhancement` targeting → brief-based, agent-resolved; the router does NOT pre-resolve the spec project.**
Locating "which existing feature" from an issue body is fuzzy name-matching that needs a failure channel. The router is I/O-free and headless — it cannot go `agent:needs-input`. The resolving run has the issue body, spec-store access, *and* the needs-input channel, so it is the correct owner. v1: the run locates the referenced existing spec project; if the reference is absent or ambiguous ⇒ `agent:needs-input` with the specific question. (A future router-side local spec-store pre-resolution is possible but not v1 — it would push ungraceable fuzzy matching into the kernel.)

## Design Notes

- **Architecture authority is `design.md`.** Read it for the Context (reused-vs-new map, every claim source-cited), the Data model (IntegrationEvent shape, thin per-issue state inside the real `IntegrationState`, the 8-label vocabulary, the 5-label workflow state machine with the `agent:queued`-reserved note, the connection-state derivation), the Resolution flow (bug + feature, each stage → label → note, incl. needs-input/failed/abandoned), the Headless pipeline-driving mechanism (brief-augmenter seam + feature-intent brief over the self-heal autonomous slow path), Security & invariants, Testing strategy, and Risks R0–R9.
- **Reviewer findings M1–M4 + m1–m6 were applied to `design.md` in the 2026-09-27 revision and verified against the revised file.** The plan assumes the corrected design: no router-side `create_project` (M1 — the run's first step does it, on the router-pre-set conversation branch); the feature run observer is new thin code wrapping `StuckDetector` (M2); feature runs inherit no self-heal cap (M3, see D-R2); 0-label events create no map entry so FR-014 label-after-open is never gated (M4); the data model sits inside the real `IntegrationState` with the top-level `connected` boolean as the receiver gate (m1); local constant-time token compare + deliberate stricter-than-Telegram divergence (m2); complete `emitIntegrationEvent` shape (m3); `create_branch` before `push_files` in the delivery tail (m4); `agent:queued` reserved (m5); `agent:failed` mirror keys off the case store (m6).
- **The single load-bearing assumption (R6):** gitlab-mcp tools must be reachable from headless `runSubAgent` children. The first spike is one validation sub-agent run; if it fails, the feature is non-functional end-to-end despite working intake (no kernel fallback by design). Flagged for the `tasks` phase to order first.
- **Companion artifacts (conscious omission, per the plan reference's "only when warranted"):**
  - *research.md* — not warranted: `design.md` §Context + §R0 already resolve every technical unknown against verified source; there are no open `NEEDS CLARIFICATION` items.
  - *data-model.md* — not warranted: `design.md` §Data model is the complete data model (event, state, labels, state machine, connection); a separate file would duplicate it.
  - *contracts/* — not warranted: the receiver reuses the existing generic webhook route (not new API surface); the emitted contract is the existing `IntegrationEvent` + the systematic-fallback bus type; the MCP surface is the existing gitlab-mcp tools. The only *new* contracts — the fixed 8-label vocabulary and the brief's allowed-mutation whitelist — are already specified in `design.md` (§Data model, §Security & invariants) and `spec.md` (Label Model).
  - *quickstart.md* — not warranted: `design.md` §Testing strategy enumerates the runnable validation scenarios (the 10 e2e cases) and the M5 prerequisites/guard; it is the validation guide.

## Documentation Obligation (Constitution VI)

The implementation change MUST include, in the same branch:
- **`docs/usage`** (end users): the GitLab Issues integration — configuring the gitlab-mcp server, validating the connection, enabling the webhook, the manual project-webhook registration steps (URL + token, Issues events only), the label model (classification vs workflow, the 5-state lifecycle), and the propose-only boundary (what `agent:preview-ready` means, that promotion is the user's). Mirror how the existing Telegram/Gmail integrations are documented.
- **`docs/dev`** (developer agent): the new integration handler on the webhook platform (the `gitlab` barrel + registry entry pattern), the I/O-free `issue-router.ts` headless executor + the thin per-issue state map, the **self-heal brief-augmenter seam** (`brief-augmenters.ts` + the one-line `brief.ts` call, and its no-op guarantee for non-gitlab cases), and the GitLab-reporting brief protocol (`brief-section.ts`).

Drift, if any emerges in `converge`, is recorded per the constitution (in the spec store's discrepancies log).
