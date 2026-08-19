# Tasks: Service Daemons

**Input**: Design documents from `user-specs/002-service-daemons/`

**Spec**: `user-specs/002-service-daemons/spec.md`
**Plan**: `user-specs/002-service-daemons/plan.md`

**Tests**: Unit and integration tests included for backend logic. UI tests included for Settings components.

## Phase 1: Setup & Foundations

**Purpose**: Project initialization, manifest format definition, and symlink infrastructure.

- [ ] T001 Create `src/core/service/types.ts` with TypeScript types: `ServiceManifest`, `ServiceDefinition`, `ServiceState`, `CrashRecoveryPolicy`, `MainToWorkerMessage`, `WorkerToMainMessage`
- [ ] T002 Define `service.json` manifest format in spec (already done in spec.md) and create validation schema in `src/core/service/manifestValidator.ts`
- [ ] T003 Implement `src/system/marketplace/install/symlinkManager.ts` — `createSymlinks(itemPath, serviceId)`, `removeSymlinks(serviceId)`, `createConfigSymlink(itemPath, serviceId)`
- [ ] T004 Implement `src/system/marketplace/install/serviceInstaller.ts` — `installService(itemPath)`, `uninstallService(serviceId)` with symlink + registry update + notification flow
- [ ] T005 [P] Add `services/<id>` directory to item structure validation (already defined in spec)

**Checkpoint**: Symlink creation/removal works for service items. Manifest validation catches malformed entries at install time.

---

## Phase 2: Service Registry

**Purpose**: Core registry that discovers services, tracks source vs installed state, and resolves dependencies.

- [ ] T006 Implement `src/core/service/ServiceRegistry.ts` — `discoverServices()`, `getService(serviceId)`, `getAllServices()`, `getSourceServices()` (returns items in user-apps/marketplace that have `services/` but no symlink)
- [ ] T007 Implement `src/core/service/DependencyResolver.ts` — `resolveOrder(manifests)` (topological sort), `detectCircular(manifests)` (warning), `detectMissing(manifests)` (warning), `checkDependenciesRunning(serviceId)` (health check for CH-003)
- [ ] T008 Wire registry into BOS startup — `ServiceRegistry.initialize()` called during boot sequence (NFR-003), services started after boot completes
- [ ] T009 Add registry event emitter — `on('service:status:changed', callback)`, `on('service:crash', callback)`, `on('service:bound', callback)` for Settings UI real-time updates (CH-009)
- [ ] T0010 [P] Implement `src/core/service/PortChecker.ts` — `checkPortAvailable(port, host)` for port conflict detection before binding

**Checkpoint**: Registry discovers services, resolves dependency order, emits events for UI updates. Port checker validates availability.

---

## Phase 3: Service Lifecycle (Core)

**Purpose**: Worker thread management, IPC protocol, crash recovery, and lifecycle operations.

