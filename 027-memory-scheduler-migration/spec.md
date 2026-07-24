# Spec: Memory Scheduler Migration to Plugin System

**Created**: 2026-07-20

**Status**: Draft

**Priority**: P1

## Summary

Migrate the memory system's scheduled jobs (fast loop and consolidation loop) from boot-time seeding in `executor.ts` to plugin-managed lifecycle. The Unified Job Engine (`engine.ts`) remains generic — the memory plugin seeds system jobs via `ensureSystemJob()` and cleans up via `pauseJob()` in its lifecycle hooks.

## User Story

**As a** BOS user,
**I want** the memory system's scheduled jobs to be managed by the memory plugin,
**So that** I can enable/disable them through Settings → Plugins and they respect the plugin lifecycle.

## Requirements

### FR-001: Plugin Manages Scheduler Lifecycle

The memory plugin MUST seed system jobs in the `initialize()` hook and pause them in the `dispose()` hook.

```typescript
// src/plugins/memory/index.ts
import { ensureFastLoopJob } from '@/lib/agent/memory/fast-loop';
import { ensureSlowLoopJob } from '@/lib/agent/memory/consolidate';
import { pauseJob, resumeJob } from '@/lib/scheduler/engine';

export const memoryPlugin: BosPluginHooks = {
  async initialize() {
    await ensureFastLoopJob();
    await resumeJob('system:memory.fast-loop');
    await ensureSlowLoopJob();
    await resumeJob('system:memory.slow-loop');
  },
  async dispose() {
    await pauseJob('system:memory.fast-loop');
    await pauseJob('system:memory.slow-loop');
  },
};
```

**Acceptance Criteria**:
- [ ] `ensureFastLoopJob()` is called in `initialize()`
- [ ] `ensureSlowLoopJob()` is called in `initialize()`
- [ ] `resumeJob()` is called for both jobs in `initialize()`
- [ ] `initialize()` catches and logs errors from `ensureFastLoopJob()` / `ensureSlowLoopJob()` without preventing plugin initialization
- [ ] `pauseJob()` is called for both jobs in `dispose()`
- [ ] Boot-time seeding is removed from `executor.ts`
- [ ] Plugin init runs before daemon `startDaemon()`
- [ ] Jobs are paused when plugin is disabled
- [ ] Jobs are re-seeded when plugin is re-enabled
- [ ] Concurrent activate/deactivate is guarded with a mutex/promise chain

### FR-002: Job Seeding Functions Stay in Memory Subsystem

The `ensureFastLoopJob()` and `ensureSlowLoopJob()` functions MUST remain in `fast-loop.ts` and `consolidate.ts` respectively — they own the canonical job spec and config. The migration only moves the **call site** from `executor.ts` to the plugin.

**Acceptance Criteria**:
- [ ] `ensureFastLoopJob()` stays in `src/lib/agent/memory/fast-loop.ts`
- [ ] `ensureSlowLoopJob()` stays in `src/lib/agent/memory/consolidate.ts`
- [ ] `ensureMemoryLoopsSeeded()` is removed or conditionalized in `src/lib/scheduler/executor.ts`
- [ ] The `isMemoryPluginActive()` guard in `fast-loop.ts:429-433` is removed. The guard is unnecessary because the onRunFinished hook only reviews the conversation that just completed (via onlyConversationId), while the scheduled fast loop reviews all idle conversations — the two paths cover different scopes and are not duplicates.
- [ ] Both files handle errors gracefully

### FR-003: Scheduler Infrastructure Remains Unchanged

The Unified Job Engine (`src/lib/scheduler/engine.ts`) MUST NOT be modified. The memory plugin uses existing APIs (`ensureSystemJob()`, `pauseJob()`, `registerInternalRef()`) without changes to the engine.

**Acceptance Criteria**:
- [ ] No changes to `src/lib/scheduler/engine.ts`
- [ ] No changes to `src/lib/scheduler/internal-handler.ts`
- [ ] No changes to `src/lib/scheduler/types.ts`
- [ ] Scheduler interface remains unchanged

### FR-004: Plugin Config Drives Scheduler Behavior

Scheduler intervals MUST be configurable through the plugin's `configSchema`. When config changes via Settings, the plugin's `setConfig` must update the job definitions via `ensureSystemJob()`.

**Acceptance Criteria**:
- [ ] Fast loop interval is controlled by `config.fastLoop.tickIntervalSec`
- [ ] Consolidation interval is controlled by `config.slowLoop.intervalSec`
- [ ] Default values match existing behavior (120s for fast loop, 3600s for consolidation)
- [ ] Config changes re-seed jobs via `ensureFastLoopJob()` / `ensureSlowLoopJob()` to pick up new intervals
- [ ] Config changes are validated before applying
- [ ] Config changes via Settings → Plugins → Memory update the scheduler job intervals (config namespace is bridged)

