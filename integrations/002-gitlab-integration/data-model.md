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
| `issueLabels` | `string[]` | **`issue.labels[].title`** (top level, R8) | The issue's **current** labels — the **gate signal**. Verified present in the note payload (the note hook carries the issue's own `labels[]` on the top-level `issue` object — `note_builder.rb`/`data_builder/note.rb`); `object_attributes` describes the *comment*. Lets the dispatcher see a human-waiting state and seed the per-issue conversation (the dispatcher reads the *gate*, not the *intent* — FR-027). |
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
| `skills` | `["gitlab-issue-triage", "spec-kit-driver", "ui-designer-craft"]` | R8: the triage skill (pins the state machine) **+ the spec-kit method driver + the UI-designer craft** — the agent runs the spec pipeline itself (R8), so it loads the same method skills the Build Studio agent does. |
| `mcp` | `[]` | Empty = all servers (robust to the user's server naming; user may narrow). The GitLab MCP server must be among them to act. |
| `tools` | **All-server pipeline gate (R8)**: `mcp_*`, `skill_*` (incl. `skill_read_file`), `web_search`/`web_fetch`, `scratchpad_*`, `memory_*`, `file_*` (spec-store writes on the branch), `gitlab_start_feature` (the new server tool), `dev_delegate` (approval-2 implementation) | R8 widens the gate from "MCP + text/skill" to the full server-tool set so the agent can write real artifacts to the branch and delegate implementation. The gate is the `baseGate.allow` filtered to `execution: "server"` (survives `headlessGate`) — **no promote/merge tool exists**, so the boundary holds by construction (FR-028). |
| system prompt | "You are the Gitlab Issue Agent… use the `gitlab-issue-triage` skill…" | |

## 6. Triage Skill (`SKILL.md`)

`seed/skills/gitlab-issue-triage/` (a `SKILL.md` **router** plus `references/*.md`, read on demand with `skill_read_file`) → `data/skills/gitlab-issue-triage/` at boot. The **pipeline driver** (model-followed guidance — design §3.8): read the issue via MCP → classify (labels first, inference fallback) → idempotency/resume (existing status label ⇒ resume) → run the matching pipeline applying the transient status label → terminal `Needs Review`. Owns the single-active-status swap and the Q2 "never close/commit" boundary.

`references/notes.md` (R7) owns the **re-entry decision**: establish the agent's own GitLab identity from the MCP token, read the thread, then either resume from the current status label (addressing the newest human note, one reply per dispatch) or — when the newest note is the agent's own — post nothing, change no label, and stop. That last branch is what makes the comment chain self-terminating (ADR-9/ADR-13). **R8** adds two references — `references/approve-spec.md` (approval-1: judge approval → `gitlab_start_feature` → run the method headless to write `spec.md` [+ `mockup.html` via `file_write` + `design.md` for UI features] → FR-031 attach → `Design Complete`) and `references/approve-design.md` (approval-2: reuse the bound branch → `plan.md` → `tasks.md` → `dev_delegate` (TDD, same branch) → FR-031 attach → `Needs Review`) — and extends `notes.md` (approval-vs-feedback, FR-027), `status-labels.md` (R8 work stages + gate reclassifications), and `finalize.md` (FR-031 attachment; `Needs Review` = implementation complete).

## 7. Labels (`labels.ts` — shared constants)

**Classification labels** (additive, persistent — one determines the workflow):

| Label | Workflow |
|---|---|
| `Bug` | analyze → verify → propose fix |
| `Feature` | draft a spec for a new feature |
| `Improvement` | draft a spec to improve an existing feature |

**Status labels** (transient, exactly one active — the pipeline stage):

R8 reclassifies the terminal state and adds the spec-pipeline work stages (FR-014). **Work stages** advance autonomously; **human-waiting gates** are where a GitLab comment is read as approval-or-feedback (FR-027).

| Label | Kind | Meaning | Path |
|---|---|---|---|
| `Triage` | work stage | received, being classified | all |
| `Analyzing` | work stage | analyzing the report | Bug |
| `Verifying` | work stage | assessing plausibility / root cause | Bug |
| `Mockup` | work stage | (R8) producing the UI mockup on the branch (a `file_write` of `mockup.html`) | Feature, Improvement (UI surface) |
| `Design` | work stage | (R8) producing the technical `design.md` on the branch | Feature, Improvement |
| `Plan` | work stage | (R8) writing `plan.md` on the branch | all (approval-2) |
| `Tasks` | work stage | (R8) writing `tasks.md` on the branch | all (approval-2) |
| `Implementing` | work stage | (R8) `dev_delegate` running the implementation on the branch | all (approval-2) |
| `Fix Proposed` | **gate (approval 1)** | (R8) Bug deliverable posted in a comment — awaiting the user's approval to branch + write the real spec | Bug |
| `Spec Drafted` | **gate (approval 1)** | (R8) Feature/Improvement deliverable posted in a comment — parallel to `Fix Proposed`; awaiting approval | Feature, Improvement |
| `Design Complete` | **gate (approval 2)** | (R8) real `spec.md` (+ `mockup.html` + `design.md`) on the branch, attached to the issue (FR-031) — awaiting approval to plan → tasks → implement | all |
| `Needs Review` | **gate (promotion)** | (R8) **implementation complete, preview healthy** — the user promotes (human-only, always). Refined from R5's "agent finished its deliverable." | all |
| `Needs Input` | side | blocked — the agent needs more info from the reporter | any |
| `Failed` | side | the run errored; a failure note was posted | any |

**State transitions** (R8, the full two-gate flow — Bug path, the richest):

```
(open, no status) ──classify──▶ Triage
  Triage ─▶ Analyzing ─▶ Verifying ─▶ Fix Proposed   (GATE 1)
                          │  user comment: approval (gitlab_start_feature → real spec.md) →  …
  Fix Proposed ─approval─▶ (branch) ─▶ [Feature/Improvement: Mockup ─▶ Design] ─▶ Design Complete   (GATE 2)
  Design Complete ─approval─▶ Plan ─▶ Tasks ─▶ Implementing ─▶ Needs Review   (GATE 3: promote)
  any work stage ─(blocked)─▶ Needs Input        any ─(error)─▶ Failed
  (feedback at any gate: revise + re-post, STAY at that gate, NO branch on feedback)
```

Feature/Improvement approval-1 path: `Triage ─▶ (work) ─▶ Spec Drafted (GATE 1) ─approval─▶ (branch) ─▶ Mockup ─▶ Design ─▶ Design Complete (GATE 2) …`.

**Re-entry** (R7, R8): a human comment on a **gate** (`Spec Drafted` / `Fix Proposed` / `Design Complete` / `Needs Review`) is disambiguated by the agent (FR-027) — **approval** advances the pipeline (approval-1 creates the branch + writes the real artifacts → `Design Complete`; approval-2 runs plan → tasks → implement → `Needs Review`); **feedback** revises the deliverable, re-posts it, and **stays** in the same gate (no branch on feedback). `Needs Review` is now *implementation complete, awaiting promotion* — a human comment there is promotion-feedback (revise) or the user promotes from the top bar (the agent never promotes). `Needs Input` resumes the same way. A human still closes the issue.

```
Spec Drafted / Fix Proposed ──human comment──┬ approval → gitlab_start_feature → real artifacts → Design Complete
                                             └ feedback → revise draft, re-post, STAY (no branch)
Design Complete ──human comment──────────────┬ approval → plan → tasks → dev_delegate → Needs Review
                                             └ feedback → revise design, re-post, STAY
Needs Review ──human comment────────────────┬ promotion (top bar) → base            (agent never promotes)
                                             └ feedback → revise implementation, re-post, STAY
Needs Input  ──human comment──▶ continue from the stage that asked
any note authored by the AGENT ──▶ no comment, no label, stop   (self-terminating)
```

**Loop prevention** (two-part, ADR-9 as revised by R7):

- **Structural, for label edits.** The skill's relabels come back as `issue`/`merge_request` hooks with `action:"update"`, excluded by the receiver's `action === "open"` requirement. The agent may relabel freely.
- **Agent-side identity, for comments.** The skill's comments come back as `note` hooks and ARE re-dispatched — that is the R7 trigger. The agent's note and a human's are the same shape on the wire, and BOS has no credential with which to tell them apart, so the **agent** determines its own GitLab identity from the MCP token and no-ops when the newest note is its own. Cost: one cheap no-op dispatch per comment the agent posts, terminating there because a no-op writes nothing.

## 8. The per-issue ghost conversation (R8)

A **stable, per-issue** conversation the dispatcher seeds on every `gitlab.issue.noted` so the agent can run **conversation-bound** and bind a feature branch **mid-run** (design ADR-14).

- **Id**: `c-gitlab-issue-<projectId>-<iid>` — derived from the note payload, **stable across every comment on the issue** (the same id on re-dispatch, so an approval's branch + artifacts persist across the issue's later comments). This is the R8 idempotency key.
- **Seeded by**: `conversationStore.saveConversationMessages(conversationId, agentId, [task])` on each noted dispatch — the **ghost seed**, idempotent (mirrors `self-heal/intake.ts:515`, keyed on the issue instead of a case id). Import-free reuse — self-heal is **not** modified.
- **Content**: a **ghost** — its transcript is ignored by `runLocalHeadless` (`io.loadMessages` returns `[]`); it exists only to (i) **carry the branch** (via `setConversationActiveFeatureBranch`) and (ii) be the **FR-030 visibility home** (the conversation on the branch). It is **not** a user Assistant conversation (Assumption A; the trajectory is the headless run transcript, not attached).
- **Relationship to the run + branch**: the run is bound to this conversation by `runSubAgent(agent, task, { conversationId })`. The branch (when an approval has bound one) is read **fresh per tool call** from this conversation (`getConversationActiveFeatureBranch` is an uncached read), so a mid-run bind is picked up by the agent's next `file_*` write / `dev_delegate` (the linchpin).
- **Lifespan**: bounded by the number of **distinct issues** processed (not notes — the id is per-issue), inert, no GC in v1 (Risk 13). Disabling the handler does not delete them.

## 9. The per-issue feature branch (R8)

The ordinary BOS feature branch the agent creates on approval, carrying the real spec-kit artifacts (FR-023/024/025/026).

- **Name**: `bos/<NNN>-<slug>` — **ordinary** naming (NNN = next feature number in the chosen `user-specs` project, `bos/` prefix), **never** self-heal naming (FR-023). Computed server-side by `gitlab_start_feature` ([contracts/gitlab-start-feature.md](./contracts/gitlab-start-feature.md)).
- **Bound mid-run**: `setConversationActiveFeatureBranch(ctx.conversationId, branch)` — bound to the ghost conversation **while the agent is already running on it**; picked up by the next spec write / `dev_delegate` because the branch is read fresh per call (§8 linchpin).
- **Git ref is lazy**: `gitlab_start_feature` does **not** create a git branch — the ref is created lazily by the first `dev_delegate` (the same mechanism self-heal relies on). A **feedback** comment never calls the tool, so the ref (and the feature dir) never materialize — no number burned, no empty branch.
- **Contents**: `spec.md` (approval-1); + `mockup.html` (a `file_write`, **not** the live UI-Preview surface — ADR-16) + `design.md` (approval-1, UI-surface features); `plan.md` + `tasks.md` (approval-2); the Developer's source edits on `dev_delegate`'s worktree of this branch (approval-2). Written to `/Specs/user-specs/<project>/<NNN>-<slug>/` via `withFeatureScope({ conversationId })`.
- **Idempotency**: re-approval of the same issue reuses the branch already bound to the ghost conversation (the tool does not allocate a second NNN). Write-time re-check-and-bump handles the concurrent-approval collision (Risk 11).
- **FR-030**: the branch is just a branch — Build Studio's spec tree + artifact viewer already read from the branch-coupled store root, so the artifacts are visible with zero integration-specific code.

## 10. The FR-031 issue artifact post (R8)

Once approved and the agent has written the **real** artifacts on the branch, it makes them visible on the **GitLab issue** (FR-031) — the human-facing view (the branch artifacts remain the source of truth, FR-030).

- **Form**: a **file attachment** where the GitLab MCP server exposes an upload/attachment capability, otherwise a **comment** carrying the artifact. The mechanism is **MCP-driven** — the skill discovers the server's tool names (same as the comment/label tools), so the integration code holds no attachment logic and no GitLab identity (R4).
- **When**: at `Design Complete` (post `spec.md`/`mockup.html`/`design.md`) and at `Needs Review` (post `plan.md`/`tasks.md`).
- **Names**: the post carries the **branch name** + **spec path** (`/Specs/user-specs/<project>/<NNN>-<slug>`), so a human reviewing in GitLab can open the branch in Build Studio.
- **vs the pre-approval draft**: the *draft* deliverable (the fix / spec the agent posts in a comment before approval-1) stays a plain comment (FR-012); FR-031 covers the **approved, real** artifacts only.

## Relationships

```
GitLab integration (authless)
└─ service "webhooks"
   ├─ config.webhook (WebhookConfig) ── verified against ── WebhookSecrets (SecretsStore)
   ├─ config.triageAgentId ── selects ── Agent (data/agents/<id>)
   └─ inbound webhook ── emits ── BOS event (gitlab.issue.opened / gitlab.issue.noted / gitlab.mr.opened)
                                        │
Triage Handler (core, gitlab-issue-triage) ── consumes gitlab.issue.* ──▶ runSubAgent(triageAgentId)
   (opened → fresh task, headless)            (noted → resume task, CONVERSATION-BOUND, R8)
                                                                                        │
                                                                              per-issue ghost conversation  (c-gitlab-issue-<projectId>-<iid>)
                                                                                        │
                                                                              Agent + skill (all-server gate, R8)
                                                                                        │
                          ┌── MCP (read issue / comment / label / attach artifact) ─────┤
                          ├── gitlab_start_feature ── binds ──▶ feature branch (bos/<NNN>-<slug>, ordinary)  (R8, mid-run)
                          ├── file_* (spec.md / mockup.html / design.md / plan.md / tasks.md) ── write to ──▶ branch  (R8)
                          └── dev_delegate (approval-2, TDD) ── implements on ──▶ branch worktree  (R8)
                                                                                        │
                                                                              Build Studio (reads the branch — FR-030)  │
                                                                              GitLab issue (FR-031 artifact post)        ▼
                                                                              GitLab MCP server (external) ──▶ GitLab REST
```