- [ ] T0011 Implement `src/core/service/ServiceManager.ts` — `start(serviceId)`, `stop(serviceId)`, `restart(serviceId)`, `getStatus(serviceId)`, `getWorker(serviceId)`
- [ ] T0012 Implement worker thread creation with resource limits — `new Worker(entryPath, { workerData: { configDirPath, logsPath, serviceId }, resourceLimits: { maxOldGenerationSizeMb: 256, maxYoungGenerationSizeMb: 64 } })`
- [ ] T0013 Implement IPC message handling — `waitForMessage(worker, type)` promise helper, `postMessage` send for `initialize`, `dispose`, `restart`
- [ ] T0014 [P] Implement crash detection in `ServiceManager` — listen for Worker `exit` event (non-zero code), Worker `error` event, and startup timeout. All trigger `handleCrash(serviceId)` (CH-001 fix)
- [ ] T0015 [P] Implement crash recovery in `src/core/service/CrashRecovery.ts` — `handleCrash(serviceId)` increments restart counter, waits exponential backoff, calls `start(serviceId)` again. If `restartCount >= maxRestarts`, stop permanently. Counter resets on `initialized` message.
- [ ] T0016 Implement `bound` message handling — when worker sends `{ type: 'bound', port, host }`, service manager writes `{ "port": port, "host": host }` to `dataDir()/config/<id>/runtime.json` (CH-005 + CH-002 fix)
- [ ] T0017 Implement log capture — redirect Worker stdout/stderr to `dataDir()/logs/services/<id>.log` (handled by logging layer, not service manager)
- [ ] T0018 Implement Start lifecycle operation — validate manifest, resolve entrypoint, create worker, send `initialize`, wait for `initialized` (timeout configurable per CH-015), emit `service:status:changed` event
- [ ] T0019 Implement Stop lifecycle operation — send `dispose`, wait for `disposed` (timeout configurable), terminate worker, emit `service:status:changed`, reset restart counter
- [ ] T0020 Implement Restart lifecycle operation — call Stop then Start, status briefly shows "Restarting"
- [ ] T0021 [P] Implement configurable timeouts (CH-015) — `startupTimeout` (default 30s, min 0), `shutdownTimeout` (default 30s, min 0). If 0, proceed without waiting. If exceeded, `worker.terminate()` and log at `warn` level

**Checkpoint**: Services start/stop/restart reliably. Crashes are detected and recovery kicks in. `runtime.json` is written with actual bound port.

---

## Phase 4: Settings UI

**Purpose**: Settings → Plugins → [Services] section with lifecycle management, config editing, and log viewing.

- [ ] T0022 Create `src/settings/plugins/ServicesTab.tsx` — main tab component that lists all installed services, subscribes to registry events, renders `ServiceCard` for each
- [ ] T0023 Implement `src/settings/plugins/ServiceCard.tsx` — service name, status indicator (running/stopped/restarting/crashed), version, Start/Stop/Restart buttons, Config button, Logs button
- [ ] T0024 Implement auto-saving config panel — `src/settings/plugins/ServiceConfigPanel.tsx` reads user config from `dataDir()/config/<id>/<configFile>.json`, writes changes directly (no Save button), shows warning if config is from read-only marketplace source
- [ ] T0025 Implement logs viewer — `src/settings/plugins/ServiceLogViewer.tsx` displays `dataDir()/logs/services/<id>.log` content, supports scroll, clear button
- [ ] T0026 Wire [Services] section into Settings app — add to Settings → Plugins sidebar, below [Plugin Pipeline] section (FR-028)
- [ ] T0027 Implement real-time status updates — Settings UI subscribes to registry events (`service:status:changed`, `service:crash`, `service:bound`) and updates UI immediately (CH-009, FR-033)
- [ ] T0028 Handle service registration via `settingsRegistration` in `service.json` — render custom settings component if `configApp` is defined, otherwise show default config panel

**Checkpoint**: Settings UI shows all services, allows start/stop/restart, auto-saves config, displays logs, updates in real-time.

---

## Phase 5: Network & Dependencies

**Purpose**: Port configuration, runtime state separation, dependency health checks, and crash-loop prevention.

- [ ] T0029 Implement `runtime.json` management — service manager writes `{ "port": number, "host": string }` after worker publishes `bound` message. Dependent services read this file to discover binding info (CH-002 fix, FR-026)
- [ ] T0030 Implement crash-loop prevention — before starting a dependent service, check if declared dependencies are in `Running` state (via `DependencyResolver.checkDependenciesRunning`). If not, log warning and skip starting (CH-003 fix)
- [ ] T0031 Implement configurable host binding — worker reads `host` from config (default `localhost`), supports external interfaces (e.g., `0.0.0.0`)
- [ ] T0032 Implement startup ordering — topological sort from `DependencyResolver.resolveOrder()`, start services in order, log failures at `warn` level, continue starting next service (CH-007, FR-021-024)
- [ ] T0033 [P] Implement port conflict detection — `PortChecker.checkPortAvailable()` called before binding, log error if port in use, no auto-retry
- [ ] T0034 Implement manifest validation at start time — validate entrypoint can be loaded (`require()`), log clear error if fails, do NOT start service

