# Implementation Plan: Unified Job Engine (Scheduler)

**Feature**: Scheduler (Unified Job Engine)  
**Spec Status**: Draft → Ready for Planning  
**Target Branch**: `scheduler`

---

## Technical Context

The existing scheduling infrastructure in BrowserOS is fragmented:
- **Integration Polling**: Each integration (Gmail, Drive) manages its own polling loop and config, often scattered across `data/integrations/` or custom state files.
- **Task Scheduling**: A separate "Scheduler App" concept exists for user-created tasks, but lacks a unified engine to manage system jobs.
- **No Centralized Control**: Users cannot view or manage all scheduled activities (system, user, integration) from one place.

This feature introduces a **Unified Job Engine** that:
1.  Manages **all** scheduled jobs (System, User, Integration) in a single store (`/Documents/System/scheduler-jobs.json`).
2.  Provides a unified API and UI for job management.
3.  Migrates existing integration polling configs into this unified model.
4.  Supports three handler types: `prompt` (user tasks), `internal` (system loops like memory), and `integration` (adapter polling).

### Existing Touchpoints

| Component | Path | Role in This Feature |
|-----------|------|---------------------|
| Integration State | `data/integrations/` | Source of legacy polling configs to be migrated. |
| Scheduler App (Legacy) | `src/apps/scheduler/` | Will be refactored to use the new engine and display all job categories. |
| Config Registry | `src/lib/config/registry.ts` | May need a namespace for scheduler defaults (e.g., tick interval). |
| Central Logging | `src/lib/logging/` | All job executions log here with `component: 'scheduler'`. |
| VFS | `/Documents/` | Unified store and history files live here. |

### New Modules to Create

```
src/lib/scheduler/
├── engine.ts             # Core daemon: loads jobs, ticks, dispatches handlers, persists state
├── migrate.ts            # One-time migration script for legacy integration configs
├── acl.ts                # Category-based ACL logic (getEditableFields, validation)
├── types.ts              # JobDefinition, ScheduleConfig, JobExecution, HandlerKind
└── api.ts                # Helper functions for job CRUD (used by API routes + UI)

src/apps/scheduler/       # Refactored Scheduler App
├── SchedulerApp.tsx      # Main UI: unified list with category badges
├── JobRow.tsx            # Row component with ACL-gated actions
└── NewJobModal.tsx       # Form for creating User jobs

src/app/api/scheduler/    # API Routes
├── jobs/route.ts         # GET (list), POST (create)
├── jobs/[id]/route.ts    # GET, PUT, DELETE
└── jobs/[id]/run/route.ts# POST (manual trigger)

/Documents/System/        # VFS Layout
├── scheduler-jobs.json   # Single source of truth for all JobDefinitions
└── scheduler-history/    # Append-only history per job
    └── <jobId>.jsonl
```

---

## Constitution Check

Against `.specify/memory/constitution.md`:

| Principle | Compliance |
|-----------|------------|
| **Specs before code** | ✅ This plan derives from the approved `scheduler/spec.md`. |
| **No npm dependencies** | ✅ No new deps; simple interval-based scheduling (cron deferred). |
| **User data ownership** | ✅ All jobs live in `/Documents/System/`, user-accessible and editable. |
| **Atomic writes** | ✅ Engine uses temp-file + rename for `scheduler-jobs.json`. |
| **Failure isolation** | ✅ Engine catches handler errors; one failing job doesn't block others. |
| **Backwards compatibility** | ✅ Migration script (FR-016) ensures existing integrations are preserved and moved safely. |

---

## Project Structure (Real Paths)

### Phase 0: Core Engine & Data Model
```
src/lib/scheduler/types.ts                # NEW — JobDefinition, ScheduleConfig, etc.
src/lib/scheduler/acl.ts                  # NEW — ACL logic
src/lib/scheduler/engine.ts               # NEW — Daemon, tick loop, handler registry
src/app/api/scheduler/jobs/route.ts       # NEW — GET/POST
src/app/api/scheduler/jobs/[id]/route.ts  # NEW — GET/PUT/DELETE
```

**Deliverable**: Engine can load/save jobs, dispatch handlers (mocked), and persist state. API works.

### Phase 1: Migration & Integration Refactor
```
src/lib/scheduler/migrate.ts              # NEW — Legacy config migration
src/lib/integrations/**                   # MODIFY — Remove self-scheduling; register with engine
src/app/api/scheduler/jobs/[id]/run/route.ts # NEW — Manual trigger
```

**Deliverable**: All legacy integration polling configs migrated to `/Documents/System/scheduler-jobs.json`. Integrations no longer self-schedule.

