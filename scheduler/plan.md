# Implementation Plan: Unified Scheduler Engine

**Feature**: `scheduler`
**Spec Status**: Ready for Implementation
**Target Branch**: `bos/scheduler`

---

## Technical Context

The Scheduler is being reshaped from a **single-purpose "prompt-to-agent" task runner** into a **Unified Job Engine** that owns every scheduled unit of work in BOS: user prompts, internal BOS routines (e.g. Memory Loops fast/slow passes, Curator sweeps), and integration polling adapters (Gmail, Calendar, etc.).

### Shift from "integration polling only" to "universal engine"

Today, scheduling lives in two disconnected places:

| Where | What it schedules | How |
|---|---|---|
| `src/lib/integrations/scheduler/` (`daemon.ts`, `jobs.ts`) | Integration polling adapters (Gmail, etc.) | Per-service `PollConfig` embedded in each integration's `state.json` (`state.services[svcId].poll`); daemon ticks every N seconds |
| Ad-hoc / voluntary | Memory reflection (via `skill_reflect`), Curator sweeps | Triggered manually or on user action; no persistence, no history |

There is no first-class user-facing task scheduling — the original scheduler spec (v1) proposed a *second, parallel* daemon for user prompts on top of the integration daemon. That's now rejected as unnecessary duplication.

**The unified engine** replaces both:

- A single loop, single persistence store, single history log.
- Three **job categories** distinguishing *ownership*: `system` (BOS internals, cannot be deleted by user), `user` (user-created, fully editable), `integration` (owned by an integration adapter, partially editable — the adapter controls what).
- Three **handler kinds** distinguishing *what runs*: `prompt` (send a prompt to a named agent), `internal` (call a registered BOS function — e.g. `memory-loops.fast`), `integration` (dispatch to a registered integration polling adapter).

The category and handler axes are orthogonal: a `system` job can be `internal` (memory-loops) or `prompt` (a system-managed nightly report); a `user` job is almost always `prompt`; an `integration` job is always `integration` handler.

### Why unify

- **One place to see everything scheduled.** Users, agents, and developers all look at the same list; the same tick loop and failure isolation cover every scheduled unit of work.
- **Memory Loops (021) can register as jobs** without inventing another scheduler.
- **Integration polling** stops being a special-case module and becomes a handler registration on the same engine — one migration, one code path afterwards.
- **Audit / observability** becomes uniform: one run-history log format, one settings UI, one MCP tool surface.

---

## Architecture

### Engine loop (`src/lib/scheduler/engine.ts`)

```
Startup
  1. Load all jobs from /Documents/System/scheduler-jobs.json
  2. Validate handler references (drop / mark broken any pointing at
     unregistered handlers; log per-job)
  3. Handle past-due jobs per policy (configurable: run-immediately | mark-review | skip)
  4. Start tick loop

Tick (every ~30s, or configured tickIntervalSec)
  for each job where status=active AND nextRunAt<=now:
    - lookup handler via registry (by kind + handlerId)
    - dispatch with job-specific inputs
    - capture RunResult { ok, durationMs, output?, error? }
    - append JSONL entry to /Documents/System/scheduler-history/<jobId>.jsonl
    - advance nextRunAt per schedule (or mark completed for one-time)
    - update failure/backoff counters (integration category retains its
      backoff semantics; see FR-005 in spec)
    - errors are caught per job; the loop never crashes
  persist mutated job records atomically

Shutdown: drain in-flight dispatches with timeout; persist; exit
```

### Handler registry

`src/lib/scheduler/handlers/registry.ts` exposes `register(kind, handlerId, handler)` and `resolve(kind, handlerId)`. Handlers are registered at module init; the engine never hardcodes handler logic.

| Kind | Handler shape | Examples of registrants |
|---|---|---|
| `prompt` | `(job, ctx) => runAgentPrompt(job.agentId, job.prompt)` | Single generic handler; jobs vary by `agentId` + `prompt` |
| `internal` | `(job, ctx) => bosInternalFn(job.inputs)` | `memory-loops.fast`, `memory-loops.slow`, `curator.sweep` |
| `integration` | `(job, ctx) => adapter.poll(job.serviceId, job.extras)` | Gmail, Calendar (each integration registers its handler at boot) |

