# Feature Specification: Scheduler (Unified Job Engine — System, User, and Integration jobs)

**Feature Branch**: `scheduler`

**Created**: 2026-01-XX

**Status**: Ready for Implementation (spec, plan, and tasks complete)

**Input**: User request for a scheduling app that allows users to schedule tasks (prompts to agents) on various schedules (once, recurring). The app must have a daemon for background execution, a UI for managing tasks, and MCP tools for agent interaction. Revised 2026-07-05 to unify System, User, and Integration jobs behind a single engine and single persistence store.

> This feature provides a **Unified Job Engine** for BrowserOS. A single daemon manages every scheduled unit of work in the system — **System** jobs (built-in loops such as `002-memory` / `021-memory-loops`), **User** jobs (agent prompts scheduled via UI or assistant), and **Integration** jobs (polling external services). All job definitions live in **one source of truth** at `/Documents/System/scheduler-jobs.json` (VFS, user-accessible); run history is appended to `/Documents/System/scheduler-history/<jobId>.jsonl`. The UI presents all jobs in one management surface, with a `category` field on each JobDefinition driving UI/ACL behavior (e.g., "can I delete this?", "can I change the interval?") — **not** storage location. Storage is unified; behavior is differentiated by category.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Schedule a task via the Assistant (Priority: P1)

A user tells the assistant "I want to schedule a task" and is guided through defining the task prompt, selecting an agent, and specifying when it should run.

**Acceptance Scenarios**:

1. **Given** the user says "schedule a task", **When** the assistant responds, **Then** it asks for the task description/prompt.
2. **Given** the user provides a prompt, **When** the assistant continues, **Then** it asks which agent should receive the prompt.
3. **Given** the user selects an agent, **When** the assistant continues, **Then** it asks for the schedule (one-time or recurring with interval).
4. **Given** all details are provided, **When** the task is created, **Then** it appears in the scheduler UI and will execute at the scheduled time.

### User Story 2 - View and manage ALL scheduled jobs (System, User, Integration) via one UI (Priority: P1)

The user opens the Scheduler app and sees **every** registered job in the system — user-created tasks, System jobs (e.g., memory-loops fast/slow), and Integration polling jobs — in a single list sorted by next run time. UI actions available on each row are determined by the job's `category` (category-based ACL, FR-017), not by which subsystem created the job.

**Acceptance Scenarios**:

