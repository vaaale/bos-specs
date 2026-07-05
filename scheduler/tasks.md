# Tasks: Unified Scheduler Engine

**Feature**: `scheduler`
**Status**: Ready for Implementation
**Branch**: `bos/scheduler`

Each task lists the target file(s), what to build, and the acceptance signal. Phase 1 is the prerequisite for spec 021 (Memory Loops) Phase 2+.

---

## Phase 1 — Engine Core & Persistence

### Task 1.1: Define core types
- **File**: `src/lib/scheduler/types.ts`
- Define:
  - `JobCategory = 'system' | 'user' | 'integration'`
  - `HandlerKind = 'prompt' | 'internal' | 'integration'`
  - `Schedule = OneTimeSchedule | RecurringSchedule` (recurring supports `unit: 'second' | 'minute' | 'hour' | 'day' | 'week'` — `second` needed to preserve legacy poll intervals down to 30s)
  - `Job` (fields per plan §Unified persistence: `id`, `category`, `name`, `handler: { kind, id }`, `inputs`, `schedule`, `status`, `nextRunAt`, `lastRunAt`, `failures`, `backoff?`, `owner`, `createdBy`, `createdAt`, `updatedAt`)
  - `RunResult = { ok: boolean; durationMs: number; output?: string; error?: string }`
  - `HandlerContext = { logger, abortSignal, now }`
  - `Handler = (job: Job, ctx: HandlerContext) => Promise<RunResult>`
- **Acceptance**: `npx tsc --noEmit` clean; types imported by remaining Phase 1 files.

### Task 1.2: Handler registry
- **File**: `src/lib/scheduler/handlers/registry.ts`
- Export `register(kind, id, handler)`, `resolve(kind, id): Handler | null`, `list(): { kind, id }[]`.
- Registry is process-global (module-level Map); no dynamic unregister needed for v1.
- Log a warning when `resolve` returns null (handler missing).
- **Acceptance**: unit test verifies register/resolve/list; missing handler logs and returns null.

### Task 1.3: Persistence store (VFS-backed)
- **File**: `src/lib/scheduler/store.ts`
- Path constant: `SCHEDULER_JOBS_PATH = '/Documents/System/scheduler-jobs.json'`.
- Functions:
  - `loadJobs(): Promise<Job[]>` — returns `[]` and no-throws on missing file; parses `{ version, jobs, schema? }`.
  - `saveJobs(jobs: Job[], meta?: SchemaMeta): Promise<void>` — atomic write via VFS temp-file + rename.
  - `updateJobs(mutator: (jobs: Job[]) => Job[]): Promise<Job[]>` — in-memory async lock (single promise chain); read → mutate → save; returns final list.
- On corrupted file: log error, preserve as `.corrupt-<ts>.json`, return empty list. Do not throw.
- **Acceptance**: unit tests cover missing file, corrupted file, concurrent `updateJobs` calls (serial).

### Task 1.4: Run-history writer
- **File**: `src/lib/scheduler/history.ts`
- Path fn: `historyPath(jobId) => '/Documents/System/scheduler-history/${jobId}.jsonl'`.
- Functions:
  - `appendRun(jobId: string, entry: RunResult & { ts: string }): Promise<void>` — append one JSONL line.
  - `readRuns(jobId: string, limit: number = 20): Promise<RunEntry[]>` — read last-N lines (tail-first).
  - `rotate(jobId, maxEntries = 200): Promise<void>` — head-truncate + archive to `${jobId}.jsonl.<yyyymm>` when line count exceeds cap; called opportunistically from `appendRun` every N appends.
- **Acceptance**: unit tests cover append + tail-read; rotation preserves last N lines and moves the excess into the monthly archive file.

### Task 1.5: Schedule engine
- **File**: `src/lib/scheduler/schedule.ts`
- `calculateNextRun(schedule, lastExecution?, now?): Date`.
- `validateSchedule(schedule): void | throws` — descriptive errors for invalid interval / unit / datetime.
- Supported units: `second`, `minute`, `hour`, `day`, `week`.
- All calculations use UTC internally.
- **Acceptance**: unit tests cover one-time, recurring across every unit, invalid inputs.