### FR-005: Backward Compatibility

Existing scheduler behavior MUST be preserved.

**Acceptance Criteria**:
- [ ] Fast loop runs on BOS startup (via plugin `initialize()`)
- [ ] Consolidation loop runs on BOS startup (via plugin `initialize()`)
- [ ] Scheduler intervals match previous defaults (120s fast loop, 3600s consolidation)
- [ ] No behavior change for users with existing config
- [ ] `lastExecutedAt` is preserved across plugin disable/enable; `nextRunAt` is recalculated from `lastExecutedAt` and current schedule config when the job is resumed

### FR-006: Boot-Time Plugin Loading

`loadAllPlugins()` must be called before `startDaemon()` to ensure plugin lifecycle hooks run before the daemon starts ticking. Currently `loadAllPlugins()` is only called lazily on the first assistant run (`src/app/api/assistant/runs/route.ts:30`), but `startDaemon()` runs at boot via `instrumentation.ts:11`.

**Acceptance Criteria**:
- [ ] `loadAllPlugins()` is called before `startDaemon()` in `instrumentation.ts` (or `daemon.ts`)
- [ ] Plugin `initialize()` hooks run before the daemon starts ticking
- [ ] Memory plugin jobs are seeded before the daemon's first tick

## Implementation Plan

### Phase 1: Remove Boot-Time Seeding

**Goal**: Remove `ensureMemoryLoopsSeeded()` from `executor.ts`

- [ ] T001 Remove `ensureMemoryLoopsSeeded()` call from `executor.ts:52`
- [ ] T002 Verify `ensureFastLoopJob()` and `ensureSlowLoopJob()` stay in their respective files
- [ ] T003 Clarify or remove `isMemoryPluginActive()` guard in `fast-loop.ts:429-433`

### Phase 2: Update Memory Plugin Lifecycle

**Goal**: Move job seeding into plugin lifecycle hooks

- [ ] T004 Import `ensureFastLoopJob` and `ensureSlowLoopJob` in `src/plugins/memory/index.ts`
- [ ] T005 Call `ensureFastLoopJob()` in `initialize()`
- [ ] T006 Call `ensureSlowLoopJob()` in `initialize()`
- [ ] T007 Import `pauseJob` from `engine.ts`
- [ ] T008 Call `pauseJob()` for both jobs in `dispose()`
- [ ] T009 Add guard against concurrent activate/deactivate
- [ ] T010 Wire config changes to re-seed jobs via `setConfig`

### Phase 3: Verify

**Goal**: Ensure behavior is preserved

- [ ] T011 Run typecheck (`npx tsc --noEmit`)
- [ ] T012 Run lint (`npm run lint`)
- [ ] T013 Verify fast loop executes on BOS startup (via plugin init)
- [ ] T014 Verify consolidation loop executes on BOS startup
- [ ] T015 Verify schedulers stop when plugin is disabled
- [ ] T016 Verify schedulers restart when plugin is re-enabled
- [ ] T017 Verify config changes update scheduler intervals
- [ ] T018 Test plugin disable during active loop execution
- [ ] T019 Test plugin re-enable re-seeds jobs correctly
- [ ] T020 Test concurrent activate/deactivate race conditions

## Dependencies

- Depends on: `user-specs/026-plugin-pipeline` (plugin infrastructure)
- Blocks: None

## Risks

| Risk | Mitigation |
|------|------------|
| Plugin init runs after daemon start | Ensure plugin init is awaited before `startDaemon()` in `daemon.ts` |
| Jobs don't resume after plugin re-enable | `ensureSystemJob()` is idempotent and preserves status |
| Config changes don't update running jobs | Re-seed jobs via `ensureFastLoopJob()` / `ensureSlowLoopJob()` on config change |
| Concurrent activate/deactivate race | Add mutex/guard in plugin lifecycle |

## Open Questions

None.

## References

- [Plugin Pipeline Spec](user-specs/026-plugin-pipeline/spec.md)
- [Scheduler Engine](src/lib/scheduler/engine.ts)
- [Scheduler Executor](src/lib/scheduler/executor.ts)
- [Memory Fast Loop](src/lib/agent/memory/fast-loop.ts)
- [Memory Consolidation](src/lib/agent/memory/consolidate.ts)
- [Memory Plugin](src/plugins/memory/index.ts)