Registration is **owned by the module that provides the handler** — `src/lib/agent/memory/fast-loop.ts` calls `register('internal', 'memory-loops.fast', ...)`; integration adapters call `register('integration', '<integrationId>.<serviceId>', ...)`.

### Unified persistence (`/Documents/System/scheduler-jobs.json`)

Single JSON file in the VFS Documents tree. Structure:

```jsonc
{
  "version": 1,
  "jobs": [
    {
      "id": "job_01H...",                    // ULID
      "category": "system" | "user" | "integration",
      "name": "Memory: fast loop",
      "handler": { "kind": "internal", "id": "memory-loops.fast" },
      "inputs": { /* handler-specific payload */ },
      "schedule": { "type": "recurring", "interval": 2, "unit": "minute" },
      "status": "active" | "paused" | "completed" | "broken",
      "nextRunAt": "2026-07-05T20:52:00Z",
      "lastRunAt": null,
      "failures": 0,
      "backoff": null,                        // { retryAfter, seconds } — integration category
      "owner": "memory-loops",                // module id for system/integration; "user" for user
      "createdBy": "seed" | "user" | "adapter" | "agent",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

- **Atomic writes**: temp-file + rename via VFS write API. All mutations go through a single `updateJobs(mutator)` fn that read-modify-writes under an in-memory lock.
- **VFS placement (`/Documents/System/`)** is deliberate: the file is inspectable and exportable by the user via the Files app, but the ACL helper (see below) restricts what the UI/agent can edit per category.

### Run history (`/Documents/System/scheduler-history/<jobId>.jsonl`)

One JSONL file per job. Append-only. Each line:
```json
{"ts":"2026-07-05T20:52:00Z","ok":true,"durationMs":412,"output":"…","error":null}
```
Rotation: files capped at N entries (default 200) via head-truncate on write; older lines archived to `<jobId>.jsonl.<yyyymm>` in the same directory. History reads for the UI use range reads (last-K lines).

### Category-based ACL (`getEditableFields`)

Central helper:
```ts
export function getEditableFields(category: JobCategory): (keyof Job)[]
```

- `user` → `['name','inputs','schedule','status']` (full edit)
- `system` → `['schedule','status']` (schedule/pause OK, but name/inputs/handler locked; delete blocked at API layer)
- `integration` → `['schedule','status','inputs.extras']` (matches today's PollConfig editing semantics: interval/enabled/extras)

The UI, the API layer, and the MCP tools all read from this single helper — no drift between agent-callable edits and human-facing edits.

### Handler-kind dispatch details

- **`prompt`**: reuses the existing agent runtime (spec 016 unified-agents). The engine creates or reuses a conversation per the job's `inputs.conversationPolicy` (`new-per-run` | `append`, default `new-per-run`).
- **`internal`**: the handler receives `(job, {logger, abortSignal})` and MUST be non-blocking or respect abort. All internal handlers are registered by their owning module at boot.
- **`integration`**: existing integration adapter API (`poll(serviceId, extras)`) is reused; failure/backoff/error surfacing behavior matches today's daemon (see `src/lib/integrations/scheduler/types.ts`).

---

## Migration Strategy (FR-016)

**One-time migration** of legacy integration polling configs runs on the engine's first startup after this change lands (detected by absence of `/Documents/System/scheduler-jobs.json` OR presence of a `schema.migratedFromLegacy` marker in the file).

**Source of truth (legacy)**: each integration's `state.json` at `<integrations root>/<integrationId>/state.json` carries `state.services[svcId].poll: PollConfig` (from `src/lib/integrations/scheduler/types.ts`). Fields: `enabled`, `intervalSec`, `lastError`, `retryAfter`, `backoffSec`, `failures`, `extras`.

**Steps** (implemented in `src/lib/scheduler/migrate.ts`):

1. **Locate**: enumerate integration state files via existing `paths.ts` helpers.
2. **Enumerate**: for each `state.services[svcId].poll` where `enabled === true`, prepare a `Job`:
   ```
   { category: 'integration',
     name: `${integrationId} · ${svcId}`,
     handler: { kind: 'integration', id: `${integrationId}.${svcId}` },
     inputs: { integrationId, serviceId, extras: poll.extras ?? {} },
     schedule: { type: 'recurring', interval: poll.intervalSec, unit: 'second' },
     status: 'active',
     failures: poll.failures ?? 0,
     backoff: poll.retryAfter ? { retryAfter: poll.retryAfter, seconds: poll.backoffSec ?? 0 } : null,
     owner: integrationId, createdBy: 'adapter', ... }
   ```
   `enabled === false` polls become jobs with `status: 'paused'`.
3. **Write**: create `/Documents/System/scheduler-jobs.json` in a single atomic write containing all migrated jobs plus a `schema.migratedFromLegacy: <ISO ts>` marker.
4. **Deprecate legacy**: leave `state.services[svcId].poll` in place for one release (read-only fallback for rollback safety), but the legacy daemon (`src/lib/integrations/scheduler/daemon.ts`) is deleted and the polling code path now goes through the engine.
5. **Log**: full migration report to central logging (`component: 'scheduler-migration'`) — one entry per integration, plus a summary (`migrated: N, skipped: M`).
6. **Idempotent**: re-invoking migration when the marker exists is a no-op with a single info log line.

**Rollback**: since legacy `PollConfig` is left in place, reverting the engine commit re-enables the old daemon with correct state.

**Failure semantics**: an unreadable/malformed legacy `state.json` is logged and skipped — never blocks migration of other integrations. Migration success does not require every integration to migrate cleanly.

---

## Implementation Phases

### Phase 1 — Engine Core & Persistence

- Types (`Job`, `JobCategory`, `Handler`, `RunResult`, `Schedule`, `ACL`).
- `src/lib/scheduler/engine.ts` — loader, tick loop, dispatch, atomic persistence, shutdown drain.
- `src/lib/scheduler/handlers/registry.ts` — handler registry.
- `src/lib/scheduler/store.ts` — VFS read/write of `/Documents/System/scheduler-jobs.json` (atomic, in-memory lock).
- `src/lib/scheduler/history.ts` — JSONL append + rotation for `/Documents/System/scheduler-history/<jobId>.jsonl`.
- `src/lib/scheduler/schedule.ts` — `calculateNextRun`, schedule validation.
- `src/lib/scheduler/acl.ts` — `getEditableFields(category)` helper.
- Prompt handler registration (single generic implementation).
- Logging integration (component `scheduler`).
- **Deliverable**: engine can be started, ticks with zero jobs cleanly, accepts jobs registered programmatically, persists + logs runs. No UI/API yet.

### Phase 2 — Migration & Integration Adapter

- `src/lib/scheduler/migrate.ts` — legacy-config migration (FR-016).
- Integration handler shim: each integration adapter registers its polling handler with the engine at boot (kind=`integration`).
- Delete `src/lib/integrations/scheduler/daemon.ts` (replaced by engine); keep `types.ts` for `PollConfig` (used by settings UI backing the integration-category jobs) and any code that still needs backoff types.
- Rewire integration Settings UI (interval / enabled / extras editors) to write to the corresponding integration-category job via `getEditableFields('integration')`, not to `state.services[svcId].poll` directly.
- Bootstrapping: engine startup path calls migration first, then loads jobs.
- **Deliverable**: existing integration polling behavior is preserved end-to-end via the engine; no user-visible change to Gmail/etc.

### Phase 3 — UI & API

- `POST/GET/PUT/DELETE /api/scheduler/jobs` (with category-aware ACL enforcement).
- `POST /api/scheduler/jobs/:id/run` — run-now regardless of schedule.
- `GET /api/scheduler/jobs/:id/history?limit=` — history read.
- Scheduler app UI (`src/apps/scheduler/`):
  - Unified list sorted by `nextRunAt` (soonest first), with **category badges** (system / user / integration).
  - Row actions per category (edit disabled for locked fields; delete disabled for `system`).
  - "New task" form → creates `user` category `prompt` job.
  - Detail view: schedule editor + last 20 history entries.
- MCP tools: `createTask`, `listScheduledTasks`, `pauseTask`, `resumeTask`, `deleteTask`, `runTaskNow`, `updateTaskSchedule`, `getTask` — all wired through the same engine + ACL. Agent-created tasks are `user` category by default (agents cannot create `system` or `integration` jobs).
- Assistant flow ("schedule a task") wires through the MCP tools unchanged from spec's User Story 1.
- **Deliverable**: users and agents manage all scheduled work from one place.

### Phase 4 — Polish (deferred, not blocking Memory Loops)

- Past-due policy configuration + UI.
- Cron-expression schedule type.
- Timezone controls per job.
- Task search/filter in UI.
- Comprehensive integration + unit tests.

---

## Dependencies

- **Spec 017 Central Logging**: engine, migration, and every handler MUST log via the central logging facility (`component: 'scheduler'`, `component: 'scheduler-migration'`, `component: 'scheduler.<handlerId>'`). History JSONL is separate from central logs (different retention / read path).
- **VFS (`src/os/vfs.ts`)**: job store and history live under `/Documents/System/`. All writes go through the VFS server API; the engine imports `server-only` code paths only.
- **Spec 016 Unified Agents**: the `prompt` handler dispatches through the unified agent runtime.
- **Spec 011 Per-Agent Capabilities**: MCP scheduler tools are gated per agent using existing capability scoping.
- **Spec 009 Installed Apps**: the Scheduler UI ships as a built-in app under `src/apps/scheduler/`.
- **Spec 021 Memory Loops** (consumer, not dependency): Memory Loops registers two `internal` handlers (`memory-loops.fast`, `memory-loops.slow`) and seeds two system jobs. Memory Loops Phase 2+ starts after Scheduler Phase 1 lands.

---

## Constitution Check

Against `.specify/memory/constitution.md`:

| Principle | Compliance |
|---|---|
| Specs before code | ✅ This plan derives from `spec.md`; implementation delegated to Developer |
| No new npm dependencies | ✅ ULID + JSONL are trivial; VFS is existing infrastructure |
| Atomic writes | ✅ Store + history use temp-file + rename via VFS |
| Failure isolation | ✅ Per-job try/catch; loop never crashes; category-scoped backoff for integrations |
| No breaking user data | ✅ Legacy `PollConfig` left in place for one release for rollback |
| Central logging | ✅ Engine, migration, and each handler emit structured logs |
| Category ACLs | ✅ Single `getEditableFields` helper enforced in UI, API, and MCP layers |

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Migration corrupts / duplicates jobs across restarts | Migration marker in job store; idempotent no-op on re-run |
| Handler registry lookup fails at runtime (module load order) | On load, mark handler-missing jobs as `broken` and skip; log; do not crash |
| Long-running internal handler blocks tick loop | Handlers dispatched non-blocking (fire-and-track); abort signal passed; per-handler timeout (default 5 min) |
| History JSONL grows unbounded | Rotate at N entries (default 200) with monthly archive suffix |
| Agent creates `system` or `integration` job via MCP | Enforced at MCP tool layer: agent-created jobs are always `user` category (categories are not agent-settable) |
| Two engine instances (e.g. dev + prod on same VFS) race the store | Store writes use OS-level advisory lock via VFS API; two-writer detection logs and aborts |

---

## Testing Strategy

### Unit
- `schedule.test.ts`: next-run calc for one-time & recurring across units.
- `store.test.ts`: atomic write, concurrent mutation, corrupted-file recovery.
- `history.test.ts`: JSONL append, rotation, range read.
- `migrate.test.ts`: legacy → job translation matrix (enabled, disabled, backoff, extras).
- `acl.test.ts`: editable-fields matrix.

### Integration
- Engine boot → migration → tick → prompt job dispatched → history recorded.
- Engine boot with legacy `PollConfig` for Gmail: Gmail continues to poll at same interval, backoff preserved.
- Memory-loops internal handler registered → job runs → episode written (validated indirectly via 021's tests).
- Failure isolation: one job's handler throws → other jobs continue; failure recorded in history.
- Category ACL: agent MCP `updateTask` on a `system` job's `name` is rejected.

### Manual
- Create user task via UI → see it in list with `user` badge → runs at scheduled time → history visible.
- Existing Gmail poll continues after upgrade (no user action).
- Memory-loops jobs (once 021 ships) show in list with `system` badge and are non-deletable.
