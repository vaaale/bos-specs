# Plan: Memory Scheduler Migration to Plugin System

**Spec**: `user-specs/027-memory-scheduler-migration/spec.md`

**Created**: 2026-07-20

**Status**: Draft

## Summary

Migrate the memory system's scheduled jobs (fast loop and consolidation loop) from boot-time seeding in `executor.ts` to plugin-managed lifecycle. The Unified Job Engine (`engine.ts`) remains generic — the memory plugin seeds system jobs via `ensureSystemJob()` and cleans up via `pauseJob()` in its lifecycle hooks.

## Tasks

### Task 0: Ensure Boot-Time Plugin Loading

**Description**: Ensure `loadAllPlugins()` runs before `startDaemon()` so that plugin lifecycle hooks (including memory plugin's `initialize()`) execute before the scheduler daemon starts ticking.

**Files**:
- `src/app/api/assistant/runs/route.ts` — Confirm `loadAllPlugins()` is called
- `src/lib/scheduler/instrumentation.ts` or `daemon.ts` — Move `loadAllPlugins()` before `startDaemon()`

**Acceptance Criteria**:
- [ ] `loadAllPlugins()` is called in `instrumentation.ts` (or `daemon.ts`) before `startDaemon()`
- [ ] Plugin `initialize()` hooks run before the daemon's first tick
- [ ] Memory plugin jobs are seeded before the daemon starts

---

### Task 1: Remove Boot-Time Seeding

**Description**: Remove `ensureMemoryLoopsSeeded()` from `executor.ts`. The `ensureFastLoopJob()` and `ensureSlowLoopJob()` functions stay in their respective files — they own the canonical job spec.

**Files**:
- `src/lib/scheduler/executor.ts` — Remove `ensureMemoryLoopsSeeded()` call
- `src/lib/agent/memory/fast-loop.ts` — Clarify or remove `isMemoryPluginActive()` guard

**Acceptance Criteria**:
- [ ] `ensureMemoryLoopsSeeded()` is removed from `executor.ts`
- [ ] `ensureFastLoopJob()` stays in `fast-loop.ts`
- [ ] `ensureSlowLoopJob()` stays in `consolidate.ts`
- [ ] `isMemoryPluginActive()` guard is clarified or removed

---

### Task 2: Update Memory Plugin Lifecycle

**Description**: Move job seeding into plugin lifecycle hooks. The plugin calls `ensureFastLoopJob()` / `ensureSlowLoopJob()` in `initialize()` and `pauseJob()` in `dispose()`.

**Files**:
- `src/plugins/memory/index.ts` — Add lifecycle management

**Acceptance Criteria**:
- [ ] `ensureFastLoopJob()` is called in `initialize()`
- [ ] `ensureSlowLoopJob()` is called in `initialize()`
- [ ] `pauseJob()` is called for both jobs in `dispose()`
- [ ] Concurrent activate/deactivate is guarded with a mutex/promise chain
- [ ] Config changes re-seed jobs via `setConfig`
- [ ] Config namespace mismatch is resolved (getMemoryLoopsConfig reads from the same namespace that setConfig writes to, OR setConfig also updates the memoryLoops config store)

---

### Task 3: Verify

**Description**: Ensure behavior is preserved and all acceptance criteria are met.

**Acceptance Criteria**:
- [ ] Fast loop executes on BOS startup (via plugin init)
- [ ] Consolidation loop executes on BOS startup
- [ ] Schedulers stop when plugin is disabled
- [ ] Schedulers restart when plugin is re-enabled
- [ ] Config changes update scheduler intervals
- [ ] Plugin disable during active loop execution is handled
- [ ] Plugin re-enable re-seeds jobs correctly
- [ ] Concurrent activate/deactivate race conditions are tested

---

## Dependencies

- Task 0 must complete before Task 1 (plugin loading must be ensured before boot-time seeding is removed)
- Task 1 must complete before Task 2 (boot-time seeding must be removed before plugin owns it)
- Task 2 must complete before Task 3 (plugin must be updated before verification)

---

## Risks

| Risk | Mitigation |
|------|------------|
| Plugin init runs after daemon start | Ensure plugin init is awaited before `startDaemon()` in `daemon.ts` |
| Jobs don't resume after plugin re-enable | `ensureSystemJob()` is idempotent and preserves status |
| Config changes don't update running jobs | Re-seed jobs via `ensureFastLoopJob()` / `ensureSlowLoopJob()` on config change |
| Concurrent activate/deactivate race | Add mutex/guard in plugin lifecycle |

---

## Open Questions

None.