### Task 1.6: Category-based ACL helper
- **File**: `src/lib/scheduler/acl.ts`
- Export `getEditableFields(category: JobCategory): readonly string[]` per plan §Category-based ACL.
- Export `canDelete(category): boolean` (`false` for `system`; `true` otherwise).
- Export `applyEdits(job: Job, patch: Partial<Job>): Job` — enforces ACL, throws on disallowed field.
- **Acceptance**: unit tests cover the matrix (`system`/`user`/`integration` × each editable/locked field, including nested `inputs.extras`).

### Task 1.7: Engine loop
- **File**: `src/lib/scheduler/engine.ts`
- `class SchedulerEngine`:
  - `async start(): Promise<void>` — load jobs, validate handler references (mark missing as `broken`, log), handle past-due per config, kick off tick interval.
  - `async stop(): Promise<void>` — clear interval, await in-flight dispatches with timeout, persist, log shutdown.
  - `private async tick()` — for each `active` job with `nextRunAt <= now`: resolve handler, dispatch, capture RunResult, append history, advance `nextRunAt` (or `status = completed` for one-time), persist. Errors are caught per job.
  - `async runNow(jobId): Promise<RunResult>` — invoked by API/MCP; dispatches immediately without waiting for `nextRunAt`; still records history.
- Default tick interval: 30 seconds (configurable via `scheduler.tickIntervalSec`).
- Log every tick summary (`processed: N, ok: M, errors: K`) at debug; log per-job errors at error.
- **Acceptance**: unit tests with a fake clock cover: due job runs, non-due job skipped, one-time completes, recurring advances, handler error is isolated & logged, missing handler marks job broken.

### Task 1.8: Prompt handler
- **File**: `src/lib/scheduler/handlers/prompt.ts`
- Register `('prompt', 'agent.run')` at module load.
- Reads `job.inputs.agentId`, `job.inputs.prompt`, `job.inputs.conversationPolicy` (`'new-per-run' | 'append'`, default `new-per-run`).
- Dispatches via existing unified-agent runtime (spec 016).
- **Acceptance**: integration test — create a prompt job, engine ticks, agent runtime is invoked with correct args, output captured in history.

### Task 1.9: Boot wiring
- **File**: existing app boot (e.g. `src/lib/boot.ts` or wherever the integrations daemon is currently started).
- Replace integrations-scheduler daemon startup with `SchedulerEngine.start()`. Migration runs before start (Task 2.1).
- Wire `stop()` into process shutdown hooks.
- **Acceptance**: server boot succeeds with zero jobs; engine visible in central logs; no dangling references to the deleted integrations daemon.

### Task 1.10: Central logging integration
- **File**: `src/lib/scheduler/logger.ts` (thin wrapper), used by engine/store/history/handlers.
- All entries use `component: 'scheduler'` (`.migration`, `.handler.<id>` sub-components).
- **Acceptance**: entries visible in Settings → Logs filtered by `scheduler`.

---

## Phase 2 — Migration & Integration Adapter

### Task 2.1: Legacy-config migration (FR-016)
- **File**: `src/lib/scheduler/migrate.ts`
- Exports `runMigration(): Promise<MigrationReport>`.
- Steps per plan §Migration Strategy:
  1. Read job store — if `schema.migratedFromLegacy` present, return `{ skipped: true }` after one info log.
  2. Enumerate integrations via `src/lib/integrations/paths.ts`.
  3. For each `state.services[svcId].poll`, translate to `Job` (see plan for field mapping). Disabled polls become `paused` jobs; retryAfter/backoffSec preserved.
  4. Atomically write the resulting job list with `schema.migratedFromLegacy = <ISO ts>`.
  5. Log full report to central logging (per-integration + summary).
- Handles malformed / unreadable `state.json` per integration: log error, skip that integration, continue.
- Called from `Task 1.9` boot wiring **before** `engine.start()`.
- **Acceptance**: unit test builds a fake integrations root with 3 services (enabled, disabled, in-backoff) → migration produces 3 jobs with correct fields, marker set; second run is no-op.

