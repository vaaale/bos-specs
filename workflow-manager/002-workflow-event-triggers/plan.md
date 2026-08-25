# Implementation Plan: Workflow Event Triggers

**Branch**: `bos/002-workflow-event-triggers` | **Date**: 2026-08-25 | **Spec**: `/Specs/user-specs/workflow-manager/002-workflow-event-triggers/spec.md`

**Input**: Feature specification for event-triggered workflows — an incremental scope-add to the installed `workflows` marketplace item.

## Summary

Add **event triggers** to the Workflow Manager: a workflow can declare one or more triggers, each naming an exact BOS event type. The service subscribes to the union of referenced types via the 034 event system's headless-handler runtime-declaration channel; when a matching event is published, the service starts a run, injects the event payload as the run's input context, and records provenance. Triggers are authorable via the existing workflow tools and the app's Triggers panel.

**Technical approach** (detailed in `design.md`): The service gains a trigger engine (`triggers.js`) that maintains an in-memory subscription index (event type → workflow IDs), syncs registrations via `handler_declare` IPC + loopback `unregister`, handles `event_dispatch` by fanning out `startRun` calls and acking over loopback, and extends the run entity with `startSource` + `trigger` + `input.event`. The item's `service.json` gains `eventNamespaces: ["com.bos.*"]` (ADR-1). Zero BOS-source files are modified.

## Technical Context

**Language/Version**: JavaScript (service facet, unbundled worker thread — no TS, no `@/` imports); TypeScript/React (app facet, bundled by esbuild)

**Primary Dependencies**: None new. The service reuses the existing 034 worker-IPC channel (`handler_declare`/`event_dispatch`), the baseline's loopback `/api/fs` bridge, and the loopback event API (`/api/events/unregister`, `/api/events/:id/ack`). The app reuses the existing `window.__bos` SDK broker.

**Storage**: Real VFS `/Workflows/<id>-workflow.json` (workflow JSON gains a `triggers` array) and `/Workflows/.runs/<workflowId>/<runId>.json` (run entity gains `startSource`, `trigger`, `input.event`). All via loopback `/api/fs`. No new storage.

**Testing**: E2E via Playwright (emit a real test event, verify run starts + provenance). Unit-testable: trigger validation, subscription index recompute/sync diffing, run provenance serialization. The service is a worker thread — testable by driving its message interface.

**Target Platform**: BOS (Node.js server, worker-thread service + React app in iframe)

**Project Type**: Marketplace item (service facet + app facet), installed via `app_build`

**Performance Goals**: Handler ack within 30s (034 default); `startRun` is fire-and-poll (<1s to return a `runId`); subscription sync is O(number of distinct event types) and runs on every mutation (typically <5 types, <10 workflows).

**Constraints**:
- The service is unbundled (cannot import `@/` modules) — all BOS interaction via worker IPC or loopback HTTP.
- The item's `service.json` is the only manifest change (`eventNamespaces` grant).
- Port defaults to `0` (OS-assigned); no external client connects to the service port for triggers.
- The event payload may be up to 1MB (034 event size limit); `input.event` stores the full payload once per run.

**Scale/Scope**: Single service (the existing `workflows` service), one new module (`triggers.js`), one new module (`events.js` loopback client), modifications to `store.js`/`runs.js`/`handlers.js`/`tools.js`/`index.js`/`service.json`/app UI. No new BOS-source files.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status |
|---|---|
| **II. Server Authority & SSR Boundary** | ✅ Pass. Service talks to BOS only over worker IPC + loopback HTTP. No `@/` imports in the worker. |
| **IV. Minimize Blast Radius** | ✅ Pass. Zero BOS-source files touched. All changes under `data/user-apps/items/workflows/`. |
| **V. The VFS Is Not the Source** | ✅ Pass. Triggers persist inside workflow JSON at real VFS `/Workflows/`. |
| **VI. Specs & Docs Stay in Sync** | ✅ Pass. The feature consumes documented 034 mechanisms; no doc change required. If implement surfaces 034 code/design drift (the `dispatchEventToWorker` name), record in `discrepancies.md`. |
| **VII. Respect Boundaries** | ✅ Pass. No new external dependency. Reuses existing IPC + loopback patterns. |

