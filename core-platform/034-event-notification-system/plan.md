# Implementation Plan: Event & Notification System

**Branch**: `bos/034-event-notification-system` | **Date**: 2026-08-23 | **Spec**: `spec.md`

**Input**: Feature specification from `/Specs/user-specs/core-platform/034-event-notification-system/spec.md` · **Design**: `design.md` (authoritative for architecture — this plan references it, does not repeat it)

## Summary

A pub/sub **event & notification system** as a first-class in-process BOS subsystem: a core event kernel (daemon started at boot) accepts events from any BOS component, stores them persistently with indefinite retention, fan-out dispatches to registered **headless handlers** (at-least-once, per-handler FIFO, retry + backoff), and tracks a two-axis state machine (processing: pending→processed; read: unread→read). A built-in **Event Viewer** app and a repurposed topbar **bell** give users visibility; **UI handlers** (app-declared) make events actionable on click. The existing GSuite/Telegram notification inbox is migrated onto this system.

**Technical approach** (from `design.md`): in-process daemon (ADR-1, sibling to the scheduler daemon), one public API over three transports (ADR-2), runtime-declared headless handlers over worker IPC (ADR-3), file-based storage — immutable per-month JSONL shards + per-month mutable state + warm in-memory index flushed on checkpoint (ADR-4, the M1 fix), hand-rolled list virtualization (ADR-5), bell = unread (ADR-6).

## Technical Context

**Language/Version**: TypeScript (BOS toolchain), Node 20+ runtime (Next.js server process + worker threads)

**Primary Dependencies**: none new (Constitution VII). Reuses: `src/os/atomic-write.ts` (`writeFileAtomic`), existing NDJSON streaming route pattern, worker-IPC channel, `launch()` window API, `serverTool` registry. *Escape hatch only if profiling fails: an embedded DB (ADR-4) — a dependency change requiring explicit approval.*

**Storage**: Files under `data/events/` (gitignored, per Constitution V): per-month JSONL shards (immutable bodies) + per-month state files (mutable) + `index.json` (unbounded projection, warm in memory, checkpointed) + `handlers.json` + `preferences.json`.

