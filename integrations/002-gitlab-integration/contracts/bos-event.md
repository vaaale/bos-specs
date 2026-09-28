# Contract: Emitted BOS Event

The events the GitLab integration emits into the BOS event kernel (`src/lib/events`). These are what handlers subscribe to — the unit of decoupling between the webhook and any consumer (the default triage dispatcher, or the Workflow app, or any future consumer).

## Types

| Event type | Triggered by | Handler target? |
|---|---|---|
| `gitlab.issue.opened` | A new (action `open`) issue in a hooked project | **Yes** — via the `gitlab-issue-triage` handler's `gitlab.issue.*` subscription. Builds the FRESH task. |
| `gitlab.issue.noted` | A new comment on an issue (`object_kind: "note"`, `noteable_type: "Issue"`) — R7 / FR-022 | **Yes** — same handler, same `gitlab.issue.*` subscription. Builds the RESUME task. |
| `gitlab.mr.opened` | A new (action `open`) merge request | **No** — emitted for user routing; the default handler does not consume it (Q1=A), and the prefix stops at `issue` so it cannot. |

All three pass `isValidEventType` (dot-separated, lowercase). Emission is via the existing `emitIntegrationEvent` → `api.emit` path (the receiver route already calls it for each event `receive` returns); the type comes from the `legacy-mapping` `KNOWN_TYPES` entries (ADR-2) — **no new code-logic branch**.

> The two `gitlab.issue.*` names are a **pinned pair**, not two independent choices. The handler subscribes to the prefix, and the systematic derivation would have produced `com.bos.gitlab.webhooks.issue-noted` — outside the prefix, and therefore never dispatched. A future GitLab event that should re-enter triage must be named inside `gitlab.issue.*`; one that should not, must not.

## Shape

`api.emit({ type, payload, source })`:

| Field | Value |
|---|---|
| `type` | `gitlab.issue.opened`, `gitlab.issue.noted` or `gitlab.mr.opened` |
| `source` | `{ appId: "gitlab", name: "GitLab" }` (from `KNOWN_NAMES.gitlab`) |
| `payload` | `{ ...data, _integrationType, _integrationService, summary }` where `data` is the projection below |
| `payload.summary` | `"issue #<iid> <title>"` / `"MR !<iid> <title>"` / `"comment on issue #<iid>"` (the `legacyFriendlySummary` branches) |

## `payload` fields — item events (`gitlab.issue.opened`, `gitlab.mr.opened`)

| Field | Type | Present in | Notes |
|---|---|---|---|
| `projectId` | `number` | both | GitLab numeric project id (`project.id`) — use for all downstream API/MCP calls. |
| `projectPath` | `string` | both | `project.path_with_namespace`. |
| `iid` | `number` | both | Project-scoped number (`object_attributes.iid`). |
| `title` | `string` | both | |
| `labels` | `string[]` | both | `object_attributes.labels[].title`. |
| `author` | `{ name, username }` | both | Top-level `user`. |
| `url` | `string` | both | `object_attributes.url`. |
| `description` | `string` | both | Truncated (~2 000 chars). |
| `sourceBranch` | `string` | MR only | `object_attributes.source_branch`. |
| `targetBranch` | `string` | MR only | `object_attributes.target_branch`. |
| `_integrationType` | `string` | both | `issue-opened` / `mr-opened` (the raw IntegrationEvent type). |
| `_integrationService` | `string` | both | `gitlab/webhooks`. |

## `payload` fields — `gitlab.issue.noted` (R7 / FR-022)

Only the issue's coordinates and the comment. Everything else the agent re-reads over MCP, which it must do anyway — the note may not be the newest one by the time the run starts.

| Field | Type | Notes |
|---|---|---|
| `projectId` | `number` | `project.id`. |
| `projectPath` | `string` | `project.path_with_namespace`. |
| `iid` | `number` | The issue's **project-scoped** number — from the **top-level `issue.iid`**. See the warning below. |
| `note` | `string` | `object_attributes.note`, truncated (~2 000 chars). |
| `author` | `{ name, username }` | Top-level `user` — the **comment's** author, not the issue's. |
| `url` | `string` | `object_attributes.url` (the issue URL with the `#note_<id>` anchor). |
| `_integrationType` | `string` | `issue-noted`. |
| `_integrationService` | `string` | `gitlab/webhooks`. |

> **`noteable_iid` does not exist.** A note hook's `object_attributes` describes the comment and carries `noteable_id`, the issue's **global/database** id; GitLab emits no `noteable_iid` (verified against `lib/gitlab/hook_data/note_builder.rb` and the documented payload). The iid comes from the top-level `issue` object. The projection reads `noteable_iid` first and falls back to `issue.iid`, purely for self-hosted tolerance — `issue.iid` is what actually answers.
>
> There is **no `author.isAgent`, and there never will be.** BOS holds no GitLab credential and cannot know which user the agent's MCP token acts as (R1/R4). Distinguishing the agent's own comment from a human's is the **agent's** job, done from the MCP server — see [dispatcher-task.md](./dispatcher-task.md) and design ADR-9/ADR-13.

## Consumer contract (what a handler gets)

A handler subscribed to `gitlab.issue.*` (e.g. `gitlab-issue-triage`) receives the event record on dispatch: `id`, `type`, `source`, `payload` (the fields above), `timestamp`, and processing fields. **It must branch on `type`** — the two issue events carry different payload shapes and mean different things (start vs. resume). It must **prompt-ack** (return/complete its dispatch entry quickly) and do long work asynchronously (the kernel's 30 s handler timeout applies to the ack, not the work — ADR-3). The default dispatcher forwards `payload` to the configured agent in one of two task shapes — see [dispatcher-task.md](./dispatcher-task.md).

## Multi-handler semantics

Per the event kernel, **multiple enabled handlers can subscribe to the same type** (e.g. `gitlab.issue.opened` → the default `gitlab-issue-triage` *and* a Workflow-app handler). Each acts independently; the webhook and each handler are separate switches (spec US3, FR-018).

A handler may register an exact type or a `prefix.*` pattern (`isValidRegistrationType`), and `typeMatches` resolves it per event — which is how one `gitlab-issue-triage` registration covers both issue events. A consumer that wants only new issues registers the exact `gitlab.issue.opened` and is unaffected by notes.

## Size

The projection is bounded (description truncated) so `payload` stays within `MAX_PAYLOAD_BYTES` (1 MB). No VFS path reference needed in practice.
