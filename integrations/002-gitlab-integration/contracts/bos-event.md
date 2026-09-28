# Contract: Emitted BOS Event

The events the GitLab integration emits into the BOS event kernel (`src/lib/events`). These are what handlers subscribe to — the unit of decoupling between the webhook and any consumer (the default triage dispatcher, or the Workflow app, or any future consumer).

## Types

| Event type | Triggered by | Handler target? |
|---|---|---|
| `gitlab.issue.opened` | A new (action `open`) issue in a hooked project | **Yes** — `gitlab-issue-triage` is registered for this exact type. |
| `gitlab.mr.opened` | A new (action `open`) merge request | **No** — emitted for user routing; the default handler does not consume it (Q1=A). |

Both pass `isValidEventType` (dot-separated, lowercase). Emission is via the existing `emitIntegrationEvent` → `api.emit` path (the receiver route already calls it for each event `receive` returns); the type comes from the `legacy-mapping` `KNOWN_TYPES` entries (ADR-2) — **no new code-logic branch**.

## Shape

`api.emit({ type, payload, source })`:

| Field | Value |
|---|---|
| `type` | `gitlab.issue.opened` or `gitlab.mr.opened` |
| `source` | `{ appId: "gitlab", name: "GitLab" }` (from `KNOWN_NAMES.gitlab`) |
| `payload` | `{ ...data, _integrationType, _integrationService, summary }` where `data` is the projection below |
| `payload.summary` | `"issue #<iid> <title>"` / `"MR !<iid> <title>"` (the `legacyFriendlySummary` branch) |

## `payload` fields

| Field | Type | Present in | Notes |
|---|---|---|---|
| `projectId` | `number` | both | GitLab numeric project id (`project.id`) — use for all downstream API/MCP calls. |
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

## Consumer contract (what a handler gets)

A handler subscribed to `gitlab.issue.opened` (e.g. `gitlab-issue-triage`) receives the event record on dispatch: `id`, `type`, `source`, `payload` (the fields above), `timestamp`, and processing fields. It must **prompt-ack** (return/complete its dispatch entry quickly) and do long work asynchronously (the kernel's 30 s handler timeout applies to the ack, not the work — ADR-3). The default dispatcher forwards `payload` (as the issue context) to the configured agent — see [dispatcher-task.md](./dispatcher-task.md).

## Multi-handler semantics

Per the event kernel, **multiple enabled handlers can subscribe to the same type** (e.g. `gitlab.issue.opened` → the default `gitlab-issue-triage` *and* a Workflow-app handler). Each acts independently; the webhook and each handler are separate switches (spec US3, FR-018).

## Size

The projection is bounded (description truncated) so `payload` stays within `MAX_PAYLOAD_BYTES` (1 MB). No VFS path reference needed in practice.