**Testing**: Unit — Node test runner (`node --test` / the repo's existing unit convention) over the kernel, dispatch engine, state machine, preferences, sequence assignment. E2E — Playwright (`e2e/034-event-notification-system.spec.ts`). **All tests self-cleaning** (FR-029/030: teardown removes every event, handler registration, preference, and UI state the test created).

**Target Platform**: BOS server process (Node) + browser (Event Viewer) + worker-thread services (emitters/handlers)

**Project Type**: bos-core feature — in-process daemon + API routes + built-in app + topbar component

**Performance Goals** (from spec NFRs): emit durable-record <100ms (NFR-001); initial list render <500ms at 100k events (NFR-002); 100k+ events without degradation (NFR-003); UI-handler resolution <100ms (NFR-004); headless handler begin <500ms post-emit (SC-006); unread-count/list updates ≤1s (NFR-008/009); 60fps list at 100k (SC-002).

**Constraints**: no `package.json`/lockfile change; no new network port; single-container trust model (ack ownership and namespace grants are integrity checks, not security boundaries); payloads ≤1MB; type namespaces ≤256 chars, dot-separated; handlers must be idempotent (at-least-once).

**Scale/Scope**: 100k+ events indefinite; N handlers per type (fan-out concurrent across handlers, sequential within a handler); 3 transports; ~7 new kernel modules + 9 routes + 1 built-in app + 1 component + integration seams.

## Constitution Check

*GATE: Must pass before research. Re-checked after design.*

| Principle | Status | Notes |
|---|---|---|
| **I. Spec-Driven** | ✅ | Plan for an agreed `spec.md`; design step completed with review + one revision round. |
| **II. Server Authority & SSR Boundary** | ✅ | Kernel is server-only (`import "server-only"`); shared types framework-free in `src/lib/events/types.ts`; viewer/bell communicate over `fetch`/NDJSON only. |
| **III. Always Delegate; Claude Codes** | ✅ | Implementation via `dev_delegate` on `bos/034-event-notification-system`. |
| **IV. Minimize Blast Radius** | ✅ | All changes on the feature branch; candidate self-tests (Playwright) before promotion. |
| **V. VFS Is Not the Source** | ✅ | Runtime state as files under `data/events/` (gitignored); spec/docs edited via spec tools; source only via Developer. |
| **VI. Specs & Docs Stay in Sync** | ✅ | FR-026 docs (developer + user) are in the file plan and delegation brief; drift → `discrepancies.md`. |
| **VII. Respect Boundaries** | ✅ (default path) | No dependency/lockfile/build-config change in the default design. Two items flagged in Complexity Tracking below. |

**Gates pass.** Re-check post-Phase-0/1: still passing (no dependency introduced; `data-model.md`/`contracts/` introduce no new mechanisms).

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/core-platform/034-event-notification-system/
├── spec.md              # Agreed feature specification
├── design.md            # Architecture (C4, ADRs, risks) — authoritative
├── mockup.html          # Binding UI contract (self-contained, interactive)
├── plan.md              # This file
├── research.md          # Phase 0 — open questions resolved
├── data-model.md        # Phase 1 — entities, state transitions, storage
├── contracts/
│   └── event-api.md     # Phase 1 — the public event API (one contract, 3 transports)
└── tasks.md             # NEXT step (tasks command — not created here)
```

### Source Code (repository root) — created/modified by the Developer at `implement`

```text
src/
├── lib/events/                        # NEW — the event kernel (all server-only except types.ts)
│   ├── types.ts                       # framework-free shared types (EventRecord, EventState, ...)
│   ├── store.ts                       # persistence: shards + state files + warm index + mutex
│   ├── kernel.ts                      # globalThis singleton daemon; public ops; boot re-dispatch
│   ├── dispatch.ts                    # fan-out, per-handler FIFO, retry/backoff, re-dispatch
│   ├── stream.ts                      # in-memory state-change stream (NDJSON source)
│   ├── api.ts                         # THE public event API (single contract facade)
│   └── migrate-integrations.ts        # one-time idempotent legacy-inbox migration
├── app/api/events/                    # NEW — thin HTTP routes (delegates to api.ts)
│   ├── route.ts                       #   POST emit · GET query
│   ├── stream/route.ts                #   GET NDJSON (replay-then-tail, ?since)
│   ├── [id]/route.ts                  #   GET event (body + state + history)
│   ├── [id]/ack/route.ts              #   POST ack (ownership-validated)
│   ├── [id]/read/route.ts             #   POST mark-read (?all=1)
│   ├── register/route.ts              #   POST register handler
│   ├── unregister/route.ts            #   POST unregister handler
│   ├── preference/route.ts            #   POST set-preference
│   ├── count/route.ts                 #   GET unread count
│   └── handlers/route.ts              #   GET registry · POST enable/disable
├── assistant/tools/server/events.ts   # NEW — agent tools (serverTool, in-process, no HTTP hop)
├── apps/event-viewer/                 # NEW — built-in app (auto-discovered, no registry edit)
│   ├── manifest.ts                    #   id "event-viewer", icon Bell, singleton
│   └── index.tsx                      #   Events + Configuration tabs
│       # internal: EventsList (windowed), EventDetail (generic view), HandlerDialog, ConfigPanel
├── components/desktop/
│   └── EventBell.tsx                  # NEW — replaces IntegrationsBadge
├── core/service/
│   ├── types.ts                       # MOD — event_dispatch / handler_declare IPC types
│   ├── workerIpc.ts                   # MOD — the two new message types (callId-keyed)
│   └── ServiceManager.ts              # MOD — dispatchEventToWorker(); stop/crash → handlers inactive
├── os/types.ts                        # MOD — AppManifest.eventHandlers? + eventNamespaces?
├── lib/apps/store.ts                  # MOD — surface installed apps' eventHandlers into registry
├── lib/integrations/notifications/    # MOD → RETIRE — emitNotification call sites re-pointed;
│                                      #   store + route removed post-migration
├── app/api/integrations/.../poll, webhook/test, webhooks routes  # MOD — emit → api.emit
├── lib/integrations/scheduler/jobs.ts # MOD — emit → api.emit
└── lib/integrations/services/telegram/notification-handler.ts    # MOD — emit → api.emit

docs/
├── dev/events/                        # NEW — API ref, handler registration, idempotency,
│                                      #   retry/timeout, namespaces, payload limits, examples
├── usage/ (events section)            # NEW — viewing, processing status, config, history
└── (architecture-overview + extending-bos "Add an event handler" recipe entries)

tests/                                 # NEW — unit (self-cleaning): kernel, dispatch, state machine,
                                       #   preferences, sequence
e2e/
└── 034-event-notification-system.spec.ts   # NEW — Playwright (self-cleaning)

data/events/                           # RUNTIME (gitignored) — shards, state, index, handlers, preferences
```

**Structure Decision**: mirrors the design's Component layer exactly. Kernel modules mirror the scheduler-daemon/notifications-store precedents the design cites; the built-in app follows the self-describing `src/apps/<id>/` convention (auto-discovered); routes follow the existing API-route pattern. No new project, no new build target.

## Design Notes (references, not repeats)

- **Architecture & ADRs**: see `design.md` §3 (C4), §6 (ADRs 1–6), §5 (integration points).
- **Open questions resolved during this plan**: see `research.md` — ack transport (loopback HTTP, single path), worker port discovery, checkpoint throttling, marketplace param delivery, GSuite email handler, migration ordering.
- **Data model & state transitions**: see `data-model.md`.
- **The public API contract** (the one surface all transports/agents/apps share): see `contracts/event-api.md`.
- **Testing strategy** (FR-029/030): unit tests exercise the kernel in isolation with a temp `data/events` dir and assert cleanup; e2e drives the real UI flows and reverts state in `afterEach`. Both suites are self-cleaning so they never pollute a live store and can run in any order.

## Complexity Tracking

> Filled because two design choices warrant explicit justification under Constitution VII / Governance.

| Item | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| **Dispatch engine** (`dispatch.ts`: fan-out + per-handler FIFO + retry/backoff + at-least-once re-dispatch + two-axis state machine) is the single most complex module | Directly required by FR-004/005/007/008 (the spec's core semantics) | A simpler fire-and-forget dispatcher cannot satisfy at-least-once, retries, or the pending→processed completion rule; it is isolated in one module and fully unit-tested so complexity is bounded and testable |
| **Hand-rolled list virtualization** instead of `@tanstack/react-virtual` (ADR-5) | FR-025 + SC-002 (60fps at 100k) require windowing; a dependency would violate VII's no-lockfile-change default | The list has uniform row height, so a small fixed-row windowing component suffices; the dependency remains a documented fallback only if UX testing shows it's insufficient |

*Note: the embedded-DB escape hatch (ADR-4) is deliberately NOT the default and would itself be a VII violation requiring explicit approval if ever exercised.*
