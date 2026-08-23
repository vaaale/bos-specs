# Contract — Public Event API

**Feature**: `034-event-notification-system` · **Phase**: 1 (plan) · **Companion to**: `data-model.md`

**One contract, three transports** (ADR-2): the operations below are a single API implemented once in `src/lib/events/api.ts`. It is reached by:

1. **In-process** — agent tools (`serverTool`, no HTTP hop) and kernel-internal calls.
2. **Same-origin HTTP** — Event Viewer + bell (the browser).
3. **Loopback HTTP** — worker-thread services (`http://127.0.0.1:$PORT`, R1/R2).

The same function signatures back all three; the HTTP routes are thin `fetch` adapters that delegate to `api.ts`. The agent-tool surface is **the same contract** (FR-001) — tools wrap the identical ops.

> All request/response bodies are JSON. Errors use a uniform envelope:
> `{ "error": { "code": string, "message": string } }` with a matching HTTP status.

---

## 1. `emit`

Publish an event. Durably records before returning (NFR-001). Does **not** block on handler completion.

**POST** `/api/events`  (or in-process `api.emit`)

Request:
```json
{ "type": "com.bos.gsuite.email.received",
  "payload": { "…": "…", "summary": "Invoice #4812 from Acme Corp" },
  "source": { "appId": "gsuite", "name": "GSuite" } }
```
- `type`: required, ≤256 chars, dot-separated (NFR-005).
- `payload`: required object; **≤1MB** serialized else `413 payload-too-large` (FR-024).
- `source`: required; the emitter identifies itself (used for the snapshot name/icon).

Response `201`:
```json
{ "id": "evt_01J…", "sequence": 4821, "ts": 1756000000000,
  "processing": "pending", "read": "unread",
  "activeHandlers": 1 }
```
Errors: `400 invalid-type` · `413 payload-too-large`.

**Semantics**: sequence allocated per-type (R7); dispatched to active headless handlers asynchronously; if no active handlers → `processing: "processed"`, `activeHandlers: 0` (R8).

---

## 2. `query`

List events. Powers the Event Viewer list and the historical toggle.

**GET** `/api/events?type=…&status=…&read=…&from=…&to=…&cursor=…&limit=…`

Query params (all optional):
| Param | Values | Notes |
|---|---|---|
| `type` | string / prefix | Filter by type or namespace prefix. |
| `status` | `pending` \| `processed` | Processing axis. |
| `read` | `unread` \| `read` | Read axis. Viewer default = `read=unread`; historical = `read=read`. |
| `from` / `to` | epoch ms | Time range on `ts`. |
| `cursor` | string | Opaque pagination cursor (offset-backed). |
| `limit` | int, default 50, max 200 | Page size. |

Response `200` (list of **summaries**, not full payloads — FR: lazy load):
```json
{ "events": [ { "id":"evt_…", "type":"…", "sequence":4821, "ts":1756…,
                "source":{ "appId":"gsuite","name":"GSuite" },
                "summary":"Invoice #4812…", "processing":"processed", "read":"unread",
                "handlersTotal":1, "handlersDone":1 } ],
  "nextCursor": null, "unreadTotal": 4 }
```
- `handlersTotal`/`handlersDone` drive the "2/3 handlers processed" status (US1).
- `unreadTotal` is the bell count (redundant with `count` for convenience).

---

## 3. `get` (single event, full)

**GET** `/api/events/:id`

Response `200`:
```json
{ "id":"evt_…", "type":"…", "sequence":4821, "ts":1756…,
  "source":{…}, "summary":"…",
  "payload": { "…full payload…" },
  "processing":"processed", "processedReason":"all-acked", "read":"read",
  "history": [ { "handlerId":"…","appId":"gsuite","name":"GSuite Sync (headless)",
                 "icon":"mail","attempt":1,"ts":1756…,"status":"acked",
                 "result":{ "stored":"mail://inbox/9921" } } ] }
```
- `history` = full HandlerAcknowledgment list (data-model §3) — the processing-history view.
- Errors: `404 not-found`.

---

## 4. `ack`

A headless handler acknowledges an event it processed, with an optional result payload.

**POST** `/api/events/:id/ack`

Request:
```json
{ "handlerId": "gsuite-sync",
  "result": { "stored": "mail://inbox/9921", "indexed": true } }
```
- `handlerId`: required — the acking handler's id.
- `result`: optional object; stored on the event (FR-005/006).
- **Ownership (FR-022)**: the caller's component id must own `handlerId`. Services pass their identity via the headless-auth header (same mechanism as `/api/fs` loopback); in-process callers pass their id explicitly. Violation → `403 ack-forbidden`.

Response `200`:
```json
{ "settled": true, "processing": "processed", "attempts": 1 }
```
- `processing` reflects the event state *after* this ack (may have just completed).
- Errors: `403 ack-forbidden` · `404 not-found` · `409 already-settled` (a terminal ack for this (event, handler) already exists — exactly-once guard; a duplicate idempotent ack returns `200` with `attempts` unchanged, `409` is for conflicting state).

**Idempotency**: handlers may receive the same event more than once (at-least-once) and must handle repeat acks gracefully; the kernel ignores a second ack for the same (event, handler, attempt) with no state change.

---

## 5. `markRead`