**Checkpoint**: Dependencies start in correct order with crash-loop prevention. Runtime state separated from user config. Port conflicts handled cleanly.

---

## Phase 6: Migration & Cleanup (✅ Implemented)

**Purpose**: Migrate existing plugin system to hooks system and update marketplace schema.

- [x] T0035 Migrate `dataDir()/plugins/<id>/` → `dataDir()/user-apps/<id>/hooks/` — update existing installations
- [x] T0036 Rename `registerPlugin()` → `registerHook()` in all codebase references
- [x] T0037 Rename `plugin.json` → `hook.json` in all codebase references
- [x] T0038 Update all plugin-related terminology (pipeline → hooks, PluginRegistry → HookRegistry, etc.)
- [x] T0039 Update marketplace schema to support service items — add `services/` directory type to marketplace item structure
- [x] T0040 Update documentation (spec templates, user guides) to reference services instead of plugins where appropriate

**Checkpoint**: Plugin system fully renamed to hooks. Marketplace supports service items. Documentation is consistent.

---

## Phase 7: Testing (Not implemented — not yet started)

**Purpose**: Unit tests for core logic, integration tests for lifecycle, UI tests for Settings components.

- [ ] T0041 [P] Write unit tests for `ServiceRegistry` — discovery, source/installed state distinction, event emission
- [ ] T0042 [P] Write unit tests for `DependencyResolver` — topological sort, circular detection, missing detection, health check
- [ ] T0043 [P] Write unit tests for `CrashRecovery` — restart counter, backoff calculation, max restarts enforcement, counter reset on `initialized`
- [ ] T0044 [P] Write unit tests for `PortChecker` — port available/unavailable detection
- [ ] T0045 [P] Write integration tests for `ServiceManager` — start/stop/restart lifecycle, IPC message handling, worker thread creation
- [ ] T0046 Write integration tests for install/uninstall — symlink creation, manifest validation, registry update
- [ ] T0047 [P] Write UI tests for `ServicesTab` — render service list, click Start/Stop/Restart, verify status changes
- [ ] T0048 [P] Write UI tests for `ServiceConfigPanel` — auto-save on input change, read-only warning for marketplace items
- [ ] T0049 [P] Write UI tests for `ServiceLogViewer` — display log content, clear button works
- [ ] T0050 Run all tests, fix failures, verify coverage

**Checkpoint**: All unit and integration tests pass. UI tests verify component behavior.

---

## Phase 8: Polish & Cross-Cutting Concerns (✅ Implemented)

**Purpose**: Final integration, edge case handling, and documentation.

- [x] T0051 Implement error handling for permission denied during symlink creation — catch and report with actionable messages (FR-033 Error Handling)
- [x] T0052 Implement "corrupted" state — if `service.json` is deleted after install, mark service as corrupted, refuse to start until restored (FR-033 Error Handling). Fixed: recovered services now transition cleanly back to `"stopped"` with `corruptedReason` removed; event only fires when state actually changes.
- [x] T0053 Implement atomic config writes — use `writeFileAtomic` for all config writes (CH-009 Race Condition)
- [x] T0054 Validate `configSchema` at install time — if present, validate JSON syntax and schema structure (CH-011 extension)
- [x] T0055 Add startup ordering max retry limit — after 3 failed attempts to start a dependent service, log "giving up" and stop retrying (CH-013 fix). Implemented in `ServiceManager.startAll()` with 3-pass retry loop.
- [x] T0056 Document worker thread isolation limitation — clarify memory vs CPU isolation (CH-005 fix). Added comment at `new Worker(...)` in `ServiceManager.ts`.
- [x] T0057 Add boot sequence failure handling — if a service fails during boot, log warning and continue; boot completes when all services attempted (CH-008 fix). Already implemented in `startAll()`.
- [x] T0058 Update spec.md with final implementation notes — mark all requirements as implemented, add known limitations
- [x] T0059 Run stress test again to verify spec is ready for implementation
- [x] T0060 Final review of all tasks, ensure completeness