1. **Given** there are scheduled jobs across all categories, **When** the Scheduler app opens, **Then** every job (System, User, Integration) is displayed in one list sorted by next run time (soonest first), with a visible `category` badge.
2. **Given** a **user** job is shown, **When** the user clicks "Pause", **Then** the job is removed from the execution queue but preserved. "Delete" and "Edit prompt/schedule" are all available.
3. **Given** a **system** job (e.g., `memory.fast-loop`) is shown, **When** the user views its row, **Then** Pause/Resume and interval-adjust are available (subject to the job's `readOnlyFields`), but Delete and prompt-edit are hidden or disabled per its category ACL.
4. **Given** an **integration** job is shown, **When** the user views its row, **Then** Pause/Resume and interval-adjust are available, but the target/prompt fields are read-only (the integration owns them).
5. **Given** a paused job of any category, **When** the user clicks "Resume", **Then** the job re-enters the execution queue.
6. **Given** any job, **When** the user clicks "Run Now" (and the category permits), **Then** the job executes immediately without waiting for its schedule.
7. **Given** no jobs exist, **When** the app opens, **Then** an empty state with "Schedule New Task" button is shown.

### User Story 3 - Schedule a new task via UI (Priority: P1)

The user can create a scheduled task directly from the Scheduler UI without using the assistant.

**Acceptance Scenarios**:

1. **Given** the user clicks "Schedule New Task", **When** the form opens, **Then** they can enter a task name, prompt, select an agent, and choose schedule type.
2. **Given** one-time schedule is selected, **When** the user picks a date/time, **Then** the task is scheduled for that exact moment.
3. **Given** recurring schedule is selected, **When** the user specifies interval (e.g., "every 2 hours"), **Then** the task repeats at that interval starting from now or a specified start time.
4. **Given** all fields are valid, **When** the user saves, **Then** the task is created and appears in the task list.

### User Story 4 - Agent uses scheduler tools (Priority: P1)

An agent can programmatically create, modify, list, pause, resume, delete, and trigger tasks using MCP tools provided by the scheduler.

**Acceptance Scenarios**:

1. **Given** an agent wants to list tasks, **When** it calls `listScheduledTasks`, **Then** it receives all tasks with their status and next run times.
2. **Given** an agent wants to create a task, **When** it calls `createTask` with prompt, agentId, and schedule, **Then** the task is created and returns its ID.
3. **Given** an agent wants to pause a task, **When** it calls `pauseTask(taskId)`, **Then** the task is paused.
4. **Given** an agent wants to resume a task, **When** it calls `resumeTask(taskId)`, **Then** the task resumes.
5. **Given** an agent wants to delete a task, **When** it calls `deleteTask(taskId)`, **Then** the task is deleted.
6. **Given** an agent wants to run a task immediately, **When** it calls `runTaskNow(taskId)`, **Then** the task executes right away.
7. **Given** an agent wants to update a task's schedule, **When** it calls `updateTaskSchedule(taskId, newSchedule)`, **Then** the schedule is updated.

### User Story 5 - Background daemon executes tasks (Priority: P1)

The scheduler daemon runs in the background, checking for due tasks and executing them by sending prompts to the selected agents.

**Acceptance Scenarios**:

1. **Given** a task is scheduled for a specific time, **When** that time arrives, **Then** the daemon executes the task by sending the prompt to the designated agent.
2. **Given** a recurring task, **When** it executes, **Then** the next run time is calculated and scheduled.
3. **Given** the daemon is running, **When** BOS starts, **Then** it loads all active tasks from storage and begins monitoring.
4. **Given** a task execution fails, **When** the error occurs, **Then** it is logged but does not crash the daemon; other tasks continue to execute.

### User Story 6 - Comprehensive logging (Priority: P2)

All scheduler activities are logged using BOS's central logging system for debugging and auditing.

**Acceptance Scenarios**:

1. **Given** a task is created, **When** it happens, **Then** a log entry is recorded with task details.
2. **Given** a task executes, **When** it runs (successfully or not), **Then** execution details are logged including timing and outcome.
3. **Given** a task is paused/resumed/deleted, **When** the action occurs, **Then** it is logged.
4. **Given** the daemon starts/stops, **When** the lifecycle event happens, **Then** it is logged to the supervisor log stream.

### User Story 7 - Task history and execution results (Priority: P2)

Users can view the history of task executions, including when they ran and their outcomes.

**Acceptance Scenarios**:

1. **Given** a task has executed multiple times, **When** the user views its details, **Then** they see a history of all executions with timestamps and status.
2. **Given** a task execution produced output, **When** viewing the history, **Then** the output or result summary is available.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The scheduler MUST support two schedule types:
  - **One-time**: Execute at a specific datetime
  - **Recurring**: Execute every N units (minutes, hours, days, weeks) with configurable interval
- **FR-002**: A **JobDefinition** entity MUST contain:
  - `id`, `name`
  - `category: 'system' | 'user' | 'integration'` — drives UI/ACL differentiation only; storage is unified across categories (FR-003).
  - `handler`: how the runtime executes the job. For prompt jobs: `{ kind: 'prompt', prompt: string, agentId: string }`. For system jobs: `{ kind: 'internal', ref: string }` (e.g., `memory.fast-loop`). For integration polls: `{ kind: 'integration', integrationId: string, action: string }`.
  - `scheduleType` (one-time | recurring), `scheduleConfig` (datetime for one-time; interval + unit + optional start time for recurring)
  - `status` (active | paused), `nextRunAt` (ISO timestamp or null if paused), `createdAt`, `updatedAt`
  - Optional: `owner` (subsystem or integration id that manages this job — informational; does NOT change storage location), `readOnlyFields: string[]` (fields the UI must not let the user edit for this specific job; enforcement follows the category defaults in FR-017 unless overridden here).
- **FR-003**: The scheduler MUST persist **all** JobDefinitions — System, User, and Integration — to a **single unified store** at the VFS path `/Documents/System/scheduler-jobs.json` (user-accessible, atomic writes). There is **no** split persistence: integrations do NOT keep polling configs in their own files, and System jobs do NOT live in a separate registry. This one file is the sole source of truth; the engine reads it on startup and re-reads on external mutation (e.g., user edited it via a text editor or the file API).
- **FR-004**: A **single Unified Job Engine** (daemon) MUST run in the background, loading all JobDefinitions from `/Documents/System/scheduler-jobs.json`, checking for due jobs every minute (or configurable interval), and dispatching each to the appropriate handler (agent runtime for prompt jobs; internal function registry for system jobs; integration adapter for integration jobs). Failure isolation applies across all categories — a failing System or Integration job MUST NOT block User jobs and vice versa.
- **FR-005**: The daemon MUST be resilient: task execution failures MUST NOT crash the daemon; errors MUST be logged and the daemon continues monitoring other tasks.
- **FR-006**: After a recurring task executes, the daemon MUST calculate and update the `nextRunAt` based on the interval.
- **FR-007**: One-time tasks that have executed MUST be marked as `completed` or removed (configurable).
- **FR-008**: The Scheduler app UI MUST display tasks sorted by `nextRunAt` (soonest first), with columns: Name, Agent, Next Run, Status, and actions (Run Now, Pause/Resume, Delete, Edit).
- **FR-009**: The UI MUST provide a "Schedule New Task" form with fields: name, prompt (textarea), agent selector, schedule type selector, and schedule-specific inputs.
- **FR-010**: The UI MUST allow editing existing tasks (name, prompt, schedule) while preserving execution history.
- **FR-011**: MCP tools MUST be provided for agents to interact with the scheduler:
  - `listScheduledTasks()` → returns all tasks with current state
  - `createTask(name, prompt, agentId, scheduleType, scheduleConfig)` → creates and returns task ID
  - `getTask(taskId)` → returns full task details including history
  - `updateTask(taskId, updates)` → updates mutable fields (name, prompt, schedule)
  - `pauseTask(taskId)` → sets status to paused
  - `resumeTask(taskId)` → sets status to active and calculates nextRunAt
  - `deleteTask(taskId)` → permanently removes the task
  - `runTaskNow(taskId)` → executes immediately regardless of schedule
  - `updateTaskSchedule(taskId, newScheduleConfig)` → updates just the schedule
- **FR-012**: All scheduler operations MUST log to BOS's central logging system using the `scheduler` component name, with appropriate levels (info for normal operations, warn for skipped tasks, error for failures).
- **FR-013**: Task execution MUST create a new conversation or append to an existing one (configurable per task) in the agent's chat history.
- **FR-014**: The daemon MUST start automatically when BOS starts (as part of the app installation), and MUST gracefully handle BOS shutdown.
- **FR-015**: Jobs MUST be loaded from storage on daemon startup, and any jobs with `nextRunAt` in the past (that weren't executed) MUST either be executed immediately or marked for review (configurable).
- **FR-016**: **One-time migration of legacy integration polling configs**. On first startup after the unified-engine upgrade, the scheduler MUST scan for any pre-existing integration polling configurations (previously stored per-integration outside the scheduler) and MUST migrate them into the unified `/Documents/System/scheduler-jobs.json` as JobDefinitions with `category: 'integration'`, preserving each integration's original interval, active/paused state, and identity (integration id + action). The migration MUST:
  - Be **idempotent**: safe to re-run; already-migrated integrations MUST NOT produce duplicate jobs. Idempotency is achieved by deriving a stable `id` (e.g., `integration:<integrationId>:<action>`) and skipping if that id already exists in the store.
  - Be **backwards-compatible**: if the legacy config file(s) still exist, they are read once and then marked migrated (e.g., renamed to `.migrated` or a marker written to the unified store) so subsequent boots do not re-read them.
  - Log every migrated integration to central logging with its old source and new job id, and log a final summary count.
  - Fail loudly on ambiguity (e.g., two conflicting legacy configs for the same integration) rather than silently choose — surface an error in the scheduler status so the user can resolve.
- **FR-017**: **Category-based ACL** governs UI behavior; storage remains unified. Default rules (may be tightened per-job via `readOnlyFields`):
  - `user` — full control: create, edit (name/prompt/agent/schedule), pause/resume, delete, run-now.
  - `system` — view + pause/resume + run-now; interval editable only if the owning subsystem allows it; name/prompt/agent/handler are read-only; delete is **disabled** (system jobs are re-created on startup if missing, so deletion is not durable).
  - `integration` — view + pause/resume + interval-edit + run-now; handler/target/integrationRef are read-only (the integration owns them); delete is available only when the integration itself is uninstalled (soft cascade), not from the scheduler UI.

### Key Entities

- **JobDefinition** — the core entity containing all scheduling information for one scheduled unit of work. Carries a `category` (system/user/integration) that drives UI/ACL only; storage is unified for all categories.
- **ScheduleConfig** — varies by type: `{ type: 'one-time', datetime: ISO }` or `{ type: 'recurring', interval: number, unit: 'minute'|'hour'|'day'|'week', startTime?: ISO }`.
- **JobExecution** — record of a job run: `{ jobId, executedAt, status: 'success'|'error', duration, output?, error? }`, appended to `/Documents/System/scheduler-history/<jobId>.jsonl`.
- **Unified Job Engine** (the daemon) — the single background process that loads all JobDefinitions from the unified store, dispatches them by handler kind, and records history.
- **Handler kinds** — `prompt` (send to agent runtime), `internal` (call a registered internal function such as `memory.fast-loop`), `integration` (invoke an integration adapter). New kinds can be registered without changing the storage schema.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can schedule a task via the assistant by having a natural conversation.
- **SC-002**: A user can schedule, view, pause, resume, delete, and manually trigger tasks via the UI.
- **SC-003**: An agent can fully manage scheduled tasks using the MCP tools.
- **SC-004**: Tasks execute at their scheduled times without manual intervention.
- **SC-005**: The daemon survives task execution errors and continues operating.
- **SC-006**: All scheduler activities are visible in the central logging system.
- **SC-007**: Tasks persist across BOS restarts and resume scheduling correctly.

## Assumptions & Dependencies

- Depends on the existing agent runtime being available to execute tasks (the agent can receive prompts and process them).
- Depends on BOS's central logging system (`017-central-logging`) for all log output.
- Depends on the app installation system (`009-installed-apps`) for deploying the scheduler as an installed app.
- The daemon runs as a background Node.js process within the BOS environment (similar to how other background services work).
- Task execution uses the existing MCP tool gateway or direct agent invocation (to be finalized in implementation).

## Design notes (non-normative)

**Architecture overview — Unified Job Engine:**
```
                       /Documents/System/scheduler-jobs.json
                          (single source of truth — all categories)
                                        │
                                        ▼
   ┌───────────────────────────────────────────────────────────────┐
   │                    UNIFIED JOB ENGINE                         │
   │  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
   │  │  Loader +    │  │  Tick / dispatch │  │  Handler         │ │
   │  │  migration   │  │  loop (~60s)     │  │  registry        │ │
   │  │  (FR-016)    │  │                  │  │  prompt/internal │ │
   │  │              │  │                  │  │  /integration    │ │
   │  └──────────────┘  └────────┬─────────┘  └────────┬─────────┘ │
   │                             │                     │           │
   │                             ▼                     ▼           │
   │                    /Documents/System/scheduler-history/       │
   │                          <jobId>.jsonl                        │
   └──────────────────────────────┬───┬───────────────────────────┘
                                  │   │
     ┌────────────────────────────┘   └─────────────────────────┐
     ▼                                                          ▼
┌─────────────────┐  ┌──────────────────┐  ┌───────────────────────────┐
│ Prompt handler  │  │ Internal handler │  │ Integration handler       │
│ → Agent runtime │  │ → e.g. memory    │  │ → integration adapter     │
│                 │  │   fast/slow loop │  │   (poll external service) │
└─────────────────┘  └──────────────────┘  └───────────────────────────┘
                                  │
                                  ▼
                        Scheduler App UI (one list, category-based ACL)
                                  │
                                  ▼
                        MCP Tools (agents interact with jobs)
```

**Storage layout (unified):**
- `/Documents/System/scheduler-jobs.json` — **the** JobDefinition store for all categories (System, User, Integration). Atomic writes; the engine re-reads on external mutation.
- `/Documents/System/scheduler-history/<jobId>.jsonl` — per-job append-only JobExecution history.
- No `data/scheduler/*` directory. No per-integration polling config files. If any legacy files exist at first boot, they are migrated per FR-016 and then archived.

**Engine loop:**
1. Load JobDefinitions from `/Documents/System/scheduler-jobs.json` (run migration FR-016 on first boot).
2. For each active job: is `nextRunAt <= now`?
3. If yes, dispatch via the handler registry (prompt → agent runtime; internal → registered function; integration → adapter).
4. Update `nextRunAt` based on schedule type; append a JobExecution line to that job's history file.
5. Log the execution to central logging (`component: 'scheduler'`, include `category` and `handler.kind` in the record).
6. Wait 60 seconds (or configured interval) and repeat.

**Category → ACL mapping (default; see FR-017):**

| Category    | Create | Edit prompt/handler | Edit interval | Pause/Resume | Run Now | Delete                     |
|-------------|--------|---------------------|---------------|--------------|---------|----------------------------|
| user        | ✅     | ✅                  | ✅            | ✅           | ✅      | ✅                         |
| system      | ❌ (subsystem creates on boot) | ❌ | ✅ (if owner allows) | ✅ | ✅ | ❌ (recreated on boot) |
| integration | ❌ (integration creates on install / migrate) | ❌ | ✅ | ✅ | ✅ | ❌ (only via integration uninstall) |

**Schedule calculation:**
- One-time: `nextRunAt = scheduled datetime`
- Recurring: `nextRunAt = lastExecution + (interval * unit)` or `startTime + (interval * unit)` for first run

**MCP tool implementation:**
- Tools are registered via the MCP gateway pattern (`014-mcp-tool-gateway`)
- The scheduler app registers as an MCP server exposing the task management tools
- Agents discover and call these tools through the standard gateway

**Logging integration:**
- All operations use the BOS logging service with `component: 'scheduler'`
- Execution results include timing, status, and any error details
- Daemon lifecycle events (start, stop, reload) are logged to supervisor stream

## Migration Strategy (normative, expands FR-016)

The move to the Unified Job Engine changes where existing integration polling lives. To make this safe:

1. **Discovery**: On first boot after upgrade, the engine invokes a migration step that scans known legacy locations for integration polling configs (per-integration config namespaces / integration-owned files under `data/integrations/**`). Each discovered polling config is normalized into a JobDefinition (`category: 'integration'`, `handler: { kind: 'integration', integrationId, action }`, preserving interval/status/schedule).
2. **Stable id**: The derived JobDefinition uses `id = 'integration:<integrationId>:<action>'` so the migration is idempotent — re-running it on a store that already contains the job is a no-op.
3. **Write into unified store**: All migrated JobDefinitions are written into `/Documents/System/scheduler-jobs.json` in a single atomic write. Existing entries in the file (e.g., user-created jobs from a partial upgrade) are preserved.
4. **Retire legacy source**: Once written, the legacy source is either (a) rewritten to reference the new job id (so the integration knows its polling is now scheduler-managed), or (b) marked migrated (e.g., renamed with `.migrated` suffix or a marker written into the unified store's `_meta.migratedSources`).
5. **Reboot safety**: On subsequent boots the migration step still runs but is a no-op (idempotent by stable id + marker). Newly installed integrations register their polling job directly into the unified store — they do NOT go back through migration.
6. **Failure mode**: If a legacy config is ambiguous or contradicts an existing job in the unified store, migration halts for that entry and surfaces the conflict via central logging + scheduler status; other entries continue. The user resolves via the UI or by editing `/Documents/System/scheduler-jobs.json` directly.
7. **Uninstall**: When an integration is uninstalled later, its owning code MAY remove its integration-category jobs from the unified store; the scheduler engine tolerates missing handlers by pausing the job with a "handler unavailable" status rather than crashing.

The migration is a one-time transitional step; after the first successful boot post-upgrade, only the unified store is authoritative. There is no ongoing "derived view" of integration jobs — they are first-class rows in the same JSON file as every other job.

## Notes

- This feature complements the assistant by providing persistent job scheduling for the whole system, not just user-authored tasks.
- The design follows BOS conventions for apps, logging, and MCP tool exposure.
- **Non-negotiable principle: one file, one API, one UI.** All jobs are the user's — even System and Integration jobs — so all configuration lives in one user-accessible VFS location (`/Documents/System/scheduler-jobs.json`). The user can inspect and edit that file directly if they wish; the engine re-reads on external changes.
- Future enhancements could include: cron-expression support, task dependencies, notifications, task templates.