**POST** `/api/events/:id/read`   (single)
**POST** `/api/events/read?all=1` (mark all unread → read)

Response `200`: `{ "read": "read", "unreadTotal": 3 }` / `{ "marked": 12, "unreadTotal": 0 }`.

One-way (UNREAD→READ, data-model §6.2). Does **not** affect processing.

---

## 6. `register` / `unregister`

Declare a handler. Headless handlers are runtime-declared by services (over worker IPC internally, but the same op) or by core; UI handlers are declared in the app manifest and surfaced at boot.

**POST** `/api/events/register`
```json
{ "handlerId": "gsuite-sync", "eventType": "com.bos.gsuite.email.received",
  "mode": "headless", "ownerId": "gsuite",
  "displayName": "GSuite Sync", "description": "Index incoming mail",
  "timeoutMs": 30000 }
```
UI mode adds: `"launch": { "appId": "gsuite", "componentHint": "mail" }`.

- **Validation (FR-023)**: `eventType` must be within `com.bos.<ownerId>.*` or a granted `eventNamespaces` prefix. Violation → `403 namespace-not-owned`.
- Response `200`: `{ "handlerId":"…", "enabled": true }`.
- Re-registering the same `handlerId` is an upsert (idempotent).

**POST** `/api/events/unregister`
```json
{ "handlerId": "gsuite-sync", "ownerId": "gsuite" }
```
- Owner-checked (`403` on mismatch). Removes the registration; pending events are re-evaluated (data-model §6.1).

---

## 7. `setPreference`

Set/clear the default **UI** handler for a type ("Always use this app").

**POST** `/api/events/preference`
```json
{ "eventType": "com.bos.gsuite.email.received",
  "preferredHandlerId": "gsuite-mail" }
```
- Omit/`null` `preferredHandlerId` → clears the preference.
- `preferredHandlerId` must be a UI-mode registration for that type (`400 invalid-preference` otherwise).
- Response `200`: `{ "eventType":"…", "preferredHandlerId":"gsuite-mail" }`.

Persists across sessions (FR-010/027).

---

## 8. `count`

**GET** `/api/events/count`

Response `200`:
```json
{ "unreadTotal": 4, "pendingTotal": 1, "grandTotal": 412 }
```
- `unreadTotal` drives the bell (ADR-6). Cheap (warm index only).

---

## 9. `listHandlers` / `setEnabled`

Configuration page (FR-018).

**GET** `/api/events/handlers` → grouped registry:
```json
{ "com.bos.gsuite.email.received": {
    "headless": [ { "handlerId":"…","displayName":"GSuite Sync","icon":"mail",
                    "enabled":true, "timeoutMs":30000, "recentFailures":0 } ],
    "ui": [ { "handlerId":"gsuite-mail","displayName":"GSuite Mail","icon":"mail",
              "description":"Open the full email", "isDefault":false } ] } }
```
`recentFailures` = count of `permanently_failed`/`failed` acks in the current month (config failure badge).

**POST** `/api/events/handlers`
```json
{ "handlerId": "…", "ownerId": "…", "enabled": false }
```
- Owner-checked. Toggles `enabled` (headless only; `400` if mode is `ui`). Disabling re-evaluates pending events (data-model §6.1). Response `200`: `{ "handlerId":"…", "enabled": false }`.

---

## 10. `stream` (NDJSON, real-time)

**GET** `/api/events/stream?since=<streamSeq>`

Replay-then-tail NDJSON (design §3.7, FR-028/NFR-009). Each line:
```json
{ "streamSeq": 812, "kind": "new", "eventId": "evt_…", "ts": 1756… }
{ "streamSeq": 813, "kind": "processing", "eventId": "evt_…", "processing": "processed" }
{ "streamSeq": 814, "kind": "read", "eventId": "evt_…", "read": "read" }
{ "streamSeq": 815, "kind": "handlers", "eventType": "…", "handlerId": "…" }
```
- `since=0` (or omitted) → full replay from the start of the in-memory ring (bounded), then live tail. The viewer reconciles a `count`/`query` snapshot first, then tails.
- Long-lived `fetch`/EventSource; heartbeats every 15s to keep the connection alive.

---

## Error code reference

| Code | HTTP | Meaning |
|---|---|---|
| `invalid-type` | 400 | Type fails namespace/length validation. |
| `payload-too-large` | 413 | Payload > 1MB. Use a VFS path reference. |
| `namespace-not-owned` | 403 | `register` for a type outside the owner's root/grants. |
| `ack-forbidden` | 403 | Caller does not own the `handlerId`. |
| `already-settled` | 409 | Conflicting terminal ack for (event, handler) — e.g. an ack arriving for an attempt that was already permanently failed. |
| `invalid-preference` | 400 | `preferredHandlerId` isn't a UI handler for the type. |
| `not-found` | 404 | Unknown event/handler id. |

## Agent-tool mapping (FR-001)

The assistant gets these as `serverTool`s (in-process, transport 1): `emit_event`, `query_events`, `get_event`, `ack_event` (the agent acks only handlers it owns, per FR-022 ownership), `mark_events_read`, `set_event_preference`, `list_event_handlers`. Tool schemas mirror the request/response shapes above 1:1 so the agent-facing and app-facing contracts are provably the same.
