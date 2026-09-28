# Implementation Plan: GitLab Integration (Webhook → Triage)

**Branch**: `002-gitlab-integration` | **Date**: 2026-09-27 | **Spec**: [spec.md](./spec.md) · **Design**: [design.md](./design.md)

**Input**: Feature specification from `/Specs/user-specs/integrations/002-gitlab-integration/spec.md`
**Target**: `bos-core` (confirms `spec.md` App Target — see design.md §1)

## Summary

Add a **webhook-only** GitLab integration to BOS. When a new GitLab issue (or MR) is opened in a hooked project, GitLab POSTs to BOS's generic webhook receiver; BOS verifies the shared token, dedups, and emits an event into the event kernel (`gitlab.issue.opened` / `gitlab.mr.opened`). A default **core headless handler** (`gitlab-issue-triage`, enabled by default, independently toggleable) is a **thin dispatcher**: it forwards the issue context to a **user-configurable agent** (the seeded **Gitlab Issue Agent**) via `runSubAgent`. That agent's **skill** (`gitlab-issue-triage`) is the pipeline driver — it classifies the issue by label (`Bug`/`Feature`/`Improvement`, agent-inference fallback), runs the matching workflow, and updates the issue (comments + status labels) **through the agent's configured GitLab MCP server**, ending at the terminal `Needs Review` state. The handler and webhook are two independent switches; the user can disable the handler and route the event elsewhere (e.g. the Workflow app). **No GitLab credential lives in BOS** (R1/R4); **self-heal is fully separate** (R6).

Technical approach: **reuse the generic webhook framework + event kernel + agent/skill model**, not special-case. Two minimal platform additions (both additive, back-compatible): an `authless` manifest flag (connected-by-default) and a `mintSecret` opt-out on webhook enable.

## Technical Context

