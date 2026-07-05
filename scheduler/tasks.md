# Tasks: Scheduler (Unified Job Engine)

**Feature**: Scheduler  
**Status**: Ready for Implementation  
**Branch**: `scheduler`

---

## Phase 0: Core Engine & Data Model

### Task 0.1: Define Types and Interfaces
- [ ] **File**: `src/lib/scheduler/types.ts`
- [ ] Define `JobDefinition` interface:
  - `id`, `name`, `category: 'system' | 'user' | 'integration'`
  - `handler: { kind: 'prompt' | 'internal' | 'integration', ... }`
  - `scheduleType: 'one-time' | 'recurring'`, `scheduleConfig`
  - `status: 'active' | 'paused'`, `nextRunAt`, `createdAt`, `updatedAt`
  - Optional: `owner`, `readOnlyFields: string[]`
- [ ] Define `ScheduleConfig`: `{ type: 'one-time', datetime: ISO }` or `{ type: 'recurring', interval: number, unit: 'minute'|'hour'|'day'|'week' }`
- [ ] Define `JobExecution`: `{ jobId, executedAt, status, duration, output?, error? }`
- [ ] Define `HandlerKind` types and registry signature.

**Acceptance**: Types are clear, complete, and match spec FR-002.

---

### Task 0.2: Implement ACL Logic
- [ ] **File**: `src/lib/scheduler/acl.ts`
- [ ] Implement `getEditableFields(job: JobDefinition): string[]`:
  - Returns default editable fields based on `category`.
  - Overrides with job-specific `readOnlyFields`.
- [ ] Implement `validateJobPatch(patch: Partial<JobDefinition>, allowedFields: string[]): void`:
  - Throws error if patch contains non-editable fields.
- [ ] Define default ACL rules per category (spec FR-017):
  - `user`: full control.
  - `system`: limited edit, no delete.
  - `integration`: limited edit, no delete via UI.

**Acceptance**: ACL logic correctly enforces category-based restrictions.

---

### Task 0.3: Implement Core Engine
- [ ] **File**: `src/lib/scheduler/engine.ts`
- [ ] Implement `SchedulerEngine` class:
  - `loadJobs()`: Reads `/Documents/System/scheduler-jobs.json`.
  - `saveJobs()`: Atomic write (temp + rename).
  - `start()`: Begins tick loop (default 60s).
  - `stop()`: Stops tick loop.
  - `tick()`: Checks due jobs, dispatches handlers.
- [ ] Implement handler registry:
  - `registerHandler(kind, handler)`: Registers prompt/internal/integration handlers.
  - `dispatch(job)`: Calls appropriate handler based on `job.handler.kind`.
- [ ] Implement failure isolation:
  - Try-catch around each handler call; log error; continue to next job.
- [ ] Implement run history:
  - Append `JobExecution` to `/Documents/System/scheduler-history/<jobId>.jsonl`.
- [ ] Implement external mutation detection:
  - Re-read jobs if file changes externally (e.g., user edit).

**Acceptance**: Engine loads, ticks, dispatches, persists, and handles failures correctly.

---

### Task 0.4: Create API Routes (CRUD)
- [ ] **File**: `src/app/api/scheduler/jobs/route.ts`
  - `GET`: List all jobs from unified store.
  - `POST`: Create a new User job (validates via ACL).
- [ ] **File**: `src/app/api/scheduler/jobs/[id]/route.ts`
  - `GET`: Get specific job.
  - `PUT`: Update job (validates patch via ACL).
  - `DELETE`: Delete job (blocks System/Integration jobs).
- [ ] **File**: `src/app/api/scheduler/jobs/[id]/run/route.ts`
  - `POST`: Trigger manual run immediately.

**Acceptance**: API endpoints work correctly; ACL validation enforced server-side.

---

## Phase 1: Migration & Integration Refactor

### Task 0.5: Implement Migration Script
- [ ] **File**: `src/lib/scheduler/migrate.ts`
- [ ] Implement `migrateLegacyConfigs()`:
  - Scans legacy integration polling configs (e.g., `data/integrations/**`).
  - For each, creates an `integration`-category JobDefinition with stable ID.
  - Writes all migrated jobs to unified store in one atomic write.
  - Marks legacy sources as migrated (e.g., `.migrated` suffix).
- [ ] Implement conflict handling:
  - If two legacy configs map to same ID, halt that entry; log error; continue.
- [ ] Ensure idempotency:
  - Re-running migration is a no-op if already done.

**Acceptance**: All legacy integration jobs appear in unified store after first boot; subsequent boots are no-ops.

---

### Task 0.6: Refactor Integration Polling
- [ ] **Files**: `src/lib/integrations/**` (all integrations with polling)
- [ ] Remove self-scheduling logic from each integration.
- [ ] Update integrations to:
  - Register their polling action as an `integration`-category JobDefinition in the unified store.
  - Read schedule state (interval, status) from the unified store, not local config.
