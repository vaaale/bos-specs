# Implementation Plan: Service Daemons

**Branch**: `bos/plugin-system-services` | **Date**: 2026-07-25 | **Spec**: `user-specs/002-service-daemons/spec.md`

**Input**: Feature specification for service daemons — long-running background processes as worker threads, installable from marketplace, with crash recovery and Settings UI integration.

## Summary

This feature extends BOS's item system to support **service daemons** — long-running background processes that run as worker threads within the BOS Node.js process. Services are installable from the marketplace, have configurable ports, crash recovery with exponential backoff, startup ordering with dependency support, and a full lifecycle management UI in Settings → Plugins → [Services].

The implementation adds:
1. A `services/` directory structure within items with `service.json` manifests
2. A Service Registry that tracks source (available) and installed (runtime) states
3. A Service Manager that creates/controls worker threads with IPC via `postMessage`
4. Crash recovery with configurable retry policy
5. Settings UI integration with real-time status, auto-saving config, and log viewer
6. Dependency resolution with crash-loop prevention

## Technical Context

**Language/Version**: TypeScript 5.x, Node.js 20+ (for worker threads with resource limits)

**Primary Dependencies**:
- `node:worker_threads` — Worker thread management with `resourceLimits`
- `node:fs` — Config file I/O (read/write with atomic writes for race condition safety)
- `node:net` — Port availability checking and network binding
- Existing BOS infrastructure: AppRegistry, Marketplace, Settings app, VFS layer

**Storage**:
- `dataDir()/user-apps/<id>/` — User marketplace (source items)
- `dataDir()/marketplace/<id>/items/<id>/` — Marketplace items (read-only source)
- `dataDir()/system/services/<id>/` — Symlinks to active services
- `dataDir()/config/<id>/` — Config directory (symlinked, contains user config + runtime.json)
- `dataDir()/logs/services/<id>.log` — Service stdout/stderr logs

**Testing**:
- Unit tests for: service registry, dependency resolution, crash recovery logic
- Integration tests for: install/uninstall lifecycle, worker thread IPC, config file I/O
- UI tests for: Settings panels (start/stop/restart, config auto-save, log viewer)

**Target Platform**: Node.js server process (BOS backend) + Electron/Tauri renderer (Settings UI)

**Project Type**: BrowserOS system feature (server-side service management + renderer UI)

**Performance Goals**:
- Service startup: < 5s for typical services (terminal WebSocket)
- Crash recovery: < 20s from crash to restart attempt (configurable backoff)
- Settings UI updates: real-time (< 100ms from state change to UI update)
- Memory per worker: ≤ 256 MB (enforced by resourceLimits)

**Constraints**:
- Worker threads share the Node.js process — no container isolation for v1
- CPU isolation is NOT provided — a runaway worker can starve the process
- Services cannot directly export functions — all communication via `postMessage`
- Config directory symlink must point to user-apps config (not a single JSON file)
- The plugin system rename (plugins → hooks) is a prerequisite for hooks integration

**Scale/Scope**: ~2000 LOC for backend, ~1500 LOC for Settings UI, ~500 LOC for tests

## Constitution Check

**Gate: Must pass before Phase 0 research. Re-check after Phase 1 design.**

| Principle | Status | Notes |
|-----------|--------|-------|
| Spec-first development | ✅ | Spec completed with 2 rounds of stress testing |
| Worker threads (no containers) | ✅ | User confirmed CH-004 — worker threads only for v1 |
| Config directory (not single JSON) | ✅ | User confirmed `dataDir()/config/<id>` is a directory symlink |
| Hooks replace plugins | ✅ | Prerequisite documented in spec |
| Real-time status in UI | ✅ | Event-based broadcast mechanism defined |
| Auto-save config (no Save button) | ✅ | Config changes written directly to config files |

## Project Structure

### Documentation (this feature)

```text
specs/002-service-daemons/
├── spec.md              # Feature specification
├── plan.md              # This file
├── tasks.md             # Implementation tasks
└── stress-test.md       # Stress test report (Round 2)
```

### Source Code (BOS repository)

```text
src/
├── core/
│   └── service/
│       ├── ServiceRegistry.ts        # Discovers/manages service lifecycle
│       ├── ServiceManager.ts         # Creates/controls worker threads
│       ├── CrashRecovery.ts          # Configurable restart policy with backoff
│       ├── DependencyResolver.ts     # Topological sort, circular/missing detection
│       ├── PortChecker.ts            # Port availability checking
│       └── types.ts                  # TypeScript types (ServiceManifest, etc.)
├── settings/
│   └── plugins/
│       └── ServicesTab.tsx           # Settings → Plugins → [Services] UI
│       ├── ServiceCard.tsx           # Individual service card component
│       ├── ServiceConfigPanel.tsx    # Auto-saving config editor
│       └── ServiceLogViewer.tsx      # Logs display component
└── system/
    └── marketplace/
        └── install/
            ├── serviceInstaller.ts   # Install/uninstall service items
            └── symlinkManager.ts     # Symlink creation/removal
```

**Structure Decision**: Backend logic in `src/core/service/` (server-side, no UI dependencies). Settings UI in `src/settings/plugins/ServicesTab.tsx` (React components, integrates with existing Settings app). Marketplace integration in `src/system/marketplace/` (extends existing install/uninstall flow).

## Complexity Tracking

No complexity violations — worker threads with resource limits are simpler than child processes or containers. The trade-off is documented in the spec (CPU isolation limitation).

## Implementation Phases

### Phase 1: Service Item Structure (Foundation)
- Define `service.json` manifest format (complete in spec)
- Create directory layout for service items
- Implement symlink creation/removal for service installation
- Validate manifests at install time

### Phase 2: Service Registry
- Implement `ServiceRegistry` with source + installed state distinction
- Load manifests from `dataDir()/system/services/<id>/service.json`
- Implement dependency resolution (topological sort)
- Expose registry API for lifecycle management

### Phase 3: Service Lifecycle (Core)
- Implement `ServiceManager` with worker thread creation
- Implement IPC protocol (postMessage `initialize`/`dispose`/`bound`/`crash`)
- Implement Start/Stop/Restart operations
- Implement crash recovery with exponential backoff
- Handle worker exit/error events (CH-001 fix)

### Phase 4: Settings UI
- Add [Services] section to Settings → Plugins
- Show service list with real-time status indicators
- Implement Start/Stop/Restart buttons
- Implement auto-saving config panel
- Implement logs viewer
- Implement status event broadcasting (CH-009)

### Phase 5: Network & Dependencies
- Implement configurable port in config files
- Implement `runtime.json` for binding information (CH-002 fix)
- Implement crash-loop prevention (CH-003 fix)
- Implement dependency health checks before starting dependent services

### Phase 6: Migration & Cleanup
- Migrate existing `dataDir()/plugins/<id>/` to `dataDir()/user-apps/<id>/hooks/`
- Rename `registerPlugin()` → `registerHook()`, `plugin.json` → `hook.json`
- Update all plugin-related terminology across the codebase
- Update marketplace schema to support service items

## Success Criteria

- Terminal service installable from marketplace, starts, binds to port, accessible via WebSocket
- Services start/stop/restart from Settings UI with real-time status updates
- Config changes auto-save, take effect on restart
- Crashing services auto-restart up to max restarts with exponential backoff
- Services start in dependency order with crash-loop prevention
- Worker crashes during `initialize` are detected and trigger recovery (no silent failures)