**Also added**: `GET /api/services/[id]` endpoint for single service detail including `corruptedReason`.

**Checkpoint**: Edge cases handled, race conditions prevented, documentation updated, stress test passes.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — can start immediately
- **Phase 2 (Registry)**: Depends on Phase 1 — requires manifest validation and symlink infrastructure
- **Phase 3 (Lifecycle)**: Depends on Phase 2 — requires registry for service discovery and dependency resolution
- **Phase 4 (Settings UI)**: Depends on Phase 3 — requires running services to manage
- **Phase 5 (Network & Dependencies)**: Depends on Phase 3 — requires worker thread management for runtime state
- **Phase 6 (Migration)**: Can run in parallel with Phases 3-4 — independent of core service logic
- **Phase 7 (Testing)**: Depends on all implementation phases — tests verify working code
- **Phase 8 (Polish)**: Depends on all phases — final integration and edge cases

### User Story Dependencies

- **User Story 1 (P1)**: Install terminal service — covered by Phases 1-3, 5
- **User Story 2 (P1)**: Manage lifecycle in Settings — covered by Phases 3-4
- **User Story 3 (P2)**: Configure ports — covered by Phases 3-5
- **User Story 4 (P2)**: View logs — covered by Phases 3-4
- **User Story 5 (P3)**: Configure crash recovery — covered by Phases 3-4

### Parallel Opportunities

- **Phase 1**: T005 (item structure validation) can run in parallel with T001-T004
- **Phase 2**: T006-T008 are sequential (registry depends on itself), T009-T010 can run in parallel
- **Phase 3**: T011-T013 are sequential (manager depends on itself), T014-T016-T017-T021 can run in parallel
- **Phase 4**: T022-T028 are largely sequential (UI components depend on each other), T024-T025 can run in parallel
- **Phase 5**: T029-T034 are sequential (runtime state depends on worker management), T033 can run in parallel
- **Phase 6**: T035-T040 can all run in parallel (independent renames/migrations)
- **Phase 7**: All test tasks (T0041-T0050) can run in parallel (different files)
- **Phase 8**: T051-T059 can run in parallel (independent edge cases)

---

## Implementation Strategy

### MVP First (User Story 1 + 2)

1. Complete Phase 1: Setup (manifest format, symlinks)
2. Complete Phase 2: Registry (discovery, dependency resolution)
3. Complete Phase 3: Lifecycle (worker threads, start/stop/restart, crash recovery)
4. **STOP and VALIDATE**: Install a terminal service, start it, stop it, restart it. Verify worker thread IPC works.
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Phase 1-3 → Core service management works (MVP!)
2. Add Phase 4 → Settings UI with real-time updates
3. Add Phase 5 → Dependencies and runtime state separation
4. Add Phase 6 → Migration and cleanup
5. Add Phase 7-8 → Testing and polish

### Parallel Team Strategy

With multiple developers:

1. Team completes Phase 1-2 together (foundation)
2. Once foundation is done:
   - Developer A: Phase 3 (core lifecycle)
   - Developer B: Phase 4 (Settings UI)
   - Developer C: Phase 6 (migration/renames)
3. After core is done:
   - Developer A: Phase 5 (network/dependencies)
   - Developer B: Phase 7 (testing)
   - Developer C: Phase 8 (polish)

---

## Notes

- **P** tasks = different files, no dependencies
- **Story** label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
- **CH-001 fix is critical**: Worker crash during `initialize` must be detected via `exit`/`error` events, not just `crash` message
- **CH-002 fix is critical**: `runtime.json` separates user config from runtime-discovered binding info
- **CH-003 fix is critical**: Crash-loop prevention checks dependency health before starting dependent services
