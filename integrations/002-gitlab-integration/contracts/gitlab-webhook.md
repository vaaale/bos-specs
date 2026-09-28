# Contract: Inbound GitLab Webhook

The GitLab webhook receiver BOS exposes. Source: [`src/lib/integrations/services/gitlab/adapters/webhook.ts`](../../../src/lib/integrations/services/gitlab/adapters/webhook.ts) (new) + the existing receiver route.

## Endpoint

```
POST /api/integrations/webhooks/gitlab/webhooks
```

Routed by the generic receiver (`src/app/api/integrations/webhooks/[integrationId]/[serviceId]/route.ts`), `integrationId=gitlab`, `serviceId=webhooks`.

## Request

| Header | Value | Required |
|---|---|---|
| `Content-Type` | `application/json` | yes |
| `X-Gitlab-Token` | the shared secret token (the value pasted into GitLab's "Secret token") | yes (once a token is generated) |
| `X-Gitlab-Event` | e.g. `Issue Hook`, `Merge Request Hook` | optional (GitLab sends it; the receiver keys off the body's `object_kind`, not this header) |

Body: GitLab's webhook JSON for the event. The receiver reads the **raw body once** (string) for verification + idempotency + parsing.

## Authentication

**Plain string match** on `X-Gitlab-Token` (GitLab's scheme — *not* HMAC). The handler compares the header, timing-safe, against `[secrets.previous, secrets.primary]`. **No candidates present ⇒ reject** (a gitlab webhook with no token yet 401s — never "accept all").

## Response matrix

| Condition | Status | Body |
|---|---|---|
| No handler registered | `404` | `{ error: { code: "no_handler" } }` |
| Webhook disabled | `404` | `{ error: { code: "not_enabled" } }` — **404, not 401** (no existence/state leak) |
| Integration not connected *(n/a for gitlab — `authless` coerces connected=true, ADR-1)* | `404` | `{ error: { code: "not_connected" } }` |
| Token missing/mismatch | `401` | `{ error: { code: "unauthorized" } }` |
| Duplicate delivery (dedup ring hit) | `200` | `{ ok: true, dedup: true, emitted: 0 }` |
| New issue, `action=open` | `200` | `{ ok: true, emitted: 1 }` |
| New MR, `action=open` | `200` | `{ ok: true, emitted: 1 }` |
| Other `object_kind` (push, pipeline, …) or non-`open` action | `200` | `{ ok: true, emitted: 0 }` |
| Handler threw | `500` | `{ error: { code: "handler_failed" } }` |

## Filtering (`receive`)

`object_kind === "issue"` **and** `object_attributes.action === "open"` → emit `issue-opened`.
`object_kind === "merge_request"` **and** `action === "open"` → emit `mr-opened`.
**Anything else** → `{ events: [] }` (2xx, `emitted: 0`). This filter is also the **loop-prevention** mechanism (the agent's own `note`/`update` webhooks are excluded — ADR-9).

## Payload projection (issued as the IntegrationEvent `data`)

See [data-model.md §3](../data-model.md#3-gitlab-event). The handler reads GitLab's documented fields defensively (tolerant access, defaults); the stable contract is `object_kind` / `action` / `open` + the projection fields.

## GitLab-side registration (manual, v1)

The user registers the endpoint in GitLab: **Project → Settings → Webhooks**, setting:
- **URL** = the receiver URL shown in the panel.
- **Secret token** = the token generated in the panel.
- Triggered on **Issues** (and **Merge requests**).

Automatic per-project `POST /projects/:id/hooks` registration is **out of scope** (Assumption).
