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
{ type: "issue-opened" | "mr-opened" | "issue-noted",
  service: "gitlab/webhooks",
  timestamp: number,
  data: <projection> }
```

**`data` projection — items** (bounded — FR: description truncated to ~2 000 chars):

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

**`data` projection — `issue-noted`** (R7 / FR-022; bounded — note truncated to ~2 000 chars):

| Field | Type | Source field | Notes |
|---|---|---|---|
| `projectId` | `number` | `project.id` | |
| `projectPath` | `string` | `project.path_with_namespace` | |
| `iid` | `number` | **`issue.iid`** (top level) | The issue's PROJECT-SCOPED number. See the warning below. |
| `note` | `string` | `object_attributes.note` | Truncated. |
| `author` | `{ name, username }` | top-level `user` | The **comment's** author, not the issue's. |
| `url` | `string` | `object_attributes.url` | Issue URL + `#note_<id>` anchor. |

> **The note-hook field layout, verified.** A note hook's `object_attributes` describes the **comment**: `note`, `noteable_type` (`"Issue"` / `"MergeRequest"` / `"Snippet"` / `"Commit"`), `noteable_id`, `url`, `action`. `noteable_id` is the issue's **global/database** id, and GitLab emits **no `noteable_iid`** — confirmed against `lib/gitlab/hook_data/note_builder.rb`'s `SAFE_HOOK_ATTRIBUTES` and the documented "Comment on an issue" payload. The project-scoped iid, which is what every GitLab MCP call takes, is on the **top-level `issue`** object. The implementation reads `noteable_iid` first and falls back to `issue.iid` for self-hosted tolerance, but `issue.iid` is the field that answers; `webhook.receive.test.ts` pins this with a payload whose two ids deliberately differ (900 vs 17).
>
> The projection carries **no** "is this the agent?" flag and cannot: BOS holds no GitLab credential and does not know which user the MCP token acts as (R1/R4). That determination is the agent's (design ADR-9/ADR-13).

**BOS event** (after mapping — ADR-2):

| Field | Value |
|---|---|
| `type` | `gitlab.issue.opened` (from `issue-opened`), `gitlab.mr.opened` (from `mr-opened`), or `gitlab.issue.noted` (from `issue-noted`) |
| `source` | `{ appId: "gitlab", name: "GitLab" }` (from `KNOWN_NAMES.gitlab`) |
| `payload` | `{ ...data, _integrationType, _integrationService, summary }` |
| `summary` | `"issue #<iid> <title>"` / `"MR !<iid> <title>"` / `"comment on issue #<iid>"` (the `legacyFriendlySummary` branches) |

All three types pass `isValidEventType` (dot-separated lowercase). The two `gitlab.issue.*` names are a pinned pair — the handler subscribes to that prefix (§4).

## 4. Triage Handler (event kernel `HandlerRegistration`)

| Field | Value |
|---|---|
| `handlerId` | `"gitlab-issue-triage"` |
| `eventType` | `"gitlab.issue.*"` — a **prefix pattern** (R7), so one handler covers `gitlab.issue.opened` and `gitlab.issue.noted`. `isValidRegistrationType` accepts `prefix.*`; `typeMatches` resolves it per event. The prefix stops at `issue`: `gitlab.*` would sweep in `gitlab.mr.opened`, which is out of scope per Q1. |
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

`seed/skills/gitlab-issue-triage/` (a `SKILL.md` **router** plus `references/*.md`, read on demand with `skill_read_file`) → `data/skills/gitlab-issue-triage/` at boot. The **pipeline driver** (model-followed guidance — design §3.8): read the issue via MCP → classify (labels first, inference fallback) → idempotency/resume (existing status label ⇒ resume) → run the matching pipeline applying the transient status label → terminal `Needs Review`. Owns the single-active-status swap and the Q2 "never close/commit" boundary.

`references/notes.md` (R7) owns the **re-entry decision**: establish the agent's own GitLab identity from the MCP token, read the thread, then either resume from the current status label (addressing the newest human note, one reply per dispatch) or — when the newest note is the agent's own — post nothing, change no label, and stop. That last branch is what makes the comment chain self-terminating (ADR-9/ADR-13).

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

**Re-entry** (R7): `Needs Review` is terminal for the agent's own initiative but **resumable** on a new human comment — the agent revises its deliverable, re-posts it, and keeps the label. `Needs Input` resumes the same way. A human still closes the issue.

```
Needs Input  ──human comment──▶ continue from the stage that asked
Needs Review ──human comment──┬ approval → stop (leave Needs Review; they close it)
                              └ feedback → revise, re-post, KEEP Needs Review
any note authored by the AGENT ──▶ no comment, no label, stop   (self-terminating)
```

**Loop prevention** (two-part, ADR-9 as revised by R7):

- **Structural, for label edits.** The skill's relabels come back as `issue`/`merge_request` hooks with `action:"update"`, excluded by the receiver's `action === "open"` requirement. The agent may relabel freely.
- **Agent-side identity, for comments.** The skill's comments come back as `note` hooks and ARE re-dispatched — that is the R7 trigger. The agent's note and a human's are the same shape on the wire, and BOS has no credential with which to tell them apart, so the **agent** determines its own GitLab identity from the MCP token and no-ops when the newest note is its own. Cost: one cheap no-op dispatch per comment the agent posts, terminating there because a no-op writes nothing.

## Relationships

```
GitLab integration (authless)
└─ service "webhooks"
   ├─ config.webhook (WebhookConfig) ── verified against ── WebhookSecrets (SecretsStore)
   ├─ config.triageAgentId ── selects ── Agent (data/agents/<id>)
   └─ inbound webhook ── emits ── BOS event (gitlab.issue.opened / gitlab.issue.noted / gitlab.mr.opened)
                                        │
Triage Handler (core, gitlab-issue-triage) ── consumes gitlab.issue.* ──▶ runSubAgent(triageAgentId)
                                               (opened → fresh task, noted → resume task)
                                                                                   │
                                                                              Agent + skill
                                                                                   │  (via the agent's MCP allowlist)
                                                                              GitLab MCP server (external, user-configured)
                                                                                   │
                                                                              GitLab REST (read issue / comment / label)
```