### Task 2.2: Integration handler adapter
- **File**: `src/lib/scheduler/handlers/integration.ts`
- On module load, iterate registered integrations (existing integration registry `src/lib/integrations/registry.ts`) and for each `serviceId` register `('integration', '<integrationId>.<serviceId>', handler)` where `handler` calls the adapter's existing `poll()` fn.
- Handler surfaces failures via `RunResult.error` **and** updates job-level `failures` / `backoff` fields via a callback the engine provides (preserving legacy backoff behaviour from `types.ts` bounds `MIN_INTERVAL_SEC..MAX_INTERVAL_SEC`, `MAX_BACKOFF_SEC`).
- **Acceptance**: an existing integration (Gmail) polls through this handler with identical timing and backoff semantics to pre-migration behaviour.

### Task 2.3: Delete legacy integrations daemon
- **File**: `src/lib/integrations/scheduler/daemon.ts`, `src/lib/integrations/scheduler/jobs.ts`
- Delete both files.
- Keep `src/lib/integrations/scheduler/types.ts` (`PollConfig`, bounds constants) — still used by the integration settings UI to shape the extras editor.
- Search-and-replace imports across the tree; wire remaining PollConfig consumers to read/write via the new job store.
- **Acceptance**: `npx tsc --noEmit` and `npm run lint` clean; grep for `integrations/scheduler/daemon` returns zero references.

### Task 2.4: Rewire integrations Settings UI to jobs
- **Files**: existing integrations settings components (`src/components/apps/settings/` — integration-editing surface); backing route `PATCH /api/integrations/[id]`.
- Change: reads and writes for polling (`enabled`, `intervalSec`, `extras`) go through the scheduler job (via new API in Task 3.1), scoped via `getEditableFields('integration')`.
- Old `state.services[svcId].poll` fields are read-only fallback for one release (see plan §Migration).
- **Acceptance**: user opens Gmail integration settings → changes interval → job's `schedule` updates → next poll respects the new interval; no writes to `state.json` `poll` block.

### Task 2.5: Register system jobs from consuming modules
- **Note**: This task is a coordination checkpoint, not a code task. Memory Loops (spec 021) will call `register('internal', 'memory-loops.fast', ...)` and seed the two system jobs itself — that work lives in 021 Phase 0 / Phase 2. No system jobs are seeded by the scheduler module itself.
- **Acceptance**: no scheduler-side change; documented in this tasks file so 021 owner knows to do it.

---

## Phase 3 — UI & API

### Task 3.1: REST API — CRUD & run-now
- **File**: `src/app/api/scheduler/jobs/route.ts` and `src/app/api/scheduler/jobs/[id]/route.ts` and `src/app/api/scheduler/jobs/[id]/run/route.ts` and `src/app/api/scheduler/jobs/[id]/history/route.ts`
- Endpoints:
  - `GET /api/scheduler/jobs` — list all (query params: `category`, `status`, `q`).
  - `POST /api/scheduler/jobs` — create; forces `category: 'user'` unless caller is a trusted internal module (validated via server-only import path); enforces `getEditableFields('user')`.
  - `GET /api/scheduler/jobs/:id` — read.
  - `PUT /api/scheduler/jobs/:id` — update; enforces `applyEdits(job, patch)` (throws 403 on disallowed field).
  - `DELETE /api/scheduler/jobs/:id` — delete; enforces `canDelete(category)` (403 for `system`).
  - `POST /api/scheduler/jobs/:id/run` — invokes `engine.runNow(id)`; returns `RunResult`.
  - `GET /api/scheduler/jobs/:id/history?limit=` — reads via `history.readRuns`.
- All endpoints log to central logging.
- **Acceptance**: manual curl covers all verbs; ACL rejection verified for `system` category `name` edit and delete.

### Task 3.2: Scheduler app — list view with category badges
- **File**: `src/apps/scheduler/index.tsx`, `src/apps/scheduler/manifest.ts`, `src/apps/scheduler/TaskList.tsx`
- Sort by `nextRunAt` (soonest first, paused/broken at bottom).
- Columns: Name, Category (badge: `system` / `user` / `integration` with distinct color), Handler kind, Next Run, Status, Actions.
- Row actions honor ACL: locked-field edits disabled with tooltip; delete hidden for `system`.
- Empty state → "Schedule New Task" CTA.
- **Acceptance**: opens from launcher; jobs from all three categories render with correct badges; ACL disables/hides appropriately.

