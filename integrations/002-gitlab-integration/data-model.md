# Data Model — GitLab Integration (Webhook → Triage)

Entities from [`spec.md`](./spec.md) Key Entities, with their concrete shapes. This feature stores **no GitLab credential** (R1/R4) — the only secrets are the webhook verification token and the (external, user-configured) GitLab MCP server's own auth.

## 1. GitLab Webhook Config

The integration's `webhooks` service stores two sibling blocks under its service config (`IntegrationServiceState.config`):

**`config.webhook`** — a `WebhookConfig` (framework type, `src/lib/integrations/webhooks/types.ts`):

| Field | Type | Notes |
|---|---|---|
| `enabled` | `boolean` | The webhook switch. When false, the receiver 404s. |
| `eventTypes` | `string[]?` | Stock `WebhookSection` filter (empty = all). The receiver applies it. |
| `extras` | `Record<string,unknown>?` | Service-specific (none required for gitlab in v1). |

**`config.triageAgentId`** — `string` (the **agent picker**, FR-017):

| Field | Type | Notes |
|---|---|---|
| `triageAgentId` | `string` | Which agent handles new issues. Default `"gitlab-issue-agent"`. Sibling of `config.webhook` — same shape as Telegram's `config.agentConfig.agentId`. The dispatcher reads it (design §3.7.1). |

> Both are written via the existing `writeWebhookConfig` / service-config mutation paths; no new store. `triageAgentId` is user-set from the panel's picker.

## 2. Webhook Secrets

`WebhookSecrets` (framework type), persisted in the integration SecretsStore under `webhook:webhooks` (never in state.json):

| Field | Type | Notes |
|---|---|---|
| `primary` | `string` | The current token — the value the user pastes into GitLab's "Secret token" (sent by GitLab as `X-Gitlab-Token`). Generated on "Generate token" (first rotate); each rotate mints a new one. |
| `previous` | `string?` | Last rotated token; still verified for a grace window. |
| `rotatedAt` | `number` | Epoch-ms of last rotation. |

- For gitlab, `mintSecret:false` on enable (ADR-1b) → **no token until the user clicks "Generate token."** Until then, inbound requests 401.
- Verification is a **plain string match** against `[previous, primary]` (timing-safe), not HMAC.

## 3. GitLab Event

Two-stage: the handler's `IntegrationEvent` (inbound) → the BOS event (after `legacy-mapping` + `api.emit`).

**IntegrationEvent** (returned by `GitLabWebhookHandler.receive`):

```
{ type: "issue-opened" | "mr-opened",
  service: "gitlab/webhooks",
  timestamp: number,
  data: <projection> }
```

**`data` projection** (bounded — FR: description truncated to ~2 000 chars):

| Field | Type | Issue | MR | Notes |
|---|---|---|---|---|
| `projectId` | `number` | ✓ | ✓ | `project.id` (numeric — use for API/MCP calls). |
| `iid` | `number` | ✓ | ✓ | Project-scoped `!`/`#` number. |
| `title` | `string` | ✓ | ✓ | |
| `labels` | `string[]` | ✓ | ✓ | `object_attributes.labels[].title` (or string form). |
| `author` | `{ name, username }` | ✓ | ✓ | Top-level `user`. |
| `url` | `string` | ✓ | ✓ | `object_attributes.url`. |
| `description` | `string` | ✓ | ✓ | Truncated. |
| `sourceBranch` | `string` | — | ✓ | `object_attributes.source_branch`. |
| `targetBranch` | `string` | — | ✓ | `object_attributes.target_branch`. |

**BOS event** (after mapping — ADR-2):

| Field | Value |
|---|---|
| `type` | `gitlab.issue.opened` (from `issue-opened`) or `gitlab.mr.opened` (from `mr-opened`) |
| `source` | `{ appId: "gitlab", name: "GitLab" }` (from `KNOWN_NAMES.gitlab`) |
| `payload` | `{ ...data, _integrationType, _integrationService, summary }` |
| `summary` | `"issue #<iid> <title>"` / `"MR !<iid> <title>"` (the `legacyFriendlySummary` branch) |

Both types pass `isValidEventType` (dot-separated lowercase).

## 4. Triage Handler (event kernel `HandlerRegistration`)

