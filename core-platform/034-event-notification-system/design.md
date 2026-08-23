# Design — Event & Notification System

**Spec:** `034-event-notification-system` · **App Target:** `bos-core` · **Mechanism:** `dev_delegate` (BOS-source feature branch)

A pub/sub event broker that is a **first-class in-process BOS subsystem** (not a worker-thread service, not a Next.js route, not a marketplace item), with one shared API surface exposed over three transports (in-process, same-origin HTTP, loopback HTTP), a built-in **Event Viewer** app, and a repurposed topbar bell. It migrates the existing GSuite/Telegram notification inbox onto this system.

> This document is the design artifact. `plan.md` references it; it does not repeat it.

---

## 1. Classification

**App Target: `bos-core`.** Agrees with `spec.md`'s own `App Target` field.

Rationale (per `references/target-bos-core.md`, `target-builtin-app.md`, `target-marketplace-item.md`):

- The **core event service** is a server-side daemon that starts at boot and runs for the process lifetime. BOS already has exactly this shape, in-process: the **scheduler daemon** (`src/lib/scheduler/daemon.ts` → `engine.ts`, started from `src/instrumentation.ts`'s `register()`). The event kernel is a sibling of that daemon, not a worker-thread service.
- The **Event Viewer** is a `builtin-app` (`src/apps/<id>/`) — a thin window over a BOS subsystem that must ship as part of BOS and be extensible by any app, which is precisely the "prefer built-in" case in `target-builtin-app.md`. It is a *constituent* of this `bos-core` feature, not a separate target.
- This is **not** a `marketplace-item`. The word "service" in the spec names the *core broker*, which does not run as an installable worker-thread daemon on its own port. **Worker-thread services are the *consumers/emitters* of this system, not the system itself.** Reaching for the marketplace-item service facet here would be the documented failure mode (treating "runs in the background" as "must be a service item") — it does not: BOS core already owns in-process daemons.
- No `src/middleware.ts`, no new routing layer. Next.js App Router limitations are irrelevant because the broker is in-process server code; HTTP exposure is a set of thin API routes (the normal, documented "Add an API route" pattern).

The one non-obvious classification decision — *why the broker is an in-process daemon and not a `src/core/service/` worker-thread service* — is ADR-1.

---

## 2. Constitution check

| Principle | Compliance |
|---|---|
| **I. Spec-Driven** | This is the `design` step of an existing agreed `spec.md`. Compliant. |
| **II. Server Authority & SSR Boundary** | All state, persistence, dispatch, and secrets-free domain logic live in server-only modules under `src/lib/events/**` + `src/app/api/events/**`; the Event Viewer and bell communicate over `fetch`/NDJSON. Framework-free shared types in one module (no React/Node). Compliant. |
| **III. Always Delegate; Claude Codes** | Implementation is a `dev_delegate` (Claude) task on a feature branch. Compliant. |
| **IV. Minimize Blast Radius** | All changes on a `bos/*` feature branch; self-modification preview/promote flow. Compliant. |
| **V. VFS Is Not the Source** | Event store persists as **files under `data/events/`** (gitignored runtime state), NOT in the VFS and NOT in `src/`. Compliant — this is the "all runtime state persists as files under `./data`" convention. |
| **VI. Specs & Docs Stay in Sync** | FR-026 requires developer + user docs in the BOS documentation hub; the delegation brief must carry this. Tracked in §4 (docs) and §7. |
| **VII. Respect Boundaries** | **No new external dependency in the default design.** Storage is file-based (`writeFileAtomic`), real-time is the existing NDJSON pattern, virtualization uses a hand-rolled fixed-row window (a `@tanstack/react-virtual` dependency is an *alternative*, flagged in ADR-5, not the default). No `package.json`/lockfile change in the default path. |

**One flag for `plan.md` (Complexity Tracking):** the dispatch engine (fan-out, per-handler FIFO, retry/backoff, at-least-once re-dispatch, the two-axis state machine) is the single most complex piece of this feature. It is justified by FR-004/005/007/008 and is isolated in one module (`src/lib/events/dispatch.ts`) so it can be reasoned about and tested in isolation. If profiling (ADR-5) shows the file-based index rebuild is too slow at the 100k floor, the escape hatch is an embedded DB — a dependency change that would then need explicit approval under VII.

---

## 3. Architecture

### 3.1 Context (C4 — how the user/agent sees it)

```
                        ┌─────────────────────────────────────────────┐
                        │            BrowserOS (the OS)               │
                        │                                             │
  user clicks bell ───▶ │  ┌──────────┐        opens                 │
                        │  │ Topbar   │──────────────────────┐        │
                        │  │ bell     │                      ▼        │
                        │  └────┬─────┘              ┌──────────────┐ │
                        │       │ unread count       │  Event       │ │
                        │       │ (NDJSON/poll)      │  Viewer app  │ │
                        │       └────────────────────▶│  (window)    │ │
                        │                            └──────┬───────┘ │
                        │                                   │click    │
                        │                            ┌──────▼───────┐ │
                        │                            │ launch UI    │ │
                        │                            │ handler app  │ │
                        │                            └──────────────┘ │
                        └─────────────────────────────────────────────┘
   agent / assistant ──▶  (emit / ack / query / register ...)   [same API]
   service (worker)  ──▶  (emit / ack / register ...)           [same API]
```

Three clients, one contract: the **agent** (assistant tools), the **Event Viewer** (browser), and any **service** (worker thread). All perform the same operation set — `emit`, `ack`, `query`, `register`, `unregister`, `mark-read`, `set-preference` — which is exactly what the spec means by "the public event API is the same API that agent tools use." They differ only in transport (ADR-2).

### 3.2 Container (C4 — BOS's real containers)

The event system lives in **one** of BOS's real containers: the **Next.js server process** (Node runtime). It does not introduce a new process.

```
┌────────────────────────── Next.js server process (Node runtime) ──────────────────────────┐
│                                                                                            │
│   src/instrumentation.ts  register()                                                      │
│        │  (boot: after services startAll)                                                 │
│        ▼                                                                                  │
│   ┌──────────────────────────────────────────────────────────────┐                        │
│   │  EVENT KERNEL  (src/lib/events/)  — globalThis singleton     │                        │
│   │  ┌────────┐ ┌───────────────┐ ┌──────────────────────────┐   │                        │
│   │  │ store  │ │ dispatch      │ │ stream                   │   │                        │
│   │  │ (disk) │ │ engine (fan-  │ │ (in-mem NDJSON events)   │   │                        │
│   │  │        │ │ out, FIFO,    │ │ subscribe(since,send)    │   │                        │
│   │  │        │ │ retry, state) │ │                          │   │                        │
│   │  └───┬────┘ └──────┬────────┘ └────────────┬─────────────┘   │                        │
│   │      │             │                       │                 │                        │
│   └──────┼─────────────┼───────────────────────┼─────────────────┘                        │
│          │             │                       │                                          │
│          ▼             │                       │   ┌───────────────────────────┐          │
│   data/events/         │                       │   │  /api/events/*  (routes)   │          │
│   (shards+state+index) │                       └──▶│  /api/events/stream (NDJSON)      │
│                        │                          └──────────────┬──────────────┘      │
│                        │                                         │ fetch / NDJSON       │
│   agent tools ◀────────┼─────────── (in-process, no HTTP hop) ───┤                     │
│   (api.ts, serverTool) │                                         ▼                     │
│                        │                                    Event Viewer + Bell (browser)│
│                        │  dispatch engine reaches service workers:                      │
│                        │         ▼                                                       │
│                 ServiceManager ──(worker IPC: event_dispatch)──▶ worker-thread services │
│                  (src/core/service/)                          (the EMITTERS / handlers)│
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

BOS containers the design **calls into** (does not create): the **ServiceManager/worker-IPC** channel (to dispatch to worker-thread services), **`data/`** persistence, the **NDJSON streaming** route pattern, the **OS window store** (`launch()`), and the **assistant tool registry** (`serverTool`). None of these are new.

**Why there is no "service with its own network port" in this design (and why the `services.md` §11 reachability machinery is N/A here):** the event kernel is an in-process daemon — it has no listening socket, so the "a service's port is never reachable behind a reverse proxy / must use `wsPath`/`httpPath`" concern does not arise for the kernel itself. The *emitters* are worker-thread services that already exist with their own reachability; they reach the kernel by **loopback HTTP to BOS's own API** (the documented "reaching the VFS from a service" pattern, `services.md` §"Reaching the VFS from a service"), not by being exposed publicly. Nothing in this design shows a user a connection URL to a new port, so §11's three-scenario table does not apply. I state this explicitly because it is the one place a reviewer's instinct ("check §11") would mis-fire.

### 3.3 Component (C4 — the actual modules this feature creates/modifies)

Two families: **the new event subsystem** and **the integration seams** into existing subsystems.

**A. Event kernel (`src/lib/events/`, all `import "server-only"` except `types.ts`)**

| Module | Responsibility | Grounded pattern |
|---|---|---|
| `types.ts` | Framework-free shared types: `EventRecord`, `EventState`, `HandlerRegistration`, `HandlerAcknowledgment`, `HandlerPreference`, `ProcessingStatus` (`pending`\|`processed`), `ReadStatus` (`unread`\|`read`), `HandlerMode` (`headless`\|`ui`). Imported by both kernel and viewer. | `src/os/types.ts` (framework-free shared types) |
| `store.ts` | Persistence. Immutable event bodies in **per-month JSONL shards**; mutable per-event state (`processing`, `read`, `history`, per-handler status) in **per-month state files**; a **persisted index snapshot** (`index.json`) of small projections for fast boot; a **per-process mutex** (read-modify-write). All writes via `writeFileAtomic`. | `src/lib/integrations/notifications/store.ts` (mutex + `writeFileAtomic`), `src/os/atomic-write.ts` |
| `kernel.ts` | The `globalThis`-backed singleton daemon. Public ops (`emit`, `ack`, `query`, `register`, `unregister`, `markRead`, `setPreference`, `unreadCount`, `getEvent`). Owns the two-axis state machine and re-dispatch-on-boot. Started by `startEventKernel()`. | `src/core/service/ServiceRegistry.ts` (globalThis singleton), `src/lib/scheduler/engine.ts` (in-process daemon) |
| `dispatch.ts` | Fan-out on emit; **per-handler FIFO queue** (concurrency 1 → free ordering); **concurrent across handlers** (NFR-007); **retry with backoff** (1s/5s/30s, 3 retries → `permanently_failed`); per-handler timeout (default 30s, NFR-006); re-dispatch of un-acked events on boot and on late handler registration. | `src/core/service/workerIpc.ts` (`callId` correlation), `src/core/service/CrashRecovery.ts` (exponential backoff) |
| `stream.ts` | In-memory "state-change" event stream with a global `streamSeq`; `subscribe(since, send)` (replay then tail). Distinct from the per-type `sequence` on events. | `ServiceRegistry.subscribe` / `GET /api/services/events` |
| `api.ts` | The **public event API** — a thin facade over the kernel exposing the one contract. This is what agent tools call in-process and what the HTTP routes delegate to. | `src/lib/integrations/notifications/store.ts` (exported op surface) |
| `migrate-integrations.ts` | One-time, idempotent migration of the legacy `notifications.json` inbox into events (see §3.4). Runs from `instrumentation.ts`. | `src/lib/marketplace/migrate-user-apps.ts` (boot-time idempotent migration) |

**B. Integration seams (edits to existing BOS source)**

| File | Change |
|---|---|
| `src/instrumentation.ts` | In `register()`, after `serviceManager().startAll()`: run `migrate-integrations`, then `startEventKernel()` (loads store, rebuilds index/queues, re-dispatches un-acked). Gated on `NEXT_RUNTIME === "nodejs"`, same as the existing service block. (The "Add server boot-time initialization" recipe — keep it in this file, in `register()`.) |
| `src/core/service/types.ts` · `workerIpc.ts` · `ServiceManager.ts` | New worker-IPC message types `event_dispatch` (Main→Worker) and the worker's reply, plus `ServiceManager.dispatchEventToWorker(serviceId, payload, {timeoutMs})` — `callId`-keyed, mirroring the 039 `tool_call`/`tool_result` transport. The kernel calls this to invoke a service's headless handler. |
| `src/os/types.ts` | Add optional `eventHandlers?: UIHandlerDeclaration[]` to `AppManifest` (UI handlers declared statically in a built-in app's `manifest.ts`; the same field on installed apps' `app.json`). Headless handlers are **not** in the manifest (they're runtime-declared by a running service — see §3.5). |
| `src/lib/apps/store.ts` / `toManifest` | Surface the installed apps' `eventHandlers` declarations into a unified UI-handler registry the viewer can query. |
| `src/components/desktop/Topbar.tsx` | Replace `<IntegrationsBadge />` with `<EventBell />`. |
| `src/components/desktop/EventBell.tsx` (new) | The bell: unread count badge (caps "99+"), subscribes to `GET /api/events/stream`, click → `launch("event-viewer")`. |
| `src/apps/event-viewer/manifest.ts` + `index.tsx` (new) | The built-in app. `manifest.ts`: id `event-viewer`, icon `Bell`, `singleton: true`, ~780×560. `index.tsx`: Events tab (windowed list + generic detail + selection dialog) and Configuration tab; fetches `/api/events/*`, subscribes to `/api/events/stream`. |
| `src/app/api/events/**` (new routes) | Thin delegates to `api.ts` (see §3.6). |
| `src/lib/assistant/tools/server/events.ts` (new) | Agent tools via `serverTool`, calling `api.ts` in-process. |
| `src/lib/integrations/notifications/store.ts` + emitters | Emitters (Gmail/Telegram adapters, scheduler jobs) switch from `emitNotification` to `api.emit` with namespaced types; the old store is retired (its data migrated). |

### 3.4 Data model & storage layout

Two orthogonal axes, stored separately for performance (never rewrite a 1MB payload to flip a `read` bit):

```
data/events/
  index.json                     # persisted projection index: { eventId -> {type, seq, ts, processing, read, shard, month} }
  events/<YYYY-MM>.jsonl         # IMMUTABLE event bodies, one JSON object per line, append-only
  state/<YYYY-MM>.json           # MUTABLE per-event state for that month: { eventId -> {processing, read, history[], perHandler{} } }
  handlers.json                  # handler registry (service-declared + core-internal): enabled state, timeout, recent-failure count
  preferences.json               # user prefs: default UI handler per event type
```

- **`EventRecord`** (immutable, in shards): `id`, `type` (dot-namespaced, ≤256 chars), `payload` (JSON ≤1MB), `source` `{appId, name, icon?}`, `ts` (epoch ms), `sequence` (**per-type** monotonic, FR-003), `summary` (derived: `payload.summary` else truncated).
- **`EventState`** (mutable, in state files): `processing` (`pending`\|`processed`), `read` (`unread`\|`read`), `history: HandlerAcknowledgment[]`, `perHandler: { handlerId -> status }`.
- **`HandlerAcknowledgment`**: `eventId`, `handlerId`, `appId`, `attempt`, `ts`, `status` (`acked`\|`failed`\|`permanently_failed`), `result?` (JSON), `error?`.
- **`HandlerRegistration`**: `handlerId`, `eventType` (exact or prefix), `mode`, `ownerId` (app/service id), `displayName`, `description?`, `icon?`, `enabled` (user-controllable, headless only), `timeoutMs` (default 30000), `declaredBy` (`service` | `core`).
- **`HandlerPreference`**: `eventType`, `preferredHandlerId`, `ts`.

**Boot (at-least-once, FR-005a):** `startEventKernel()` loads `index.json` (fast) — if missing, rebuilds by scanning shards (100k JSON lines is fast, seconds, and happens once at boot, well before any "bell click → list render" budget since the viewer opens after boot). It reconstructs per-handler FIFO queues from un-acked events and re-dispatches them to active handlers. Handlers whose service isn't running are not active; their events stay `pending` until that handler (re)registers (late catch-up, FR-005b — the `handler_declare` path in §3.5 is the trigger).

**Emission (NFR-001 <100ms):** `emit` = acquire mutex → append body to the current month shard → create state record → bump per-type `sequence` → update `index.json` entry → release mutex → return. Then (non-blocking, after the durable record + return) the dispatch engine fans out. The emitter's call returns once the body + state are durably written; handler completion is not awaited (FR-004). Index `index.json` writes are the one full-file write per emit — bounded and small (projections only), acceptable; if profiling disagrees, ADR-5's DB option removes it.

### 3.5 Handler registration & invocation

**UI handlers — static, declared, click-resolved.** Declared in an app's manifest (`AppManifest.eventHandlers`, built-in `manifest.ts` or installed `app.json`). The viewer builds a UI-handler registry from all installed + built-in manifests. **Resolution at click** (NFR-004 <100ms, in-memory):
- 0 UI handlers for the type → the **generic detail view** inside the Event Viewer (full payload + history). This is always available.
- exactly 1, or a user-set default (`preferences.json`) → **launch** that app: `launch(appId, { event: <eventId>, handler: handlerId })`. `launch()` honors `singleton` and merges params (`src/store/os-store.ts`), so re-clicking focuses the same window with fresh params.
- >1 and no default → the **selection dialog** (mockup "Open with…") with "Always use this app for this event type" → writes `preferences.json`.

UI handlers never call `ack` and never affect `processing` (FR-014). "Launching" a UI handler = opening the owning app's window with the event as a launch param; the app renders its event-specific view. This works uniformly for built-in (React component reads `params`) and marketplace (iframe receives params) apps — no component import needed at resolution time.

**Headless handlers — dynamic, runtime-declared, service-owned.** Not in the manifest. A worker-thread service declares its headless handlers at startup by posting `handler_declare` over the existing worker IPC (the same shape as 039's `tool_declare`, ADR-3). The kernel registers them (owner = the service id) in `handlers.json` + memory. Consequences fall out for free:
- **Late-registration catch-up (FR-005b):** when a (re)started service posts `handler_declare`, the kernel adds it to the active set and re-dispatches matching un-acked events.
- **Service-offline = not active (FR-019/edge cases):** when a service stops, `ServiceManager`'s stop/crash path (which already does `unregisterServiceTools` for 039) also tells the kernel to mark its handlers inactive and remove them from the active set.
- **Uninstall (FR-020):** the service uninstall path removes its handlers; pending events are re-evaluated → processed if no other active handlers remain.

**Core-internal headless handlers** (bos-core code reacting to events, e.g. a future in-core workflow trigger) register directly in-process with `api.register({ownerId: "<core>", declaredBy: "core"})` — the dispatch engine calls them as plain functions (no IPC). This keeps a path for core automation without requiring a service.

**Invocation (core → service worker):** `dispatch.ts` calls `ServiceManager.dispatchEventToWorker(serviceId, {eventId, eventRecord, handlerId}, {timeoutMs})`. The worker (unbundled, plain Node — it cannot `import` any `src/` module) runs its own handler function, then **acknowledges via the public `ack` API** (see transport note below). The kernel, on each `ack`/final-failure, updates `perHandler`, and when **all active handlers** for the event have acked or permanently failed → `processing = processed` (FR-008). An event type with no active headless handlers is `processed` immediately on emit.

**Ack ownership (FR-022):** the kernel validates `ack`'s caller owns the handler. Trust model: single-container (per `services.md`, the container boundary is the isolation; there is no cross-user adversary inside one user's container). For in-process/IPC acks the owner is known from the channel (service id / core); for HTTP acks the caller identifies its `ownerId` (a loopback, same-container request). This is a logical integrity check, not a network security boundary.

**Namespace validation (FR-023):** a component may register handlers only for type namespaces it owns. Convention: a component's owned root is declared in its manifest/`service.json` (e.g. a service whose id maps to `com.bos.<id>.*`); registration validates the prefix. Same single-container trust model as ack ownership — a soft guard, documented in the developer docs (FR-026).

### 3.6 The one public API, three transports (answers "server routes? in-process? both?")

**Both — sharing one kernel.** `api.ts` is the single contract. Transports:

| Transport | Who | Why |
|---|---|---|
| **In-process** (`api.ts` direct) | agent tools (`serverTool`, no HTTP hop, exactly the `integrations.ts`/`scheduler.ts` pattern), core-internal code | lowest latency; the "agent tool API" literally |
| **Same-origin HTTP** (`/api/events/*`) | Event Viewer, Event Bell (browser) | the browser can only reach BOS this way |
| **Loopback HTTP** (`http://127.0.0.1:$PORT/api/events/*`) | worker-thread services (emit + ack) | the documented "service reaches BOS via loopback" pattern (`services.md` "Reaching the VFS from a service"); works identically for core-adjacent and marketplace services; satisfies "the API must work from service worker-thread context" |

**Routes** (`src/app/api/events/`, thin delegates to `api.ts`; `import "server-only"`):

| Route | Methods | Maps to |
|---|---|---|
| `/api/events` | `POST` (emit) · `GET` (query: `?type&status&read&from&to&page&cursor`) | `api.emit` / `api.query` |
| `/api/events/stream` | `GET` (NDJSON `?since=<streamSeq>`) | `stream.subscribe` (mirrors `/api/services/events`) |
| `/api/events/:id` | `GET` | `api.getEvent` (body + state + history) |
| `/api/events/:id/ack` | `POST` | `api.ack` |
| `/api/events/register` · `/unregister` | `POST` | `api.register` / `api.unregister` |
| `/api/events/:id/read` | `POST` (mark-read; `?all=1` for mark-all) | `api.markRead` |
| `/api/events/preference` | `POST` | `api.setPreference` |
| `/api/events/count` | `GET` | `api.unreadCount` (cheap bell poll fallback) |
| `/api/events/handlers` | `GET` · `POST` (enable/disable headless) | registry read / toggle |

`/api/events/stream` is a copy of the `GET /api/services/events` shape (I read `route.ts`): `ReadableStream`, `?since` replay-then-tail, 25s keepalive, `maxDuration`, `x-accel-buffering: no`. The Event Viewer subscribes for FR-028/NFR-009 live updates (new event, `processing` flip, `read` flip) — push, so the ≤1s budget is met with margin. The bell subscribes to the same stream (or polls `/count`) for NFR-008.

### 3.7 Real-time updates (FR-028 / NFR-009)

- **Event Viewer:** `GET /api/events/stream` subscription. On `new` → prepend to list (with the mockup's flash highlight). On `processed` → update the row's badge in place. On `read` → move between unread/historical. No polling, no refresh.
- **Bell:** same stream (or cheap `/count` poll as fallback). Unread count updates on `new` (increment) and `read`/mark-all (decrement), ≤1s.
- The viewer's initial list (<500ms from bell click, NFR-002) reads the **warm in-memory index** (a server process is running; the index is in memory), so it never scans disk per render.

### 3.8 GSuite / notification migration

Existing mechanism (confirmed in source): `src/lib/integrations/notifications/store.ts` (append-only `notifications.json`, 500-item cap, per-process mutex) + `src/components/desktop/IntegrationsBadge.tsx` (topbar, polls `?count=1`, click = mark-all-read) + emitters in the Gmail/Telegram adapters and scheduler jobs calling `emitNotification(event)`.

Migration plan:
1. **Re-point emitters.** Every `emitNotification(ev)` becomes `api.emit({ type: <namespaced>, source: {appId:"gsuite"|...}, payload: <ev> })` using the mockup's type conventions (`com.bos.gsuite.email.received`, etc.). The `IntegrationEvent` payload maps directly into the event payload.
2. **One-time data migration** (`migrate-integrations.ts`, from `instrumentation.ts`, idempotent, guarded by a marker file so it runs once): read `notifications.json`, re-emit each item as an event preserving `read` state and `ts`, then archive the old file. No existing unread notification is lost (SC-008 spirit).
3. **Retire the old surface.** `IntegrationsBadge` → `EventBell` (FR-009: the existing bell is repurposed). The old `notifications.json` store and its route are removed once migration has run on a real deployment (gated behind the marker; safe to delete in the same branch since data is preserved in the new store).
4. **GSuite UI handler.** GSuite declares a UI handler for `com.bos.gsuite.email.received` that opens the email detail (existing BOS email UI). This is the concrete payoff: clicking a migrated email event launches the same view the old inbox implied.

---

## 4. Concrete file/module plan (what the Developer creates/modifies)

**Create — event subsystem** (`src/lib/events/`): `types.ts`, `store.ts`, `kernel.ts`, `dispatch.ts`, `stream.ts`, `api.ts`, `migrate-integrations.ts`.

**Create — API & tools:** `src/app/api/events/route.ts` (+ `/stream`, `/:id`, `/:id/ack`, `/register`, `/unregister`, `/:id/read`, `/preference`, `/count`, `/handlers`); `src/lib/assistant/tools/server/events.ts`.

**Create — built-in app** (`src/apps/event-viewer/`): `manifest.ts` (id `event-viewer`, icon `Bell`, `singleton: true`), `index.tsx` (+ internal components: `EventsList`, `EventDetail`, `HandlerDialog`, `ConfigPanel`; `npm run gen:apps` discovers it — no registry edit).

**Create — topbar:** `src/components/desktop/EventBell.tsx`.

**Modify — integration seams:** `src/instrumentation.ts` (start kernel + migration); `src/components/desktop/Topbar.tsx` (swap badge); `src/os/types.ts` (`eventHandlers?` on `AppManifest`); `src/lib/apps/store.ts`/`toManifest` (surface installed UI-handler declarations); `src/core/service/types.ts` + `workerIpc.ts` + `ServiceManager.ts` (`event_dispatch` IPC + `dispatchEventToWorker`); GSuite/Telegram/scheduler emitters (`emitNotification` → `api.emit`); retire `src/lib/integrations/notifications/store.ts` + its route.

**Docs (FR-026):** `docs/dev/events/` (API reference, handler registration UI+headless, idempotency, retry/timeout semantics, namespace conventions, payload limits, integration examples) and `docs/usage/` (viewing, processing status, configuration, enable/disable, reading processing history/failures). Plus an `architecture-overview.md` entry and a `extending-bos.md` "Add an event handler" recipe.

**Tests (FR-029/030):** unit tests under `tests/` for the kernel (emit/dispatch/ack/retry/permfail/re-dispatch/register/unregister/ack-auth/payload-validation), the read/unread + processing state machine, the preference system, and `sequence` assignment — **each test cleans up** (remove created events/handlers/prefs in `afterEach`) so the store isn't polluted. Playwright e2e under `e2e/034-event-notification-system.spec.ts` covering the spec's listed flows (emit→viewer, click→read+launch, selection dialog + always-use, config enable/disable, generic view, historical toggle, bell accuracy) — **each test reverts** its UI/state in teardown.

---

## 5. Integration points (existing BOS mechanisms this design calls into, does NOT create)

- **`src/core/service/ServiceManager.ts` + `workerIpc.ts`** — worker-IPC channel to reach service workers (the dispatch path); the 039 `callId` transport is the template for the new `event_dispatch` messages.
- **`src/instrumentation.ts` `register()`** — boot hook where the kernel starts (scheduler/services precedent).
- **`src/os/atomic-write.ts` (`writeFileAtomic`)** — crash-safe persistence for all event store writes.
- **`GET /api/services/events` NDJSON pattern** — the exact shape copied for `/api/events/stream` (replay + tail + keepalive).
- **`src/store/os-store.ts` `launch()`** — bell→viewer and viewer→UI-handler window launching; honors `singleton` + param merge.
- **`src/lib/assistant/...` `serverTool` + tool registry** — how the agent tools are declared and gated (`integrations.ts`/`scheduler.ts` pattern).
- **`src/lib/apps/store.ts` (`listInstalledManifests`, `toManifest`)** — installed-app manifest discovery for the UI-handler registry.
- **`data/` runtime-state convention** — `data/events/` (gitignored), consistent with `data/integrations/`, `data/services`, etc.
- **`src/lib/marketplace/migrate-user-apps.ts` pattern** — boot-time idempotent migration for the legacy inbox.

---

## 6. ADRs

### ADR-1 — The event kernel is an in-process daemon, not a worker-thread service
- **Context:** The spec's "core event service (starts at boot, runs for session lifetime, dispatches to handlers)" could be read as a `src/core/service/` worker-thread daemon.
- **Options:** (a) a `src/core/service/`-style worker-thread service on its own port; (b) an in-process server singleton (scheduler-daemon shape); (c) a marketplace-item service facet.
- **Decision:** **(b)** — a `globalThis` singleton in `src/lib/events/`, started from `instrumentation.ts`, sibling to the scheduler daemon.
- **Consequences:** No new process/port, no `services.md` §11 reachability concerns, no `package.json` change. It *reuses* the worker-thread service machinery (ServiceManager IPC) to *reach* services, but is not itself one. Rejecting (a): the kernel is not independently installable/lifecycle-managed like a service, and a port would invite the exact §11 pitfalls. Rejecting (c): a broker that BOS core owns, starts unconditionally at boot, and every service depends on cannot be an opt-in installable item. **This is the load-bearing decision; if a reviewer disagrees, everything else is downstream of it.**

### ADR-2 — One kernel, three transports ("server routes? in-process? both?" → both)
- **Context:** FR-001 says the public API is the same for agents, services, and apps; emitters/handlers live in very different execution contexts (assistant server code, browser, worker threads).
- **Options:** (a) HTTP only; (b) in-process only; (c) one kernel API + multiple thin transports.
- **Decision:** **(c).** `api.ts` is the single contract; agent tools call it in-process (no HTTP hop, the `serverTool` norm), the viewer/bell use same-origin HTTP, worker-thread services use loopback HTTP.
- **Consequences:** "The same API agent tools use" is literally true (one operation set, one implementation). Each context uses the transport it can actually reach. The loopback-HTTP choice for services matches the documented VFS-bridge pattern and needs no new mechanism. Cost: a small route layer (thin delegates) — standard for every BOS subsystem.

### ADR-3 — Headless handlers are runtime-declared over worker IPC, not manifest-declared
- **Context:** UI handlers are UI (static, click-resolved). Headless handlers are executable code that runs in a worker thread and only exists while its service runs.
- **Options:** (a) declare headless handlers in the manifest (like UI handlers); (b) declare at runtime via a `handler_declare` IPC message (the 039 `tool_declare` pattern).
- **Decision:** **(b)** — a service posts `handler_declare` at startup; the kernel registers (owner = service id). Reuses the trusted-by-construction worker-IPC channel; no new transport.
- **Consequences:** Late-registration catch-up (FR-005b) and service-offline = not-active (FR-019) fall out for free from the same lifecycle hooks that already manage 039 tools (`ServiceManager` stop/crash/uninstall). A manifest-declared headless handler (a) would describe code that may not be running and would need a separate "is the owner live?" check — strictly more machinery. UI handlers remain manifest-declared because they *are* static UI.

### ADR-4 — File-based storage (shards + per-event state + index), no embedded DB, by default
- **Context:** 100k+ events, indefinite retention, ≤500ms initial list, ≤1s updates (NFR-002/003, SC-002). The legacy store is a single 500-capped JSON file — inadequate.
- **Options:** (a) single JSON file; (b) per-month JSONL shards (immutable bodies) + per-month state files (mutable) + a persisted index snapshot; (c) embedded DB (SQLite).
- **Decision:** **(b)** — no new dependency; immutable/mutable split means flipping `read`/`processing` never rewrites a 1MB payload; the warm in-memory index serves the list view with no disk scan per render; `writeFileAtomic` for crash safety; per-process mutex for serialization.
- **Consequences:** Fits the "runtime state = files under `data/`" convention and avoids a `package.json`/lockfile change (Constitution VII). Boot rebuild (scanning shards) is one-time and off the viewer-latency path. **If profiling shows the index or shard scan misses the floors at 100k+, the documented escape hatch is (c) SQLite — but that is a dependency change requiring explicit approval (VII) and is deliberately not the default.**

### ADR-5 — Hand-rolled fixed-row windowing for the list (virtualization) by default
- **Context:** FR-025 (paginate or virtualize) + SC-002 (60fps at 100k).
- **Options:** (a) a dependency (`@tanstack/react-virtual`); (b) a hand-rolled fixed-row-height windowed list (render only the visible slice ± overscan, backed by the index).
- **Decision:** **(b)** by default; a built-in app is React and the event rows have uniform height, so a small windowing component suffices with zero dependencies.
- **Consequences:** No `package.json` change. If UX testing shows the hand-rolled version is insufficient (variable-height rows, drag-select, etc.), adopt (a) as a scoped dependency change. **Flagged for `plan.md` Complexity Tracking.**

### ADR-6 — Bell shows UNREAD (not pending); the viewer is the host of the generic view + dialog
- **Context:** Two orthogonal axes (processing vs read) create ambiguity about what the bell counts and where the detail lives.
- **Options:** bell = pending vs bell = unread; generic detail in the viewer vs always in the owning app.
- **Decision:** Bell = **unread** count (matches the spec's "architectural review" clarification and the mockup). The Event Viewer renders the generic detail view and the selection dialog; a registered UI handler *launches* the owning app's window (mockup "Open with…", User Story 5, and the spec's "UI handler components launch in their own window or pane").
- **Consequences:** The generic view is always reachable in the viewer (zero-handler fallback, FR-017) without opening an app; UI-handler "launch" reuses `launch()` with event params. The pending/processed state is a per-row badge, not the bell — keeping the user-facing "what haven't I seen" (unread) distinct from pub/sub internal state.

---

## 7. Risks / open questions

1. **Dispatch engine complexity** (the per-handler FIFO + retry + at-least-once + two-axis state machine) is the riskiest code. Mitigations: isolated in `dispatch.ts`, fully covered by FR-029 unit tests, and the "no active handlers → immediately processed" + "disabled/offline handler is not active" rules are simple to assert. A reviewer should scrutinize the exactly-once-ack-per-(event,handler) invariant under concurrent handler fan-out (a `callId`/pair key must prevent double-settle).
2. **Ack transport asymmetry** (invocation via IPC; ack via the public API which, from a worker, is loopback HTTP — or IPC-forward). I recommend the worker **ack over loopback HTTP to `/api/events/:id/ack`** to keep "ack is the public API" literal and testable, and note the IPC-forward alternative. **Open for plan:** pick one and wire it; both are grounded, but only one should ship to avoid two ack paths.
3. **How a worker-thread service discovers BOS's loopback port** — it reads its own process's `process.env.PORT` (the worker is spawned by the process that serves HTTP; under the Supervisor that's the base or the preview's port, so loopback is always correct). I'm **inferring** the exact discovery line (the docs cite the loopback pattern, not the precise env lookup) — confirm in `plan` against `ServiceManager`/`instrumentation`.
4. **`index.json` write-per-emit** at high volume is the one full-file write in the hot path. It's small (projections only) and acceptable, but if a burst of emits contends, consider batching index flushes. Low risk; note for profiling.
5. **Namespace-ownership rule** (FR-023) is a soft convention under single-container trust — the exact "owned root" derivation (id → `com.bos.<id>.*`? explicit grant list?) needs a one-paragraph decision in `plan` + developer docs.
6. **Built-in vs marketplace UI-handler launch parity** — `launch()` + params works for both, but a marketplace (opaque-origin) app receiving event params must read them via the documented broker/postMessage channel. Confirm the installed-app param delivery path in `plan` (the mockup only exercises built-in apps).
7. **GSuite email UI handler** depends on existing email-detail UI being launchable with an event param — verify the existing email view accepts a mail id/param (open for plan; if not, the handler renders a thin detail from the payload).
8. **Migration idempotence & ordering** — the `notifications.json` → events migration must run before the kernel starts serving and must be a no-op on repeat boots (marker file). Confirm no race with the scheduler, which also currently emits to the old store during the migration window.

---

## 8. UI mockup reference

**Path:** `/Specs/user-specs/core-platform/034-event-notification-system/mockup.html` (interactive).

Its screens map onto the Component design as follows:

| Mockup screen | Component | Notes |
|---|---|---|
| **Topbar bell + badge** (`bellBadge`, caps "99+") | `EventBell.tsx` (replaces `IntegrationsBadge`) | unread count; subscribes to `/api/events/stream`; click → `launch("event-viewer")`. |
| **Event Viewer window** (title bar, toolbar with Events/Configuration tabs, footer "N events · M processing · Live") | `src/apps/event-viewer/index.tsx` shell | `singleton`; footer "Live" indicator = active stream subscription (FR-028). |
| **Events list** (`eventRow`: app chip + unread dot, source name, summary, `type`/`seq`, status badge "processing…"/✓, relative ts; "Show historical events" toggle; "Mark all as read") | `EventsList` (windowed, ADR-5) | unread-by-default view; `processing` badge from `EventState.processing`; historical toggle flips the `read` filter; mark-all → `POST /:id/read?all=1`. |
| **Event detail / generic view** (header, meta chips `type`/`seq`/`id`/`ts`, pending banner "X of Y handlers acknowledged", Payload table, Processing history with per-handler status/attempt/result/error) | `EventDetail` | This is the FR-017 generic view. Data from `GET /api/events/:id` (body + state + history). "No UI handlers → showing the generic event view" note = the 0-handler fallback. |
| **Handler selection dialog** ("Open with…", radio list of UI handlers, "Always open <type> with this app" checkbox, "Open in <app>") | `HandlerDialog` | FR-016; "always use" → `POST /api/events/preference`; confirms → `launch(appId, {event, handler})`. Shown only when >1 UI handler and no default. |
| **Configuration page** (grouped by event type; Headless handlers panel with enable/disable toggle + recent-failure count; UI handlers panel with "Set as default"/"Clear" + Default badge) | `ConfigPanel` | FR-018; handlers from `GET /api/events/handlers`; toggle → `POST /api/events/handlers`; default → preference API. |
| **Real-time behaviors** (new-event flash `event-flash`; delayed pending→processed on event `e3`; bell live count) | stream subscription in `EventsList`/`EventDetail`/`EventBell` | FR-028/NFR-009 — push via NDJSON, no refresh. |

The mockup is a single-window demo; in BOS the window chrome (traffic-light buttons, pin) is the standard `Window.tsx` frame, and the "UI handler launches" cases open the *owning app's* window via `launch()` (not a new in-viewer screen) — the mockup models that as the "Open in <app>" confirmation.