- [ ] Ensure integrations expose their polling function via the engine's integration handler.

**Acceptance**: No integration self-schedules; all polling flows through the engine.

---

## Phase 2: Handler Implementation

### Task 0.7: Implement Prompt Handler
- [ ] **File**: `src/lib/scheduler/handlers/prompt.ts`
- [ ] Implement `handlePromptJob(job)`:
  - Extracts `prompt` and `agentId` from job handler config.
  - Sends prompt to agent runtime (existing agent API).
  - Returns execution result (output/error).

**Acceptance**: User tasks execute correctly via the engine.

---

### Task 0.8: Implement Internal Handler
- [ ] **File**: `src/lib/scheduler/handlers/internal.ts`
- [ ] Implement `handleInternalJob(job)`:
  - Maps `job.handler.ref` to a registered internal function (e.g., `memory.fast-loop`).
  - Calls the function; returns result.
- [ ] Update memory loops (and other system jobs) to register their handlers here.

**Acceptance**: System jobs (like memory loops) execute correctly via the engine.

---

### Task 0.9: Implement Integration Handler
- [ ] **File**: `src/lib/scheduler/handlers/integration.ts`
- [ ] Implement `handleIntegrationJob(job)`:
  - Maps `job.handler.integrationId` + `action` to an integration adapter function.
  - Calls the adapter; returns result.

**Acceptance**: Integration polling jobs execute correctly via the engine.

---

## Phase 3: UI & UX

### Task 0.10: Refactor Scheduler App UI
- [ ] **File**: `src/apps/scheduler/SchedulerApp.tsx`
- [ ] Update to fetch all jobs from API (all categories).
- [ ] Display unified list with category badges (System/User/Integration).
- [ ] Sort by `nextRunAt`.

**Acceptance**: UI shows all jobs in one list.

---

### Task 0.11: Implement JobRow Component
- [ ] **File**: `src/apps/scheduler/JobRow.tsx`
- [ ] Render job details (name, schedule, status, category).
- [ ] Implement ACL-gated actions:
  - Pause/Resume (all categories).
  - Edit Interval (User/System if allowed, Integration).
  - Delete (User only).
  - Run Now (all categories if allowed).
- [ ] Hide/disable actions based on `getEditableFields()` from engine.

**Acceptance**: UI actions are correctly restricted per category.

---

### Task 0.12: Implement New Job Modal
- [ ] **File**: `src/apps/scheduler/NewJobModal.tsx`
- [ ] Form for creating User jobs (prompt, agent, schedule).
- [ ] Validates input; calls API to create job.
- [ ] Only creates `category: 'user'` jobs.

**Acceptance**: Users can create new scheduled tasks via UI.

---

## Phase 4: Testing & Hardening

### Task 0.13: Unit Tests
- [ ] **File**: `tests/scheduler/engine.test.ts`
  - Tick loop, handler dispatch, failure isolation, atomic writes.
- [ ] **File**: `tests/scheduler/migrate.test.ts`
  - Migration idempotency, conflict handling, legacy retirement.
- [ ] **File**: `tests/scheduler/acl.test.ts`
  - `getEditableFields` logic per category.

**Acceptance**: All unit tests pass.

---

### Task 0.14: Integration & E2E Tests
- [ ] **File**: `e2e/scheduler-ui.spec.ts`
  - Create job via UI; verify it runs.
  - Pause System job; verify engine respects pause.
  - Verify migration brings in integration jobs.
- [ ] **File**: `tests/scheduler/api-integration.test.ts`
  - API CRUD operations work correctly with ACL validation.

**Acceptance**: All e2e and integration tests pass.

---

## Implementation Order (Recommended)

1.  **Phase 0** (Tasks 0.1–0.4): Core Engine + API → Foundation for everything.
2.  **Phase 1** (Tasks 0.5–0.6): Migration + Integration Refactor → Ensures unified store is the sole source of truth.
3.  **Phase 2** (Tasks 0.7–0.9): Handler Implementation → Makes jobs actually run (including memory loops).
4.  **Phase 3** (Tasks 0.10–0.12): UI & UX → User-facing management interface.
5.  **Phase 4** (Tasks 0.13–0.14): Testing → Hardening and verification.

Each phase is independently testable. Phase 0 + Phase 2 is sufficient for system jobs to run; UI can follow.

---

## Notes for Developer

- **Unified Persistence**: All jobs live in `/Documents/System/scheduler-jobs.json`. No other scheduler config files.
- **Atomic Writes**: Always use temp-file + rename for the unified store.
- **Failure Isolation**: One failing job must NOT block others. Catch errors per handler.
- **Migration is Critical**: Phase 1 ensures no integration polling is lost during the transition.
- **ACL is Server-Side**: UI restrictions are a courtesy; server must enforce them too.
- **No Cron Yet**: MVP uses `intervalMs` only. Cron is a future enhancement.