### Task 3.3: Scheduler app — create/edit form (user jobs)
- **File**: `src/apps/scheduler/TaskForm.tsx`
- Fields for user prompt jobs: name, prompt (textarea), agent selector, schedule type (one-time / recurring), schedule details.
- On save → `POST /api/scheduler/jobs` (category forced to `user` server-side).
- Editing an existing job: form pre-populates; `PUT` on save; disabled inputs for locked fields (per ACL).
- **Acceptance**: user creates a task via UI → appears in list → runs at scheduled time → history visible.

### Task 3.4: Scheduler app — detail / history view
- **File**: `src/apps/scheduler/TaskDetail.tsx`
- Shows job metadata + last 20 history entries (via `GET /jobs/:id/history?limit=20`).
- Expandable rows for full output/error.
- **Acceptance**: history entries visible with timestamps, status, duration, output/error.

### Task 3.5: MCP tools mapped to engine
- **File**: `src/lib/scheduler/mcp-server.ts` (registered via the MCP gateway per spec 014).
- Tools (agent-facing names preserved from v1 spec — "task" is user-visible language even though internally jobs are unified):
  - `createTask(name, prompt, agentId, scheduleType, scheduleConfig)` → creates `user` category `prompt` job. Category is NOT a tool param.
  - `listScheduledTasks()` → returns all jobs (filtered per per-agent capabilities, spec 011).
  - `getTask(taskId)` → single job.
  - `updateTask(taskId, updates)` → `PUT /jobs/:id` semantics; ACL applies (agents cannot edit `system`/`integration` fields they aren't authorized for).
  - `pauseTask(taskId)`, `resumeTask(taskId)`.
  - `deleteTask(taskId)` → 403-equivalent for `system` category.
  - `runTaskNow(taskId)`.
  - `updateTaskSchedule(taskId, newScheduleConfig)`.
- All handlers go through the same store / ACL as the REST API — no bypass path.
- **Acceptance**: agent creates a task via `createTask` → task appears in the UI with `user` badge; `deleteTask` on a `system` job returns an ACL error.

### Task 3.6: Assistant flow — "schedule a task"
- **File**: existing assistant flow location (e.g. `src/lib/agent/flows/` or the CopilotKit action wiring under `src/components/agent/`).
- Detect intent; elicit prompt / agent / schedule; call `createTask`; confirm.
- **Acceptance**: manual: "schedule a task" → guided flow → task appears in the Scheduler app.

### Task 3.7: `memoryLoops` config namespace hand-off (for 021)
- **Note**: Not a scheduler task — but Memory Loops registers its `internal` handlers and seeds system jobs during 021 Phase 0. See `021-memory-loops/tasks.md` Phase 0.
- **Acceptance**: 021 Phase 0 references this scheduler tasks file for API surface.

---

## Phase 4 — Polish (deferred)

Not blocking Memory Loops. Tracked here for completeness; do only after Phases 1–3 are stable in main.

- Past-due handling policy (config + UI).
- Cron-expression schedule type.
- Timezone controls per job.
- Task search / filter in UI.
- Comprehensive unit + integration test coverage; performance test with 100+ jobs.

---

## Dependency Chain (explicit)

```
Phase 1 (Engine Core)  ──┬──►  Phase 2 (Migration & Integration Adapter)
                         │
                         └──►  021 Memory Loops Phase 2 (fast loop scheduler job)
                              └──►  021 Memory Loops Phase 3 (slow loop scheduler job)

Phase 2  ──►  Phase 3 (UI & API — user-facing jobs)
```

Phase 1 unblocks Memory Loops Phase 2. Phase 3 can proceed in parallel with Memory Loops Phase 2+ once Phase 1 is done.

---

## Notes for Developer

- **VFS server-only**: engine, store, history, migration are `import "server-only"`. Client UI talks via the `/api/scheduler/jobs` routes.
- **Atomic writes** everywhere via the existing VFS write helpers (temp-file + rename). No direct `fs.writeFile` calls.
- **Failure isolation**: every dispatch is wrapped in try/catch at the engine; a handler crash is a history entry, not a daemon crash.
- **Category is server-truth**: never trust a client-supplied `category`; the API layer sets it based on the request source (user CRUD → `user`; internal registration → whatever the module claims; migration → `integration`).
- **Idempotency**: migration marker + handler-registry checks make repeat startups safe.
- **No new npm deps**. ULID can be a tiny inline impl or an existing helper in the tree if available.
- **`npx tsc --noEmit` and `npm run lint` clean** required at each phase boundary.
