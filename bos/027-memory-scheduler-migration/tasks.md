# Tasks: Memory Scheduler Migration to Plugin System

**Spec**: `user-specs/027-memory-scheduler-migration/spec.md`
**Plan**: `user-specs/027-memory-scheduler-migration/plan.md`

**Created**: 2026-07-20

**Status**: Draft

## Phase 1: Ensure Boot-Time Plugin Loading & Remove Boot-Time Seeding

**Purpose**: Ensure `loadAllPlugins()` runs before `startDaemon()`, then remove `ensureMemoryLoopsSeeded()` from `executor.ts` and remove the `isMemoryPluginActive()` guard.

- [ ] T000 Call `loadAllPlugins()` before `startDaemon()` in `instrumentation.ts` (or `daemon.ts`)
- [ ] T001 Remove `ensureMemoryLoopsSeeded()` call from `executor.ts:52`
- [ ] T002 Remove `isMemoryPluginActive()` guard from `fast-loop.ts:429-433`. The guard is unnecessary because the onRunFinished hook only reviews the conversation that just completed (via onlyConversationId), while the scheduled fast loop reviews all idle conversations — the two paths cover different scopes and are not duplicates.

**Checkpoint**: Boot-time seeding is removed, plugin loading is ensured before daemon start, `isMemoryPluginActive()` guard is removed.

---

## Phase 2: Update Memory Plugin Lifecycle

**Purpose**: Move job seeding into plugin lifecycle hooks, add error handling and concurrency guards, and resolve config namespace mismatch.

- [ ] T003 Import `ensureFastLoopJob` from `src/lib/agent/memory/fast-loop.ts`
- [ ] T004 Import `ensureSlowLoopJob` from `src/lib/agent/memory/consolidate.ts`
- [ ] T005 Import `pauseJob` and `resumeJob` from `src/lib/scheduler/engine.ts`
- [ ] T006 Call `ensureFastLoopJob()` and `resumeJob('system:memory.fast-loop')` in plugin `initialize()`
- [ ] T007 Call `ensureSlowLoopJob()` and `resumeJob('system:memory.slow-loop')` in plugin `initialize()`
- [ ] T008 Call `pauseJob('system:memory.fast-loop')` in plugin `dispose()`
- [ ] T009 Call `pauseJob('system:memory.slow-loop')` in plugin `dispose()`
- [ ] T010 Add mutex (promise chain) guard in plugin lifecycle to prevent concurrent activate/deactivate
- [ ] T011 Wire config changes to re-seed jobs via `setConfig`
- [ ] T012 Resolve config namespace mismatch between `getMemoryLoopsConfig()` (reads memoryLoops.json) and `setConfig()` (writes plugins.json)
- [ ] T013 Wrap `ensureFastLoopJob()` and `ensureSlowLoopJob()` calls in `initialize()` with try/catch and logging

**Checkpoint**: Plugin manages scheduler lifecycle via `initialize()` and `dispose()`, with error handling, concurrency guards, and bridged config namespaces.

---

## Phase 3: Verify

**Purpose**: Ensure behavior is preserved and all acceptance criteria are met.

- [ ] T014 Run typecheck (`npx tsc --noEmit`)
- [ ] T015 Run lint (`npm run lint`)
- [ ] T016 Verify fast loop executes on BOS startup (via plugin init)
- [ ] T017 Verify consolidation loop executes on BOS startup
- [ ] T018 Verify schedulers stop when plugin is disabled
- [ ] T019 Verify schedulers restart when plugin is re-enabled
- [ ] T020 Verify config changes update scheduler intervals
- [ ] T021 Test plugin disable during active loop execution
- [ ] T022 Test plugin re-enable re-seeds jobs correctly
- [ ] T023 Test concurrent activate/deactivate race conditions

**Checkpoint**: All acceptance criteria met, implementation complete

---

## Dependencies

- Phase 1 must complete before Phase 2 (boot-time seeding must be removed and plugin loading ensured before plugin owns it)
- Phase 2 must complete before Phase 3 (plugin must be updated before verification)

---

## Implementation Strategy

1. **Phase 1**: Ensure boot-time plugin loading, remove boot-time seeding from `executor.ts`, remove `isMemoryPluginActive()` guard
2. **Phase 2**: Update plugin lifecycle to manage jobs with error handling, concurrency guards, and config namespace bridging
3. **Phase 3**: Verify all behavior is preserved

---

## Notes

- The Unified Job Engine (`src/lib/scheduler/engine.ts`) MUST NOT be modified
- The `ensureFastLoopJob()` and `ensureSlowLoopJob()` functions stay in their respective files
- `dispose()` takes no arguments (per `types.ts:94`)
- `ensureSystemJob()` is idempotent and preserves `lastExecutedAt`; recalculates `nextRunAt` for active jobs
- Config changes must re-seed jobs to pick up new intervals
- The `owner: "memory"` field on `JobDefinition` should be used for cleanup
- T002 and T003 from the original task list were no-op verifications and have been removed