### Phase 2: Handler Implementation
```
src/lib/scheduler/handlers/prompt.ts      # NEW — Dispatch to agent runtime
src/lib/scheduler/handlers/internal.ts    # NEW — Dispatch to internal function registry
src/lib/scheduler/handlers/integration.ts # NEW — Dispatch to integration adapter
src/lib/agent/memory/fast-loop.ts         # MOD — Register `memory.fast-loop` handler
src/lib/agent/memory/consolidate.ts       # MOD — Register `memory.slow-loop` handler
```

**Deliverable**: All three handler types functional. Memory loops can register themselves as System jobs.

### Phase 3: UI & UX
```
src/apps/scheduler/SchedulerApp.tsx       # UPDATE — Unified list view
src/apps/scheduler/JobRow.tsx             # NEW — ACL-gated actions
src/apps/scheduler/NewJobModal.tsx        # NEW — Create User jobs
```

**Deliverable**: Users can view, manage, and create jobs from a single UI. Category badges visible.

### Phase 4: Testing & Hardening
```
tests/scheduler/engine.test.ts            # NEW — Tick loop, failure isolation
tests/scheduler/migrate.test.ts           # NEW — Migration idempotency
e2e/scheduler-ui.spec.ts                  # NEW — UI flows
```

**Deliverable**: All unit and e2e tests passing. Migration verified on clean upgrade.

---

## Design Notes

### Unified Store Strategy
- **Single File**: `/Documents/System/scheduler-jobs.json` holds all JobDefinitions.
- **Atomic Writes**: Engine writes to a temp file, then renames to the final path.
- **External Mutation**: Engine watches for external changes (e.g., user editing JSON) and re-reads on change.
- **History**: Run history is appended to `/Documents/System/scheduler-history/<jobId>.jsonl` (JSONL format).

### Handler Registry
- **Prompt Handler**: Sends a prompt to a specific agent (user tasks).
- **Internal Handler**: Calls a registered function (e.g., `memory.fast-loop`).
- **Integration Handler**: Invokes an integration adapter's polling action.
- **Extensibility**: New handler kinds can be registered without changing the storage schema.

### Migration Strategy (FR-016)
- **Idempotent**: Uses stable IDs (`integration:<id>:<action>`) to avoid duplicates.
- **Ambiguity Handling**: Halts on conflicting legacy configs; logs error; continues with others.
- **Legacy Retirement**: Marks migrated sources (e.g., `.migrated` suffix) to prevent re-migration.

### Category-Based ACL (FR-017)
- **User Jobs**: Full control (create, edit, delete).
- **System Jobs**: View, pause/resume, run-now; limited edit (interval only if allowed); no delete.
- **Integration Jobs**: View, pause/resume, interval-edit; no handler/target edit; no delete (via UI).

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Migration fails or creates duplicates | Idempotent stable IDs; conflict halting; clear logging. |
| Engine crashes and loses state | Atomic writes; history files for recovery; re-read on boot. |
| One failing job blocks others | Try-catch per handler; log error; continue to next job. |
| User edits JSON manually | Engine re-reads on change; validation on load. |
| History file grows too large | Rotate/compact history (e.g., keep last 100 runs); future enhancement. |

---

## Testing Strategy

### Unit Tests
- `engine.test.ts`: Tick loop, handler dispatch, failure isolation, atomic writes.
- `migrate.test.ts`: Idempotency, conflict handling, legacy source retirement.
- `acl.test.ts`: `getEditableFields` logic per category.

### Integration Tests
- Engine loads jobs from JSON; executes them; appends history.
- Migration runs once; subsequent runs are no-ops.
- Manual trigger via API works for all categories.

### E2E Tests (Playwright)
- User creates a job via UI; it appears in the list and runs.
- User pauses a System job; engine respects pause.
- Integration job appears in UI after migration; user can pause/resume.

---

## Dependencies & Ordering

| Step | Depends On | Blocks |
|------|------------|--------|
| Phase 0 (Engine + API) | None | Phase 1, Phase 2 |
| Phase 1 (Migration + Integration Refactor) | Phase 0 | Phase 3 (UI needs migrated jobs) |
| Phase 2 (Handler Implementation) | Phase 0 | Phase 3 (UI needs working handlers) |
| Phase 3 (UI & UX) | Phase 1, Phase 2 | Feature complete |
| Phase 4 (Testing) | All phases | Shippable release |

**MVP Shippable After Phase 2**: Engine runs, jobs execute (including memory loops), migration done. UI can be added in Phase 3 if needed for immediate use (CLI/API access sufficient for system jobs).

---

## Open Questions

1.  **Tick Interval**: Default is 60 seconds. Should this be configurable?
2.  **History Retention**: How many runs to keep per job before rotation? (Default: 100)
3.  **Manual Trigger Auth**: Should `POST /api/scheduler/jobs/[id]/run` require specific permissions? (Assume yes, standard user auth).

These are answered in the `clarify` step if needed; otherwise defaults apply as written in spec.