- **Language/Version**: TypeScript 5.x (strict, `erasableSyntaxOnly`) on Node (Next.js runtime `nodejs` for API routes).
- **Primary Dependencies**: none new. Reuses: Next.js App Router, the integration framework (`src/lib/integrations`), the event kernel (`src/lib/events`), the sub-agent runtime (`src/lib/agent/subagents/runner.ts`), the MCP gateway (`src/lib/mcp/gateway.ts`), Tailwind + lucide for the panel.
- **Storage**: integration state in `state.json` (`src/lib/integrations/state/`); webhook secret in the integration SecretsStore (`webhook:webhooks`); seeded agent/skill as markdown under `data/` (from `seed/`). No new persistence layer.
- **Testing**: unit (Vitest) for the webhook handler (verify/filter/projection), the dispatcher's post-run failure detection, and the label constants; e2e (Playwright) for the end-to-end loop — `e2e/002-gitlab-integration.spec.ts`. Follow the existing self-heal `e2e/031-self-healing.spec.ts` + unit-test conventions.
- **Target Platform**: the running BrowserOS server (Node). Previewed via the Supervisor; promoted code-only.
- **Project Type**: web application (Next.js App Router) — server-side integration + React settings panel.
- **Performance Goals**: webhook receiver acks promptly (well under GitLab's retry timeout); the agent run is fire-and-forget (multi-minute, async) and must never block the HTTP response (FR-007).
- **Constraints**: `X-Gitlab-Token` is a **plain string match** (not HMAC) — GitLab's scheme; 404 (not 401) when disabled/disconnected to avoid state leaks; in-memory dedup ring; no BOS-owned GitLab credential.
- **Scale/Scope**: one integration + one handler + one seeded agent + one skill + one settings panel; ~14 new files, ~9 modified, 2 platform additions.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design (see design.md §2). All gates pass.*

| Principle | Status |
|---|---|
| **I. Spec-Driven** | ✅ `spec.md` + `design.md` exist and are approved (R1–R6). |
| **II. Server Authority & SSR Boundary** | ✅ Webhook parsing, secret verification, state, event emission, and the dispatcher are all server-only. The webhook token is returned to the client **exactly once** (the reveal). **No GitLab credential is stored in BOS** (R1/R4). |
| **III. Always Delegate; Claude Codes** | ✅ Implementation is delegated to the Developer (Claude) sub-agent via `dev_delegate` on `bos/002-gitlab-integration`. At runtime the dispatcher hands off to a `local` agent (`runSubAgent`) — a headless local agent (it reads/classifies/drafts + calls MCP tools; it does not edit BOS source). No Claude agent used for a non-development task. |
| **IV. Minimize Blast Radius** | ✅ All `src/` changes on the feature branch; the two platform additions (`authless`, `mintSecret`) are additive and back-compatible (G Suite/Telegram behavior unchanged). Previewed + self-tested before promote. |
| **V. The VFS Is Not the Source** | ✅ Source changes via the Developer on the branch, not VFS. The agent's deliverable is a comment posted to GitLab via MCP (not a BOS spec-store write). Runtime state under `data/` (gitignored). |
| **VI. Specs & Docs Stay in Sync** | ✅ `docs/usage/integrations/gitlab.md` + a `docs/dev/integrations/` note are in the file plan. Spec + design + plan present. |
| **VII. Respect Boundaries** | ✅ No `package.json`/lockfile/build changes; no new external dependency (the GitLab MCP server is user-configured external tooling). Gates: `npx tsc --noEmit` + `npm run lint`. No `npm run build` while `next dev` is running. |

**Gate result: PASS. No Complexity Tracking required.**

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/integrations/002-gitlab-integration/
├── spec.md              # Feature spec (approved, R1–R6)
├── design.md            # v5 design (architect + 2 reviewer rounds) — the research record
├── plan.md              # This file
├── data-model.md        # Data shapes (webhook config, event payload, labels, agent/skill frontmatter)
├── contracts/
│   ├── gitlab-webhook.md     # Inbound GitLab webhook contract (token, payload, filtering)
│   ├── bos-event.md          # Emitted BOS event contract (gitlab.issue.opened / gitlab.mr.opened)
│   └── dispatcher-task.md    # Handler → agent dispatch task contract
├── quickstart.md        # End-to-end validation guide (SC-001 manual path)
├── mockup.html          # Binding UI contract (Settings → Integrations → GitLab)
└── checklists/requirements.md
```

### Source Code (repository root)

```text
# ── NEW ─────────────────────────────────────────────────────────────
src/lib/integrations/services/gitlab/
├── manifest.ts                    # GITLAB_MANIFEST (authless:true, service "webhooks", placeholder oauthConfig)
├── index.ts                       # registerIntegration + registerAdapter side effects
├── labels.ts                      # classification + status label constants (shared: skill, panel, handler)
├── adapters/
│   ├── webhook.ts                 # GitLabWebhookHandler: verify (X-Gitlab-Token plain match), receive (parse+filter+project)
│   └── webhooks-adapter.ts        # zero-method ServiceAdapter, capabilities { webhook: true }
└── triage/
    ├── handler.ts                 # gitlab-issue-triage core headless handler: register + prompt-ack + fire-and-forget dispatch
    └── dispatch.ts                # resolve selected agent, build issue-context task, runSubAgent, post-run outcome-based failure detection

seed/agents/gitlab-issue-agent/AGENT.md        # seeded Gitlab Issue Agent (type local; skills:[gitlab-issue-triage], mcp allowlist)
seed/skills/gitlab-issue-triage/SKILL.md       # the triage skill = pipeline driver (classify → workflow → status labels → Needs Review)

src/components/apps/settings/integrations/
├── GitLabDetailView.tsx           # dedicated detail view (header + WebhookSection + WorkflowSection + LabelsReference)
├── GitLabWorkflowSection.tsx      # "Workflow" card: agent picker (default Gitlab Issue Agent) + skill note + Events toggle pointer
└── GitLabLabelsReference.tsx      # classification + status chips (sourced from labels.ts)

docs/
├── usage/integrations/gitlab.md   # end-user: enable webhook, generate token, register in GitLab, configure the agent
└── dev/integrations/gitlab-triage.md  # dev: webhook-only/authless pattern, thin-dispatcher pattern, post-run failure detection

# ── MODIFIED ────────────────────────────────────────────────────────
src/lib/integrations/types.ts      # + authless?: boolean on IntegrationManifest
src/lib/integrations/state/store.ts# readState: coerce connected=true when manifest is authless
src/lib/integrations/index.ts      # + import "./services/gitlab"
src/lib/integrations/webhooks/registry.ts        # + gitlab: { webhooks: new GitLabWebhookHandler() }
src/lib/integrations/webhooks/manager.ts         # enableWebhook: + mintSecret?: boolean (default true)
src/app/api/integrations/[id]/services/[serviceId]/webhook/route.ts  # POST enable: forward body.mintSecret
src/lib/events/legacy-mapping.ts   # + KNOWN_TYPES (issue-opened/mr-opened) + KNOWN_NAMES.gitlab + legacyFriendlySummary
src/instrumentation.ts             # + registerGitLabTriageHandler() after startEventKernel()
src/components/apps/settings/IntegrationsTab.tsx  # route gitlab → GitLabDetailView; generic fallback excludes gitlab

# ── TESTS ───────────────────────────────────────────────────────────
e2e/002-gitlab-integration.spec.ts # end-to-end: webhook → event → dispatch → agent → issue update
src/lib/integrations/services/gitlab/__tests__/   # unit: webhook verify/receive/filter/projection; dispatch failure detection; labels
```

**Structure Decision**: The gitlab integration follows the exact container shape of the `telegram`/`gsuite` integrations (`src/lib/integrations/services/<id>/{manifest,index,adapters}`) — it is a **service of the integration framework**, not a new top-level module. The handler is a **core headless** handler (registered from `instrumentation.ts` like `spine-handler.ts`), not a worker service. The settings panel is a **dedicated detail view** (like `TelegramDetailView`), not the generic `IntegrationDetailView` (which would render the git-provider OAuth panel — design.md ADR-6). Seed agent + skill live under `seed/` and sync to `data/` at boot (`seed-sync.ts`).

## Design Notes (folded from design.md)

The design step produced the full source-verified analysis (classification, context/container/component, ADRs) — it is the research record for this plan. Key decisions the implementer must honor (see design.md §6 for the full ADR set + §9 review history):

- **Two registries, two registrations** (ADR-8): `gitlab/webhooks` in both the webhooks *handler* registry and the *actions adapter* registry (zero-method, `capabilities:{webhook:true}`).
- **`authless` = connected-by-default** (ADR-1): derived in `readState`, not persisted — no per-feature branch in the receiver.
- **`mintSecret` opt-out** (ADR-1b): gitlab enable passes `mintSecret:false`; "Generate token" = the stock rotate (one-time reveal). "Enabled, no token" is a real, safe state.
- **Event-type mapping** (ADR-2): data-table entries in `legacy-mapping.ts` (`KNOWN_TYPES`, `KNOWN_NAMES`, `legacyFriendlySummary`) — no code-logic branch.
- **Thin dispatcher** (ADR-3, ADR-12): the handler is `registerCoreExecutor` + `api.register({declaredBy:"core", mode:"headless"})`, **omitting `enabled`** so disable persists across restarts (F1). Prompt-ack + fire-and-forget `void dispatchToAgent(record)` (never block the 30 s ack window).
- **Post-run, outcome-based failure detection** (ADR-4, the v5 blocker fix): **do not** rely on a `try/catch` around `runSubAgent` — the gateway is in-band and the agent loop "never dies of tools," so the run returns clean even with no MCP. Detect failure by: (a) agent missing, (b) `run.error`/`endedReason==="error"`, or (c) **no successful `mcp_tool_call`** in the collected `onEvent` tool results → emit `com.bos.gitlab.triage.failed`. Keep it outcome-based (no pre-flight).
- **Agent + skill = unit of capability** (ADR-5, ADR-12): seed a `local` `Gitlab Issue Agent` (frontmatter `skills:["gitlab-issue-triage"]`, `mcp` allowlist default empty=all) + the `gitlab-issue-triage` skill. The panel's agent picker (`state.services.webhooks.config.triageAgentId`) selects any agent. `contentOnly:true` is **inert for a local agent** (passed only for precedent consistency).
- **Skill is model-followed guidance, not a mechanism** (ADR-12, §3.8): the single-active-status invariant (FR-014) and the pipeline are *expected* from the model, not code-enforced — write the skill accordingly (it owns the label swap + resume/idempotency + the Q2 "never close/commit" boundary).
- **Loop prevention is structural** (ADR-9): the receiver filter (`object_kind ∈ {issue,merge_request}` **and** `action === "open"`) categorically excludes the agent's own `note`/`update` webhooks. No sender-id/timestamp guard needed.
- **`gitlab` string overlap** (ADR-6): the gitlab *integration* id is distinct from the git-remote OAuth *provider* id — do not reuse git-providers code; route to a dedicated `GitLabDetailView`.

### Open items (carry into tasks; none block)

- **WebhookSection parity** (low stakes): accept stock labels ("Rotate secret", event-type filter + save chrome) or add a small label prop for exact mockup parity — decide during implement; functionally identical.
- **"Draft spec" delivery** (design open q c): post as a GitLab **issue comment** (not a BOS spec-store / Build Studio write) — confirm with the user at implement if ambiguous.
- **Seed agent `mcp` default** (design open q d): default `mcp: []` (= all servers) for naming robustness; the panel/agent lets the user narrow it.
