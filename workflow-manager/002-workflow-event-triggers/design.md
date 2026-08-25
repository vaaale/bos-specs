# Workflow Event Triggers — Design

**Spec**: `user-specs/workflow-manager/002-workflow-event-triggers/`
**App Target (spec.md)**: `marketplace-item` — **agreed** (see §1). No BOS-source change required; the one manifest-level addition (`eventNamespaces`) lives in the item's *own* `service.json`, not BOS source.
**Date**: 2026-08-25
**Baseline**: `001-workflow-manager-service-tools` (implemented & promoted) — the `workflows` marketplace item whose service facet owns the complete workflow engine (node model, scheduler, routing, persistence, execution) and exposes it as 039 service-declared tools, with the app facet as the graph UI.

> **Scope.** An incremental scope-add to the installed `workflows` item: a workflow may declare one or more **event triggers** (each naming an exact BOS event type); the service subscribes to the union of those types; when a matching event is published, the service automatically starts a run, injects the event as the run's input, and records provenance. Everything below is inside the item (service facet + app facet + its own manifest); the event system (034) and the 039 tool/IPC channel are **called into, not modified** (§5).

---

## 1. Classification

**App Target: `marketplace-item`** (service facet + app facet + the item's own `service.json`). **Agrees with spec.md's `App Target` field.**

Rationale (per `target-marketplace-item.md` — app and service are two facets of one item, not two targets):

- **Service facet** (`data/user-apps/items/workflows/services/`): gains the *trigger engine* — maintain the subscription set (subscribe/unsubscribe over the 034 headless-handler channel), receive `event_dispatch`, match + fan-out, start runs, inject the event payload, ack, and persist trigger provenance on the run. It also extends the 039 tool surface to author/read triggers (FR-008).
- **App facet** (`data/user-apps/items/workflows/app/`): gains the **Triggers panel** in the workflow detail view (add/edit/remove/list) and the **⚡ provenance** rendering in run history + live run (mockup-driven, §8).
- **The item's own `service.json`**: gains a static `eventNamespaces` grant (§3.5, ADR-1) so the service may subscribe to BOS event types outside its owned root. This is the *item's* manifest — not a BOS-source change. `ServiceManifest` already declares `eventNamespaces?: string[]` (`src/core/service/types.ts`, added by 034), and `manifestValidator.ts` does not reject it (it simply isn't validated, so any value passes) — so no BOS change is needed to accept the grant.

**No `bos-core` scope.** Unlike baseline 001 (which carried a bos-core *retirement*), this increment modifies zero BOS-source files. The event kernel, the worker-IPC `event_dispatch`/`handler_declare` messages, the loopback event API, and the 039 tool bridge all already exist. The feature is entirely an item change.

**Agreement with spec.md**: the spec states "No BOS-source change is required" and `App Target: marketplace-item`. This design concurs on both. The one place that needs reconciliation is *how* the service is allowed to subscribe to arbitrary BOS events — resolved by a manifest `eventNamespaces` grant (still an item-only change), and flagged in §7 because it sharpens the spec's mental model (see Open Question #1).

---

## 2. Constitution Check

| Principle | Compliance |
|---|---|
| **II. Server Authority & SSR Boundary** | Complies. The service talks to BOS only over (a) worker IPC (`handler_declare` it emits; `event_dispatch` it receives; `tool_call`/`tool_result` for the 039 tools) and (b) loopback HTTP to BOS's own API (`/api/events/unregister`, `/api/events/:id/ack` — the documented worker-thread loopback pattern, `src/lib/events/loopback.ts`; and the baseline's `/api/fs`, `/api/subagents/delegate`). The worker never imports `@/` code. The event *kernel* runs on BOS's main thread; the service is only a handler target. |
| **IV. Minimize Blast Radius** | Complies — stronger than baseline: **zero** BOS-source files touched. All changes are under `data/user-apps/items/workflows/` (installed via `app_build`, which lands on the item's own `app-candidate` branch). No feature branch / `dev_delegate` needed (per `target-marketplace-item.md`). |
| **V. The VFS Is Not the Source** | Complies. Triggers persist *inside* the workflow JSON at the real VFS `/Workflows/<id>-workflow.json` (NFR-003), reached via loopback `/api/fs` (baseline ADR-2) — never a host path. Run provenance extends the run entity at `/Workflows/.runs/<workflowId>/<runId>.json`. |
| **VI. Specs & Docs Stay in Sync** | Complies. `docs/dev/events/` (034 developer docs) already document the headless-handler runtime-declaration, the loopback transport, and the `eventNamespaces` grant mechanism this feature relies on; no doc change is required for an *item* that consumes a documented mechanism. If implement surfaces any spec/code drift in the 034 event subsystem, record it in `discrepancies.md` (see §7, Open Question #3). |
| **VII. Respect Boundaries** | Complies. No new external dependency — the design reuses the existing 034 event API + 039 IPC channel + baseline loopback `/api/fs`. No `package.json`/lockfile change. |

No conflicts flagged.

---

## 3. Architecture

### 3.1 Context (C1)

The user/assistant authors a workflow and attaches event triggers (via the app's Triggers panel or the `workflow_create`/`workflow_modify`/`workflow_read` tools). Independently, any BOS component publishes durable events onto the event bus (034). When a published event's type matches a live trigger, the `workflows` service — running as a headless handler registered with the 034 event kernel — is invoked, starts a run, injects the event payload as the run's input, and acks. The run is an ordinary workflow run (streams, persists, cancellable, inspectable) with added provenance.

```
   ┌──────────────┐   author/read triggers (tools)   ┌──────────────────────────┐
   │  Assistant   │ ───────────────────────────────▶ │  workflows SERVICE (worker│
   │  (BOS)       │   workflow_create/modify/read     │  thread, deploymentMode   │
   └──────────────┘                                   │  "tools") — owns engine   │
        │                                             │  + NEW trigger engine:     │
        │                                             │  subscribe, match, fan-out,│
   ┌──────────────┐   configure triggers (UI)         │  fire run, inject, ack     │
   │  User / app  │ ───────────────────────────────▶  └───────┬──────────┬────────┘
   │  (iframe)    │   Triggers panel / ⚡ run history          │          │
   └──────────────┘                                   emit/recv│          │ loopback HTTP
                                          ┌──────────────────────┐          │ (BOS origin)
                                          │  BOS main thread      │          │
                                          │  EVENT KERNEL 034     │  ack      │
   any BOS emitter ── emit(type,payload) ▶ │  (src/lib/events/*)  │ ◀─────────┤
   (gsuite, gitops,        (durable)      │  dispatch engine     │  unregister│
    integrations, …)      ──────────────▶ │  per-handler FIFO    │          │
                                          └──────────┬───────────┘          │
                                                     │ event_dispatch (IPC)  │
                                                     └──▶ start run(s) ──────┘
```

### 3.2 The headless-handler runtime-declaration wire protocol (Decision 1)

This is **not** something the service invents — the wire protocol already exists in 034 and the 039 channel. The design's job is to specify *which* messages the service uses and the exact data that crosses the boundary. Everything below is verified against source.

**Two directions of the existing 034/039 worker-IPC protocol** (`src/core/service/types.ts`):

1. **Subscribe — Worker→Main `handler_declare`** (the 034 ADR-3 runtime-declaration path, mirrors `tool_declare`).
   Payload (`HandlerDeclarePayload`):
   ```ts
   { callId: string;            // correlation id the service generates per declaration
     handlerId: string;         // service-owned, unique; see §3.4 handlerId scheme
     eventType: string;         // the exact event type to subscribe to
     displayName: string;
     description?: string; icon?: string;
     timeoutMs?: number }       // handler invocation timeout (default 30000)
   ```
   ServiceManager's `handler_declare` handler (`ServiceManager.ts`) calls `eventsApi.register({ handlerId, eventType, mode:"headless", ownerId: <serviceId>, …, declaredBy:"service", grantedNamespaces: manifest.eventNamespaces })`. Registration is an **idempotent upsert** (re-declaring the same `handlerId` preserves a previously-set `enabled`), and 034's late-registration catch-up (`catchUpHandler`) re-dispatches any un-acked pending events of that type. Note: `handler_declare` does **not** require `deploymentMode:"tools"` (only `tool_declare` does), so it works for this service regardless.

2. **Invocation — Main→Worker `event_dispatch`** (the dispatch engine calling the handler).
   Payload (`EventDispatchPayload`):
   ```ts
   { callId: string;            // exactly-once-settle guard; echo on ack
     eventId: string;           // the EventRecord.id
     handlerId: string;         // which of the service's handlers fired
     record: EventRecord }      // the FULL durable event: { id, type, payload, source, ts, sequence, summary }
   ```
   This is the *only* data that crosses IPC into the handler: the complete `EventRecord` (id, type, and the full JSON `payload`), not just the type. `dispatch.ts` sends this via `sendEventDispatch`; the kernel's `setServiceInvoker` (wired in the `ServiceManager` constructor) is what fires it.

3. **Ack — Worker→BOS over loopback HTTP** (no IPC reply message exists for `event_dispatch`). The handler acks via `POST /api/events/:id/ack` with `{ handlerId, result?, callerId, callId? }`. `dispatch.ts`'s `invokeAndWait` only *confirms the IPC message was sent*; settlement arrives via this public ack API (034 R1: "services ack over loopback HTTP, exactly one ack path"). `callerId` must equal the handler's `ownerId` (the service id — `kernel.ack` enforces `reg.ownerId === input.callerId`), and `callId` is the dispatch's `callId` echoed back (exactly-once-settle guard).

4. **Unsubscribe — Worker→BOS over loopback HTTP `POST /api/events/unregister`** with `{ handlerId, ownerId }`. **This is the only removal channel.** There is **no `handler_undeclare`** worker-IPC message in the 034 protocol — the spec's Context hedges "e.g. a `handler_declare`/`handler_undeclare` emission," but the actual source has only `handler_declare` in the `WorkerToMainMessage` union (`src/core/service/types.ts`). Removal therefore goes over the loopback event API (the same transport as ack). This asymmetry — subscribe over IPC, unsubscribe over loopback — is forced by the real protocol and is deliberate (ADR-2).

**What the handler must do per `event_dispatch`** (the trigger engine's hot path):
```
on event_dispatch({callId, eventId, handlerId, record}):
  types = [ record.type ]
  workflows = subscriptionIndex[record.type]   // in-memory: event type → set of workflowIds
  for each workflowId in workflows:            // fan-out (multiple workflows may share a type)
     runId = startRun(workflowId, {           // fire-and-poll (baseline ADR-8): returns runId <1s
        source: "event",
        trigger: { eventId: record.id, eventType: record.type },   // FR-007 provenance
        input:   { event: { id, type, payload, ts, summary, source } }  // FR-006 injection
     })
     // record per-workflow outcome (runId or error) — never await the run
  ack(record.id, { handlerId, callerId: <serviceId>, callId,
                   result: { started: [runIds], failed: [{workflowId, error}] } })
  // return promptly — the runs continue asynchronously (FR-012)
```
The handler **acks after attempting** (not after run completion) — see ADR-3. `startRun` is fire-and-poll, so the ack lands well within the 30s handler timeout.

### 3.3 Container (C2) — BOS's real containers

1. **The Next.js app process (main thread)** — hosts the **034 event kernel** (`src/lib/events/*`, a `globalThis` singleton, in-process — *not* a worker-thread service), the worker-IPC channel + `ServiceManager` (which bridges `event_dispatch` to the worker and `handler_declare` to the kernel), and the loopback event API routes (`/api/events/*`). Also hosts the VFS bridge and the sub-agent delegate route the baseline uses. **This container is unchanged.**
2. **The `workflows` worker-thread service** — the item's service facet (unbundled, outside the `@/` graph, own port defaulting to `0`). It already owns the engine (baseline). This increment adds the **trigger engine** (subscription index, dispatch handler, fan-out, provenance). It binds a real port (`configSchema.port` default `0`, per ADR-3 of baseline / `target-marketplace-item.md`), but **no external client ever connects to that port** for triggers — dispatch is worker-IPC and the service→BOS calls are loopback; the port exists only because every worker service binds one (the Supervisor's proxy target). So the 034 "no new public port" property holds and `services.md` §11's reachability table does not apply to this feature.
3. **The VFS store** — workflows + run logs under `/Workflows/` (via loopback `/api/fs`), unchanged.
4. **The item's git repo** (`data/user-apps/items/workflows/`) — the item source, changed by `app_build`.

### 3.4 Component (C3) — item-level modules (the real files this feature touches)

Under `data/user-apps/items/workflows/` only. Nothing under `src/`.

```
workflows/                                    # item root
├── services/
│   ├── service.json                          # MODIFY — add "eventNamespaces": ["com.bos.*"] (§3.5/ADR-1)
│   ├── index.js                              # MODIFY — on boot: build subscription index from VFS + declare handlers;
│   │                                         #        handle event_dispatch; wire tool handlers to the trigger engine
│   ├── triggers.js                           # NEW — the trigger engine:
│   │                                         #   · subscriptionIndex: Map<eventType, Set<workflowId>>
│   │                                         #   · recompute() from persisted workflows (union of trigger types)
│   │                                         #   · sync(): diff vs. registered handlers → declare new / unregister dropped
│   │                                         #   · onDispatch(record): match + fan-out + startRun + ack  (§3.2)
│   │                                         #   · handlerId scheme + event-type validation (§3.4.1, FR-014)
│   ├── events.js                             # NEW — thin loopback event-API client for the item (unregister + ack),
│   │                                         #      reusing the baseline origin resolution (ADR-6); the service cannot
│   │                                         #      import BOS's src/lib/events/loopback.ts, so this is a small local fetch wrapper
│   ├── engine/store.js                       # MODIFY — persist workflow JSON incl. new `triggers` array; validate triggers (FR-014)
│   ├── runs.js                               # MODIFY — run entity: add startSource + trigger + input.event + seeded
│   │                                         #        `triggered` first log entry; run_list/run_get surface provenance (FR-011)
│   ├── handlers.js                           # MODIFY — workflow_create/modify accept `triggers`; read returns them (FR-008);
│   │                                         #        after any mutation call triggers.sync()
│   └── tools.js                              # MODIFY — extend create/modify/read/run_list/run_get schemas for triggers +
│                                             #        provenance (names/schemas; no new tools — see §3.4.3)
├── app/
│   └── src/main.tsx (+ TriggersPanel.tsx, RunRow ⚡, etc.)  # MODIFY — Triggers panel in detail view; ⚡ + triggered-by
│                                                            #       banner in run history + live run (mockup, §8)
└── config/
    └── workflows.json                        # unchanged ({ port: 0, host: "127.0.0.1" })
```

#### 3.4.1 Trigger + handlerId scheme + validation (FR-001, FR-014, NFR-002)

**Trigger schema** (part of the workflow JSON, persisted at `/Workflows/<id>-workflow.json` — NFR-003):
```jsonc
{
  // …existing workflow fields (id, name, version, nodes, dependencies, config)…
  "triggers": [
    { "id": "trg_<uuid>", "eventType": "com.bos.gsuite.email.received" }
  ]
}
```
- `eventType`: exactly one, **exact match** (no payload filter this increment — FR-001 / Q1).
- `id`: stable identity for the UI (edit/remove). The trigger is a *configuration of the workflow*, not a separate entity (spec Key Entities).
- Zero triggers = manual-only (spec; mockup "manual only" state).

**Handler / subscription identity.** Subscriptions are **per distinct event type** (the spec says the service subscribes to the *union* of referenced types), not per (workflow, trigger). The service maintains one headless handler per distinct event type:
- `handlerId = "wft_" + eventType` (e.g. `wft_com.bos.gsuite.email.received`). Stable across restarts (derived from the type) — required so the idempotent-upsert registration preserves a user's `enabled` toggle and so boot catch-up re-attaches cleanly. No length limit on `handlerId` in the kernel, so long types are fine.
- `displayName = "Workflow trigger: <eventType>"`, `ownerId = <service id>` (the `workflows` service id, known from `workerData.serviceId`).
- Fan-out: on `event_dispatch` for type `T`, the handler resolves *all* workflows whose triggers reference `T` (from the in-memory index) and starts a run for each (spec: "Multiple workflows may trigger on the same event type (fan-out to several workflows is allowed)").

**Validation (FR-014).** On `workflow_create`/`workflow_modify` that adds/changes a trigger, the service rejects with a clear error if the `eventType` is not (a) well-formed (dot-namespaced, the `isValidRegistrationType` shape from `src/lib/events/types.ts`) **and** (b) within the service's subscription rights — i.e. `T ∈ com.bos.workflows.*` (owned root) **or** `T` matches the `eventNamespaces` grant (§3.5). The service checks this **locally** (a ~10-line prefix check, reimplemented in plain JS — it cannot import `@/lib/events/types.ts`) *before* attempting registration, so a bad type never produces a register-then-rollback. With the default `com.bos.*` grant (§3.5), (b) reduces to "is `T` a well-formed `com.bos.*` type," so FR-014's "well-formed" requirement is effectively sufficient for every real BOS event (Open Question #1).

**Coarseness to flag.** The 034 handler `enabled` flag (toggable in the built-in Event Viewer's Configuration) is **per handler = per event type**, so enabling/disabling a handler in 034 Configuration turns *all* workflow triggers on that type together, not one individual trigger. The spec does not require per-trigger enable/disable, so this is an accepted tradeoff of the per-type subscription model (noted for the reviewer; §7).

#### 3.4.2 The trigger engine's subscription lifecycle (FR-002, FR-005, FR-010, NFR-002)

`triggers.js` owns an in-memory `subscriptionIndex: Map<eventType, Set<workflowId>>` plus the set of currently-registered handlerIds. The **single source of truth is the persisted workflow JSON** (NFR-002: "re-derivable from the persisted workflow config alone"); the index is a derived cache rebuilt on startup and updated on every mutation.

- **`recompute()`** — read all workflows from the VFS (`/api/fs`), build `Map<eventType, Set<workflowId>>` from every trigger.
- **`sync()`** — diff the recomputed union against the currently-registered handlers:
  - *Newly referenced type* → emit `handler_declare` (Worker→Main IPC) for `wft_<T>`.
  - *Type dropped to zero* → loopback `POST /api/events/unregister` for `wft_<T>` (the only removal channel — §3.2).
  - *Type still referenced* → no registration change (only the workflowId set updates).
- **Startup** — `recompute()` + `sync()`; this re-subscribes from persisted config with no separate subscription store (NFR-002). 034's `catchUpHandler`/`reevaluateAfterOwnerStarted` then re-dispatch any un-acked pending events of the re-registered types (late-registration catch-up, FR-005b in 034).
- **Runtime add/change** (tool or UI create/modify/delete) — `handlers.js` persists the change to VFS, then calls `recompute()`+`sync()` (FR-005: "without requiring a service restart").
- **Stop/crash — no explicit unregister (FR-010).** The service does **not** delete its handler registrations on stop. 034's *active-set rule* (`dispatch.ts`'s `isHandlerActive`: `ownerRunningCheck(ownerId)` = `serviceRegistry().getService(ownerId)?.state === "running"`, wired in the `ServiceManager` constructor) makes every `workflows` handler **inactive** the moment the service is not running, so `activeHandlersForType` excludes them and **no dispatch to a stopped service can occur** — which is exactly the spec's "no stale dispatch." `ServiceManager`'s stop/crash paths additionally call `reevaluateAllPending()`. This is "deactivated," not literally "removed," but achieves the spec's FR-010 intent; it is the correct behavior because it also preserves the user's per-handler `enabled` state across a stop/start cycle (an explicit unregister would reset it on re-declare). (See ADR-2 for why the service never self-unregisters on stop.)

**Why per-type (not per-(workflow,trigger)) subscriptions** — ADR-2.

#### 3.4.3 Tool surface (FR-008) — no new tools

The trigger is part of the workflow config (not a separate entity), so the baseline's 12-tool surface already covers FR-008; this increment only *extends schemas*, it adds no 13th tool:

| Tool (existing) | This increment |
|---|---|
| `workflow_create` | Accepts `triggers: [{id, eventType}]` in the config body; validates (FR-014); persists; `sync()`. |
| `workflow_modify` | Accepts `triggers` in the JSON-merge patch (add/edit/remove); validates; persists; `sync()`. This is what the app's "Add/Edit/Remove trigger" buttons call — the mockup's `workflow_trigger_add` hint is a **UI affordance name**, not a new tool. |
| `workflow_read` | Returns the full workflow JSON incl. `triggers` — satisfies "a read/inspection tool MUST return a workflow's triggers" (FR-008). |
| `workflow_run_list` | Each run gains `startSource` (`event`\|`manual`) + `eventType` when event-triggered (FR-011) — so event vs. manual is distinguishable in the list (US4 AC3). |
| `workflow_run_get` | Returns `trigger: {eventId, eventType}` (FR-007) + `input.event` (the full payload, FR-006) + the seeded `triggered` first log entry. |

`workflow_status` (live) surfaces the running run's `startSource`/`eventType` too (the live-run ⚡ badge in the mockup). The other tools (`run`, `cancel`, `delete`, `export`, `validate`) are unchanged in behavior; `workflow_run` on an event-triggerable workflow is still a *manual* start (`startSource:"manual"`), independent of its triggers.

### 3.5 The namespace-ownership grant (the one manifest decision)

034 enforces a **register-time namespace-ownership gate** (`api.ts` → `ownsNamespace`, `src/lib/events/types.ts`): a service with `ownerId = workflows` may only register handlers for event types under its owned root `com.bos.workflows.*` **or** under a namespace in its static `eventNamespaces` grant list (`service.json`). The grant is static (declared in the manifest, never self-granted at runtime — 034 FR-023).

This is the crux the spec's assumptions gloss over: the spec's example triggers (`com.bos.assistant.task.done`, `com.bos.events.file.changed`, `com.bos.assistant.message.received`) are **not** under `com.bos.workflows.*`, so without a grant the service could subscribe to *none* of them. (These examples are also illustrative, not real emitters — §7 Open Question #2.)

**Decision: the item's `service.json` declares `eventNamespaces: ["com.bos.*"]`** (ADR-1). Justification: the workflow service's *entire purpose* is to react to arbitrary BOS events, so its subscription rights must span the BOS namespace, not a single emitter. `com.bos.*` is a valid grant entry (accepted by `ownsNamespace`'s prefix match — verified: `baseOf("com.bos.*")` = `com.bos`, and any `com.bos.…` type starts with `com.bos.`), and `manifestValidator.ts` does not reject it. A **scoped alternative** (enumerate only the namespaces the user cares about, e.g. `["com.bos.gsuite.*","com.bos.gitops.*"]`) is safer but requires the user to know and list namespaces and would silently prevent triggers on anything else; it is offered as the reviewer's fallback if the broad grant is deemed too wide. The grant breadth is a *config* decision on the item's own manifest (a soft integrity guard under 034's single-container trust model, not a network boundary), trivially narrowable without code change.

---

## 4. Concrete File/Module Plan

All under `data/user-apps/items/workflows/` (installed via `app_build`; lands on the item's own `app-candidate` branch — **no** BOS feature branch, no `dev_delegate`). No `src/` files.

| Path | Action | Purpose |
|---|---|---|
| `services/service.json` | **Modify** | Add `"eventNamespaces": ["com.bos.*"]` (the subscription grant, §3.5/ADR-1). Everything else unchanged (`deploymentMode:"tools"`, entry, `configSchema` with `port:0`). |
| `services/index.js` | **Modify** | On boot (after `initialized`): `triggers.recompute()` + `triggers.sync()` to re-subscribe from persisted config. Add an `event_dispatch` message handler → `triggers.onDispatch(record)`. Wire tool handlers to call `triggers.sync()` after mutations. |
| `services/triggers.js` | **Create** | Trigger engine: `subscriptionIndex` (Map<eventType, Set<workflowId>>), `recompute()`, `sync()` (declare new / unregister dropped), `onDispatch()` (match + fan-out + `startRun` + ack), `handlerId` scheme (`wft_<type>`), local event-type validation (FR-014). |
| `services/events.js` | **Create** | Item-local loopback event-API client: `unregister(handlerId, ownerId)` → `POST /api/events/unregister`; `ack(eventId, {handlerId, result, callerId, callId})` → `POST /api/events/:id/ack`. Reuses baseline origin resolution (ADR-6). (The service cannot import BOS's `src/lib/events/loopback.ts`.) |
| `services/engine/store.js` | **Modify** | Persist workflow JSON incl. `triggers` array; validate trigger shape (FR-014) on create/modify. |
| `services/runs.js` | **Modify** | Run entity: add `startSource` (`event`\|`manual`), `trigger` (`{eventId, eventType}`), `input.event` (full event), and seed the run's log with a first `triggered` entry (`{type:"triggered", eventType, eventId}`) for event runs. `run_list`/`run_get` surface provenance (FR-011/FR-007). |
| `services/handlers.js` | **Modify** | `workflow_create`/`workflow_modify` accept `triggers`; `workflow_read` returns them; after any mutation persist to VFS then `triggers.sync()`. `workflow_run` on an eventable workflow = manual start. |
| `services/tools.js` | **Modify** | Extend create/modify/read/run_list/run_get(+status) schemas for `triggers` + provenance fields. No new tools (§3.4.3). |
| `app/src/main.tsx` (+ `TriggersPanel`, run-row ⚡, triggered-by banner, etc.) | **Modify** | Triggers panel in the detail view (list/add/edit/remove, empty state, "listening" indicator); ⚡ badge + `⌁ <type>` on run-history rows and the live-run header; triggered-by banner in the historical replay panel. All mutations go through `window.__bos` → the service tools (`workflow_modify`/`workflow_read`). |
| `config/workflows.json` | unchanged | `{ port: 0, host: "127.0.0.1" }` (port stays `0` — every worker service binds a real port; no external client connects for triggers). |

**`builtin-app` target shape: does not apply — omitted** (per the hard rule: omit non-applicable target shapes rather than list-and-dismiss).

---

## 5. Integration Points

Existing BOS mechanisms this design **calls into** but does **not** create/modify — dependencies, not deliverables. Each cited to the real route/file that makes it real.

| Mechanism | Real route/file | Used for |
|---|---|---|
| **034 event kernel** (emit/durable record/dispatch) | `src/lib/events/kernel.ts`, `dispatch.ts`, `store.ts`, `types.ts` | The pub/sub broker the service subscribes to; publishes durable `EventRecord`s; fans `event_dispatch` out to the service's handlers. In-process, unchanged. |
| **Worker-IPC `handler_declare`** (subscribe) | `src/core/service/types.ts` (`HandlerDeclarePayload`, `WorkerToMainMessage`), `src/core/service/ServiceManager.ts` (the `handler_declare` case) | The service declares a headless handler per subscribed event type. |
| **Worker-IPC `event_dispatch`** (invocation) | `src/core/service/types.ts` (`EventDispatchPayload`), `src/core/service/workerIpc.ts` (`sendEventDispatch`), `src/core/service/ServiceManager.ts` (`setServiceInvoker` wiring) | The kernel invokes the service's handler, delivering the full `EventRecord`. |
| **034 active-set / offline-inactivity rule** | `src/lib/events/dispatch.ts` (`isHandlerActive`, `ownerRunningCheck`), `src/core/service/ServiceManager.ts` (`reevaluateAllPending` on stop/crash) | FR-010: handlers are inactive (no dispatch) while the service is stopped — no stale dispatch. |
| **Loopback event API — unregister + ack** | `src/app/api/events/unregister/route.ts`, `src/app/api/events/[id]/ack/route.ts`, `src/lib/events/api.ts` (the single public API), `src/lib/events/loopback.ts` (the documented worker-thread transport shape) | Runtime unsubscribe (the only removal channel) and the handler ack (FR-007/011 settlement). |
| **039 service-tool bridge + tool IPC** (baseline) | `src/lib/agent/service-tool-bridge.ts`, `src/core/service/ServiceManager.ts`, `src/core/service/workerIpc.ts`, `serviceToolTypes.ts` | The 12 workflow tools (create/modify/read/run_list/run_get/…) the app + assistant use to author/read triggers and inspect provenance. Unchanged surface, extended schemas. |
| **Real VFS bridge** (baseline) | `src/app/api/fs/route.ts`, `src/os/vfs.ts` | Persist triggers inside the workflow JSON + run provenance under `/Workflows/` and `/Workflows/.runs/` (NFR-003, ADR-2 of baseline). |
| **Sub-agent delegate route** (baseline, node execution) | `src/app/api/subagents/delegate/route.ts` | How the triggered *run* executes its nodes (unchanged from baseline ADR-1); the event payload rides in the run's input and is referenced by nodes. |
| **`window.__bos` broker + app capabilities** | `src/lib/apps/store.ts` (`setAppCapabilities`) | The app facet reaches the service tools through the SDK broker with granted capabilities (baseline; `services:read`/`fs:read` grants post-install). |
| **Event Viewer Configuration (034)** | `src/apps/event-viewer/`, `GET/POST /api/events/handlers` | The user can enable/disable the `workflows` service's per-type handlers (the coarseness noted in §3.4.1). Unchanged. |

---

## 6. Key ADRs

### ADR-1 — Subscribe via a broad `com.bos.*` `eventNamespaces` grant (the one manifest decision)
- **Context**: 034's register-time gate (`ownsNamespace`, `src/lib/events/api.ts`) lets a service subscribe only to `com.bos.<ownerId>.*` plus its static `eventNamespaces` grants. The `workflows` service's owned root is `com.bos.workflows.*` — effectively empty of real emitters. The spec's whole premise ("trigger on a specific event from the event system") requires subscribing to types outside the owned root.
- **Options**: (a) grant `["com.bos.*"]` (whole BOS namespace); (b) grant a scoped list (e.g. `["com.bos.gsuite.*","com.bos.gitops.*"]`); (c) no grant (owned root only).
- **Decision**: **(a)** — the item's `service.json` declares `eventNamespaces: ["com.bos.*"]`. Rationale: a general-purpose event-trigger sink must react to *arbitrary* BOS events; scoping (b) forces the user to know/enumerate namespaces and silently blocks the rest; (c) makes the feature nearly useless out of the box. The grant is the item's *own* manifest (a soft integrity guard under 034's single-container trust model, trivially narrowable), not a BOS change.
- **Consequences**: Every well-formed `com.bos.*` event type is a valid trigger (so FR-014's "well-formed" check is sufficient in practice). **Risk (flagged, §7 OQ#1): broad grant breadth.** If the reviewer deems `com.bos.*` too wide, the fallback is (b) scoped grants, with the service validating triggers against the actual grant list and erroring clearly on types outside it. Either way the service validates locally before registering (FR-014).

### ADR-2 — One headless handler per distinct event type (not per (workflow, trigger)); subscribe over IPC, unsubscribe over loopback
- **Context**: The spec says the service subscribes to the *union* of referenced types. A headless handler's match key is its `eventType`. Choices: (a) one handler per distinct event type; (b) one handler per (workflow, trigger). Also: subscribe channel — `handler_declare` IPC vs. loopback `register`; and the only removal channel is loopback `unregister` (no `handler_undeclare` exists).
- **Decision**: **(a)** per-type, with `handlerId = "wft_" + eventType`. Subscribe = `handler_declare` (Worker→Main IPC, the 034 ADR-3 runtime path, reusing the `parentPort` channel the service already uses for `tool_declare`). Unsubscribe = loopback `POST /api/events/unregister` (the only removal channel).
- **Why**: (a) matches the spec's "union" language, minimizes registrations + per-handler FIFO queues, and makes fan-out a single handler resolving all workflows for a type. (b) would create N handlers for one type (N workflows), each with its own queue, for no benefit. The subscribe/unsubscribe asymmetry is forced by the actual protocol (verified: `WorkerToMainMessage` has `handler_declare` but no `handler_undeclare`) — the spec's "e.g. handler_declare/handler_undeclare" is corrected here.
- **Consequences**: The 034 `enabled` toggle is per-type (coarseness noted §3.4.1). Runtime add/remove is two channels (IPC + loopback) — both already exist and are grounded. Loopback `register` is a fully-equivalent alternative to `handler_declare` for subscribe (both funnel to `eventsApi.register`); `handler_declare` is preferred because it is the 034-intended runtime path, matches the spec's framing, and its ServiceManager handler runs `reevaluateAfterOwnerStarted` for catch-up.

### ADR-3 — The handler acks *after attempting* to start the run(s), not after run completion (fire-and-poll)
- **Context**: 034's dispatch waits for the handler ack (or the 30s timeout) and retries a failing handler up to 3× before `permanently_failed`. A workflow run can take minutes. If the handler waited for run completion to ack, the event would stay `pending` for the whole run and a failing run would trigger handler retries — conflating run failure with handler failure.
- **Decision**: The handler attempts `startRun` for each matching workflow (fire-and-poll — returns a `runId` in <1s per baseline ADR-8), records per-workflow outcomes, then **acks immediately** with `result: {started:[…], failed:[…]}`. The runs continue asynchronously; the event is processed.
- **Consequences**: Satisfies FR-012 (fast response; the long run is deferred) and NFR-004 (bounded work per event, no unbounded fan-out). A run that later fails is a *run* failure (inspectable in the run log), not a handler failure — the event is not left stuck retrying. If `startRun` itself throws (service-level, e.g. a VFS write failure), the handler logs it (NFR-001) and still acks (with the failure in `result`), so the event is processed rather than retried. Re-entrancy (FR-013, always-start-a-new-run) is honored because each matching event starts fresh runs with no coalescing.

### ADR-4 — The event payload is injected as a reserved named run input binding, `input.event`
- **Context**: FR-006 requires the full event (type + JSON body) be available to the run, and defers the injection point (named input binding vs. seeded variable vs. tool-call argument). Nodes execute via the baseline loopback `/api/subagents/delegate`; the run's context is assembled by the worker and passed into node execution.
- **Decision**: **Named input binding.** The run record gains an `input` object; for an event-triggered run the worker seeds `input.event = { id, type, payload, ts, summary, source }` (the full `EventRecord`-derived object). The worker's executor includes the run's `input` when assembling each node's delegate request, so any node can reference the reserved `event` binding. Manual runs simply have no `input.event`.
- **Why over the alternatives**: A seeded variable is effectively the same thing with less structure; a tool-call argument would pollute every tool's schema and wouldn't be visible in the run's provenance. A named run input (a) composes with the baseline run entity, (b) is inspectable on the run for provenance (FR-007), (c) is uniform for all nodes, and (d) keeps the run's `input` as the single documented "what started this and with what" context.
- **Consequences**: FR-004 (an event-triggered run is an ordinary run) holds — it just carries an `input`. The full payload lives in `input.event`; the *provenance header* (`trigger: {eventId, eventType}`) stays small on the run so a 1MB payload isn't duplicated in the log.

### ADR-5 — Run provenance: small header + seeded log entry, surfaced by run_list/run_get + the app ⚡
- **Context**: FR-007 (record triggering event id+type + a start-source marker) and FR-011 (expose event vs. manual in run_list/run_get + app history).
- **Decision**: The run entity adds `startSource: "event" | "manual"` and, when event-triggered, `trigger: { eventId, eventType }` (small — no payload duplication; the payload is in `input.event` per ADR-4). The run's event log is seeded with a first entry `{ type: "triggered", eventType, eventId }` (the mockup's event stream begins `⚡ triggered · <type>`). `workflow_run_list` returns `startSource` (+ `eventType`) per run; `workflow_run_get` returns `trigger` + `input.event` + the seeded entry; the app renders the ⚡ badge + "started by event <type>" banner (mockup §8).
- **Consequences**: Satisfies FR-007/FR-011 and US4. Manual runs are marked `startSource:"manual"` with no `trigger` (US4 AC2). The seeded `triggered` entry makes the provenance visible in the replayed stream, matching the mockup.

---

## 7. Risks / Open Questions

1. **Broad namespace grant (`com.bos.*`) — the headline reconciliation.** The spec's mental model ("a trigger names a specific event type from the event system"; FR-014 requires only "well-formed") silently assumes the service *can* subscribe to any BOS event. 034's gate requires a grant for that; this design ships `eventNamespaces: ["com.bos.*"]` (ADR-1). **Build Studio should confirm the broad grant is acceptable**, or pick scoped grants (ADR-1 fallback) and accept that triggers outside the scoped set are rejected. This is the one place the spec and the 034 model diverge; it does not change the spec's requirements (FR-014's "well-formed" remains the authoring-time check, which under a `com.bos.*` grant is sufficient for all real BOS events) but it changes *what is actually triggerable*.
2. **The spec/mockup example event types are illustrative, not real emitters.** `com.bos.assistant.task.done` appears in BOS source *only* as an example string in the `emit` tool's schema description (`src/lib/assistant/tools/server/events.ts:28`) — it is not emitted by any component. The real emitters I found are `com.bos.gitops.conflict.escalated` (gitops), `com.bos.gsuite.email.received` / `com.bos.gsuite.calendar.event` (GSuite), and legacy-mapped `com.bos.<integration>.<service>.<kind>` types (`src/lib/events/legacy-mapping.ts`). So the mockup's trigger examples are placeholders. The feature is **emitter-agnostic** (it works on any granted BOS type that is actually emitted); the mockup/spec examples should not be read as a catalog of real event types. (No design change needed — flagged so e2e tests emit a *real* or a *test-emitted* event rather than assuming `com.bos.assistant.task.done` fires on its own.)
3. **034 spec/code drift worth a `discrepancies.md` note if implement confirms it.** The 034 *design.md* §3.3/§3.6 describes a `ServiceManager.dispatchEventToWorker(serviceId, payload, {timeoutMs})` helper and references `src/instrumentation.ts` wiring; the *actual* source implements dispatch via `workerIpc.sendEventDispatch` + the `setServiceInvoker` closure (no `dispatchEventToWorker` method exists, and I did not verify the exact `instrumentation.ts` start ordering this session). If implement relies on the design's method name it will not find it. Recommend recording this as a 034 doc/code drift in `discrepancies.md` (not blocking — the real mechanism is unambiguous from `types.ts`/`workerIpc.ts`/`ServiceManager.ts`).
4. **In-memory index vs. out-of-band VFS edits.** The subscription index is authoritative and updated on every tool-mediated mutation, but a workflow JSON edited *directly* in the VFS (outside the tools) would not update the index until the next `sync()`/restart. This is a pre-existing baseline concern (the engine reads workflow definitions from VFS on demand), not trigger-specific; `sync()` on startup and on any tool mutation covers the normal path. If a user hand-edits a workflow's triggers in the VFS, a service restart (or a `workflow_read`-triggered `sync()`) reconciles it. Minor; noted for completeness.
5. **Handler timeout vs. slow `startRun`.** The handler acks after *attempting* (ADR-3), and `startRun` is fire-and-poll (<1s, baseline ADR-8), so the ack lands well within the default 30s handler timeout. If a future change makes `startRun` synchronous/slow, the handler's `timeoutMs` (settable per-declaration) would need raising. Not a concern under the current fire-and-poll contract.
6. **App capability grants.** The app needs the baseline grants (`services:read`, and `fs:read` if it reads `/Workflows` directly) post-install in Settings → Apps — carried from baseline 001 (unchanged by this increment; the app reaches triggers via the service tools through `window.__bos`).

---

## 8. UI Mockup Reference

**Path**: `mockup.html` (same spec directory). Maps onto the app facet (`app/src/main.tsx` + components):

| Mockup element | Component | FR |
|---|---|---|
| **List-view trigger badge** (card footer: `⚡ 3 triggers` / `⚡ manual only`) | List cards read `triggers.length` from `workflow_list`/`workflow_read`. | US3 (inspect) |
| **Triggers panel** in detail view (`#view-detail`): header + count ("3 active"), Add-trigger input (`com.bos.assistant.task.done` placeholder), trigger list rows (icon + mono `eventType` + "listening" dot + hover edit/remove), empty state ("No event triggers… add a trigger to start it from a BOS event"), helper note. | **New** `TriggersPanel` in the detail view. List from `workflow_read`; Add/Edit/Remove → `workflow_modify` (the mockup's `workflow_trigger_add` hint is the affordance name, not a tool — §3.4.3). Persists + `sync()`. | FR-009 (US3) |
| **Run history ⚡** (rows `#6`, `#3` show an amber ⚡ chip + `⌁ <type>`; `#5`, `#4` show `manual`) | Run-history rows render `startSource`/`eventType` from `workflow_run_list`. | FR-011 (US4) |
| **Triggered-by banner** in the historical replay panel (`#history-triggered`: "Started by event `<type>`", shown only for event-triggered runs) | History panel (replayed from `workflow_run_get`) shows the banner when `startSource === "event"`, using `trigger.eventType`. | FR-007/FR-011 |
| **Live-run ⚡** (run-view header "event" chip + `⌁ <type>`; event stream first line `⚡ triggered · <type>`) | Live run view shows `startSource`/`eventType` from `workflow_status`; the seeded `triggered` log entry (ADR-5) is the first stream line. | FR-007/FR-011, US1 |
| **Event stream** (starts with `⚡ triggered · <type>`, then `step.start`/`step.complete`…) | Baseline run event stream + the seeded `triggered` entry. | FR-004 (ordinary run) |

The mockup's **service pill** ("Service running") and **stopped banner** reflect service state — when stopped, the handlers are inactive (no dispatch, FR-010) and runs are refused (baseline FR-007). The ⚡/`⌁` provenance affordances are the visible half of ADR-5.