No violations. No Complexity Tracking table needed.

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/workflow-manager/002-workflow-event-triggers/
├── spec.md              # Feature specification (complete, clarified)
├── design.md            # Architect's design (complete, reviewed — Ready for plan)
├── mockup.html          # UI mockup (complete, live in preview)
├── plan.md              # This file
└── tasks.md             # Phase 2 output (next step)
```

### Source Code (item root: `data/user-apps/items/workflows/`)

```text
workflows/
├── services/
│   ├── service.json          # MODIFY — add "eventNamespaces": ["com.bos.*"]
│   ├── index.js              # MODIFY — boot: recompute+sync; handle event_dispatch; wire tool→sync
│   ├── triggers.js           # NEW — subscription index, recompute, sync, onDispatch, handlerId, validation
│   ├── events.js             # NEW — loopback event-API client (unregister + ack)
│   ├── engine/store.js       # MODIFY — persist workflow JSON incl. triggers; validate (FR-014)
│   ├── runs.js               # MODIFY — run entity: startSource + trigger + input.event + seeded log entry
│   ├── handlers.js           # MODIFY — create/modify accept triggers; read returns them; sync after mutation
│   └── tools.js              # MODIFY — extend schemas for triggers + provenance (no new tools)
├── app/
│   └── src/
│       ├── main.tsx          # MODIFY — Triggers panel in detail view; ⚡ in run history + live run
│       └── (components)      # TriggersPanel, run-row ⚡ badge, triggered-by banner (mockup §8)
└── config/
    └── workflows.json        # unchanged ({ port: 0, host: "127.0.0.1" })
```

**Structure Decision**: All changes are within the existing item layout (no new top-level directories). Two new service modules (`triggers.js`, `events.js`) are added alongside the existing `services/` files. The app facet gains components within its existing `app/src/` tree. This mirrors the baseline's structure exactly — the trigger engine is a self-contained module that `index.js` bootstraps and `handlers.js` calls into on mutations.

## Design Notes (from `design.md`)

Key decisions relevant to task breakdown:

1. **Wire protocol** (§3.2): Subscribe = `handler_declare` IPC (one per distinct event type, `handlerId = "wft_" + eventType`). Invoke = `event_dispatch` IPC (delivers full `EventRecord`). Ack = loopback `POST /api/events/:id/ack`. Unsubscribe = loopback `POST /api/events/unregister`. No `handler_undeclare` exists.

2. **Payload injection** (ADR-4): `input.event = {id, type, payload, ts, summary, source}` — a reserved named run input binding. The worker's executor includes the run's `input` when assembling node delegate requests.

3. **Subscription lifecycle** (§3.4.2): `recompute()` reads all workflows from VFS → `Map<eventType, Set<workflowId>>`. `sync()` diffs against registered handlers → declare new / unregister dropped. Runs on startup + every mutation. Stop = active-set deactivation (no explicit unregister), which preserves the `enabled` toggle.

4. **Run provenance** (ADR-5): `startSource: "event"|"manual"`, `trigger: {eventId, eventType}`, seeded first log entry `{type:"triggered", eventType, eventId}`. Surfaced by `run_list`/`run_get` + app ⚡.

5. **No new tools** (§3.4.3): The baseline's 12-tool surface already covers trigger authoring via `workflow_create`/`workflow_modify`/`workflow_read` (extended schemas). `workflow_run_list`/`workflow_run_get`/`workflow_status` surface provenance.

6. **Namespace grant** (ADR-1): `eventNamespaces: ["com.bos.*"]` in `service.json`. Local validation (FR-014) checks well-formedness + prefix match before registering.
