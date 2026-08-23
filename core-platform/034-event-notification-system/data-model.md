# Data Model — Event & Notification System

**Feature**: `034-event-notification-system` · **Phase**: 1 (plan) · **Companion to**: `design.md` §3.4 (storage layout), `contracts/event-api.md` (API)

Entities are defined once here; `design.md` keeps the storage layout, this file keeps the fields, validation, and state transitions.

---

## 1. EventRecord (immutable — stored in per-month JSONL shards)

| Field | Type | Validation / Notes |
|---|---|---|
| `id` | string | Unique. Kernel-generated (`evt_` + ULID/timestamp+random). Legacy-migrated events: `legacy-<originalIndex>` (stable across re-runs, R6). |
| `type` | string | Dot-separated namespace, **≤256 chars** (NFR-005). Must match `^[a-z0-9]+(\.[a-z0-9_-]+)+$`. |
| `payload` | object (JSON) | **≤1MB serialized** (FR-024/NFR). Arbitrary structure; optional `summary` string field drives the list summary. |
| `source` | `{ appId, name, icon? }` | `appId` = emitter's component id; `name`/`icon` snapshot at emit time (so uninstalled sources still render, spec edge case). |
| `ts` | number | Epoch ms, set by kernel at durable record. |
| `sequence` | number | **Per-type** monotonic (R7); allocated under the store mutex at emit (FR-003). |
| `summary` | string | Derived at emit: `payload.summary` if a string, else truncated payload stringification (≤140 chars). |

**Immutability**: never rewritten. The shard line is the source of truth for identity + payload; mutable facts live in EventState.

## 2. EventState (mutable — stored in per-month state file; warm in memory)

| Field | Type | Notes |
|---|---|---|
| `eventId` | string | FK → EventRecord.id |
| `processing` | `pending` \| `processed` | The pub/sub axis. See state machine §4. |
| `processedReason?` | `all-acked` \| `no-active-handlers` \| `all-settled-with-failures` | Present when `processed`. Inspectable in the viewer history view (R8). |
| `read` | `unread` \| `read` | The user axis. Orthogonal to `processing`. |
| `history` | HandlerAcknowledgment[] | Append-only within a handler's retry window; one final entry per (event, handler) pair. |
| `perHandler` | `{ [handlerId]: { status, attempts, lastError? } }` | Working set used to evaluate completion. |

## 3. HandlerAcknowledgment

| Field | Type | Notes |
|---|---|---|
| `eventId` | string | |
| `handlerId` | string | FK → HandlerRegistration.handlerId |
| `appId` | string | Owner component id (ack-ownership, FR-022). |
| `attempt` | number | 1..3 (then permanent failure). |
| `ts` | number | Epoch ms. |
| `status` | `acked` \| `failed` \| `permanently_failed` | `failed` = a retryable attempt; `permanently_failed` = terminal for this (event, handler). |
| `result?` | object (JSON) | Present on `acked` — the handler's result payload (FR-005/006). Stored on the event, viewable in the UI. |
| `error?` | string | Present on `failed`/`permanently_failed`. |

**Exactly-once-settle invariant** (design §7.1): the kernel settles at most one terminal record per (event, handler) — a `callId`/pair key guards against a late ack arriving after a timeout permanently-failing the same attempt.

## 4. HandlerRegistration

| Field | Type | Notes |
|---|---|---|
| `handlerId` | string | Owner-provided, unique within owner. |
| `eventType` | string | Exact type or prefix namespace. |
| `mode` | `headless` \| `ui` | |
| `ownerId` | string | App/service id (or `core` for core-internal). |
| `displayName` | string | Shown in selection dialog + config. |
| `description?` | string | |
| `icon?` | string | |
| `enabled` | boolean | User-controllable, **headless only** (FR-018/019). Default true. |
| `timeoutMs` | number | Headless invocation timeout. Default **30000** (NFR-006), per-app configurable. |
| `declaredBy` | `service` \| `manifest` \| `core` | Headless: runtime-declared (service) or core-internal. UI: manifest-declared. |
| `launch?` | `{ appId, componentHint? }` | UI mode only: which app window to `launch` with event params (R4). |

**Validation at register (FR-023)** — accepted iff `eventType` prefix-equals/extends **`com.bos.<ownerId>.*`** (owned root from the component id, no declaration) **or** matches a prefix in the owner's static **`eventNamespaces`** grant list (manifest `eventNamespaces?: string[]` / `service.json`). Otherwise rejected with a clear error. Grants are static (declared in the manifest), never self-granted at runtime.

**Active set** (what completion and dispatch consider): `enabled === true` ∧ owner running (service alive) — core handlers always running. Disabled/offline handlers are **not active** and never block completion (FR-008/019/edge cases).

## 5. HandlerPreference

| Field | Type | Notes |
|---|---|---|
| `eventType` | string | |
| `preferredHandlerId` | string | FK → a UI-mode HandlerRegistration for that type. |
| `ts` | number | Last-set time. |

UI-only. Persisted in `preferences.json`; survives sessions (FR-010/027). Does **not** affect processing.

## 6. Stream event (transient — NDJSON wire format)

`{ streamSeq, kind: "new" | "processing" | "read" | "handlers", eventId?, ... }` — replay-then-tail via `?since` (design §3.7). Distinct from per-type `sequence`.

---

## State transitions

### 6.1 Processing axis (`processing`)

```
                 emit
                  │
     active handlers? ── no ──▶ PROCESSED (reason: no-active-handlers)   [immediate]
                  │ yes
                  ▼
              PENDING ──dispatch──▶ (per handler: attempts 1..3)
                  ▲                     │
                  │                     ├─ ack ───────────────────────────────┐
                  │                     ├─ fail (attempt<3) → retry 1s/5s/30s │
                  │                     └─ fail (attempt=3) → permanently_failed
                  │                                                                     │
                  └──────────────────────── all active handlers settled ◀───────────────┘
                                               (acked ∪ permanently_failed)
                                                          │
                                                          ▼
                                              PROCESSED (reason: all-acked
                                                       or all-settled-with-failures)
```

- Transition to `processed` is **one-way** (no re-pending).
- On boot / late (re)registration: un-acked PENDING events are **re-dispatched** (at-least-once, FR-005) — state unchanged, dispatch retried.
- **Uninstall/disable** of the last active handler for a PENDING event → re-evaluate: no active handlers left ⇒ PROCESSED (reason: no-active-handlers) (FR-020).

### 6.2 Read axis (`read`)

```
UNREAD ──user clicks event in viewer / mark-read / mark-all──▶ READ   (one-way)
```

- Orthogonal to processing: an event can be `read`+`pending` (viewed while still processing — badge updates live, FR-028) or `unread`+`processed`.
- Bell = count of `read === unread` (ADR-6). Default viewer list = unread; "Show historical" = read.

### 6.3 Invariants

1. **Bodies are source of truth**; index + month-state are derived projections (repair from shards at boot; at-least-once covers lost sub-checkpoint state — R3).
2. **At most one terminal HandlerAcknowledgment per (event, handler)** (R3-exactly-once guard).
3. **`sequence` is per-type monotonic, gap-free at emit time**; handlers may observe gaps only if a prior event was never durable (impossible) — gaps indicate a type-level emit failure, not loss.
4. **Processing completion is evaluated only when the active set changes or an ack arrives** — never on read/click (FR-014).
5. **`processedReason` is set exactly once**, at the PENDING→PROCESSED transition.