| Field | Value |
|---|---|
| `handlerId` | `"gitlab-issue-triage"` |
| `eventType` | `"gitlab.issue.opened"` (exact — issue-only per Q1; **not** a prefix) |
| `mode` | `"headless"` |
| `ownerId` | `"core"` |
| `declaredBy` | `"core"` (⇒ `isHandlerActive` is a plain `enabled` boolean; Event Viewer toggle is authoritative — ADR-7) |
| `displayName` | `"GitLab issue triage"` |
| `enabled` | **omitted on register** (default-true-on-first-only; user can disable, persists across restarts — F1). |
| `timeoutMs` | `30000` (default; irrelevant to the actual run — the handler prompt-acks, ADR-3). |

## 5. Gitlab Issue Agent (`AGENT.md` frontmatter)

Seeded from `seed/agents/gitlab-issue-agent/AGENT.md` → `data/agents/gitlab-issue-agent/` at boot (`seed-sync.ts`).

| Field | Value | Notes |
|---|---|---|
| `id` | `gitlab-issue-agent` | |
| `type` | `local` | Headless local agent (reads/classifies/drafts + calls MCP; no BOS source edits). |
| `skills` | `["gitlab-issue-triage"]` | Pins the triage skill (re-review note: must be explicit). |
| `mcp` | `[]` | Empty = all servers (robust to the user's server naming; user may narrow). The GitLab MCP server must be among them to act. |
| `tools` | MCP tools + text/skill scope | No source-edit / VFS-spec-write tools. |
| system prompt | "You are the Gitlab Issue Agent… use the `gitlab-issue-triage` skill…" | |

## 6. Triage Skill (`SKILL.md`)

`seed/skills/gitlab-issue-triage/SKILL.md` → `data/skills/gitlab-issue-triage/` at boot. The **pipeline driver** (model-followed guidance — design §3.8): read the issue via MCP → classify (labels first, inference fallback) → idempotency/resume (existing status label ⇒ resume) → run the matching pipeline applying the transient status label → terminal `Needs Review`. Owns the single-active-status swap and the Q2 "never close/commit" boundary.

## 7. Labels (`labels.ts` — shared constants)

**Classification labels** (additive, persistent — one determines the workflow):

| Label | Workflow |
|---|---|
| `Bug` | analyze → verify → propose fix |
| `Feature` | draft a spec for a new feature |
| `Improvement` | draft a spec to improve an existing feature |

**Status labels** (transient, exactly one active — the pipeline stage):

| Label | Meaning | Path |
|---|---|---|
| `Triage` | received, being classified | all |
| `Analyzing` | analyzing the report | Bug |
| `Verifying` | assessing plausibility / root cause | Bug |
| `Fix Proposed` | a fix was posted in a comment | Bug |
| `Spec Drafted` | a spec was posted in a comment | Feature, Improvement |
| `Needs Review` | **terminal** — agent finished; a human verifies before finalizing (Q2) | all |
| `Needs Input` | blocked — the agent needs more info from the reporter | any |
| `Failed` | the run errored; a failure note was posted | any |

**State transitions** (Bug path, the richest):

```
(open, no status) ──classify──▶ Triage
        Triage ──▶ Analyzing ──▶ Verifying ──▶ Fix Proposed ──▶ Needs Review   (terminal)
        any ──(blocked)──▶ Needs Input            any ──(error)──▶ Failed
```

Feature/Improvement path: `Triage ──▶ (work) ──▶ Spec Drafted ──▶ Needs Review`.

**Loop prevention** (structural, ADR-9): the skill's MCP writes produce GitLab `note` (comment) and `issue`/`merge_request` `action:"update"` (label) webhooks — both excluded by the receiver's `object_kind ∈ {issue, merge_request}` **and** `action === "open"` filter, so they never re-trigger the handler.

## Relationships

```
GitLab integration (authless)
└─ service "webhooks"
   ├─ config.webhook (WebhookConfig) ── verified against ── WebhookSecrets (SecretsStore)
   ├─ config.triageAgentId ── selects ── Agent (data/agents/<id>)
   └─ inbound webhook ── emits ── BOS event (gitlab.issue.opened / gitlab.mr.opened)
                                        │
Triage Handler (core, gitlab-issue-triage) ── consumes gitlab.issue.opened ──▶ runSubAgent(triageAgentId)
                                                                                   │
                                                                              Agent + skill
                                                                                   │  (via the agent's MCP allowlist)
                                                                              GitLab MCP server (external, user-configured)
                                                                                   │
                                                                              GitLab REST (read issue / comment / label)
```
