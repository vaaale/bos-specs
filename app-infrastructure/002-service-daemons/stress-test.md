# Stress Test Report: Service Daemons Spec

**Reviewed by**: Specification Reviewer  
**Date**: 2026-07-25  
**Spec Under Review**: `user-specs/002-service-daemons/spec.md`  
**Status**: DRAFT — requires clarification before implementation

---

## Executive Summary

The spec is structurally sound and covers the canonical happy paths well. However, it contains **five categories of critical issues** that would cause implementation surprises, race conditions, or architectural contradictions if implemented as written:

1. **Config path contradiction** — the two-layer config system has two different "runtime config" paths that conflict (`dataDir()/config/<id>.json` vs. `dataDir()/system/config/<id>`)
2. **Startup ordering circular dependency** — NFR-003 says services must initialize before the chat pipeline, but the chat pipeline depends on plugins, and the plugin system itself depends on the hooks mechanism that services provide. The boot sequence is circular.
3. **Missing concurrency controls for lifecycle** — only install/uninstall have a mutex; start/stop/restart/crash-restart can race against each other, creating undefined state
4. **Worker thread crash semantics are incomplete** — the spec says V8 GC kills are treated as crashes, but V8 kills can crash the entire Node.js process (not just the worker thread), contradicting NFR-002
5. **No mechanism for services to publish their actual runtime binding info** — port 0, dynamic ports, or post-bind port discovery have no defined protocol

---

## Challenges (Socratic Questions)

### CH-001: Two-Layer Config System — Path Contradiction

**Question**: The spec defines two config layers:
- **Layer 1** (template): `<item-id>/config/` directory — "Read-only when from marketplace, writable in user-apps"
- **Layer 2** (runtime): `dataDir()/config/<id>.json` — "This is what the service actually reads at runtime"

But the installation symlinks create `dataDir()/system/config/<id> -> dataDir()/user-apps/<id>/config` AND the uninstall removes `dataDir()/system/config/<id>`.

So which path is the "runtime config"? The spec says the service reads from `dataDir()/config/<id>.json`, but the symlink at `dataDir()/system/config/<id>` points to a *directory* (`user-apps/<id>/config/`), not a file. If the service reads config from `dataDir()/config/<id>.json`, why does the symlink at `dataDir()/system/config/<id>` exist at all — and what happens when someone reads from the symlink vs. the runtime path?

**Recommended Decision**: Eliminate the symlink at `dataDir()/system/config/<id>`. The config layer should be:
- `dataDir()/config/<id>.json` = runtime config (what the service reads)
- `<item-id>/config/` = template defaults (copied to runtime on install)

The symlink is redundant because `dataDir()/system/services/<id>` already points to the services directory. Config is not a "system" resource — it's user data. Removing this symlink eliminates the path contradiction.

**Risk if Unresolved**: Implementers will be confused about which path is authoritative. The symlink suggests config is a system resource, but the service reads from user data — creating a maintenance burden (who updates the symlink when the user-apps directory is recreated?).

---

### CH-002: Startup Sequence Circular Dependency

**Question**: NFR-003 states: "Service startup MUST be awaited before the chat pipeline starts." But the chat pipeline depends on plugins (compaction, memory), and the plugin system depends on the hooks mechanism (`registerHook()`). The hooks mechanism is provided by services (FR-033-35). 

This creates a **boot circular dependency**:
- Services need the hooks system → provided by BOS core (not a service)
- Chat pipeline needs services to be ready → NFR-003
- But plugins (memory, compaction) need to initialize → which themselves need the chat pipeline infrastructure to be running

How does the boot sequence resolve this? What if a service depends on a plugin that depends on a service?

**Recommended Decision**: Define a **phased boot sequence**:
1. **Phase 1 (Core)**: BOS core initializes, including the hooks registration system (`registerHook()`) — this is a *function*, not a service. Services use it but don't provide it.
2. **Phase 2 (Services)**: Service registry scans, dependency graph is built, services start in dependency order.
3. **Phase 3 (Plugins)**: Plugin registry loads, plugins initialize (they can now use the hooks system).
4. **Phase 4 (Pipeline)**: Chat pipeline starts — services are ready, plugins are registered.

This resolves the circular dependency by making the hooks system a *primitive* provided by the core, not a service-provided feature.

**Risk if Unresolved**: The boot sequence will deadlock or require arbitrary ordering hacks. If a service declares a dependency on a plugin that hasn't initialized yet, there's no defined failure path.

---

### CH-003: Missing Concurrency Controls for Lifecycle Operations

**Question**: The spec has a mutex for install/uninstall operations (Concurrency Control section). But what about concurrent **lifecycle operations** on the same service?

Consider these race conditions:
- User clicks "Start" and "Stop" simultaneously → does the service end up in an undefined state?
- Crash recovery triggers a restart while the user clicks "Stop" → who wins?
- A service is restarting (Stop → Start) and another lifecycle operation arrives → does it queue or overwrite?

The spec defines lifecycle states (Stopped, Running, Restarting) but doesn't define what happens when a state transition is interrupted by another operation.

**Recommended Decision**: Add a **per-service mutex** (or a global service manager lock) that serializes ALL operations on a given service:
- Start/Stop/Restart/Crash-restart ALL acquire the lock
- Operations queue behind the lock and are executed in order
- The "Restarting" state is atomic — no other operation can interrupt it

Additionally, define **operation rejection rules**:
- Stop while stopping → return "already stopping"
- Start while starting → return "already starting"
- Stop during crash recovery → queue the stop, cancel recovery on completion

**Risk if Unresolved**: Race conditions will produce inconsistent state (e.g., service marked "Running" but worker thread is terminated, or vice versa). This is a high-severity bug that will be hit frequently in real usage.

---

### CH-004: Worker Thread Crash Semantics — V8 Kills Can Crash the Process

**Question**: NFR-002 states: "A service crash MUST NOT crash BOS — worker threads have isolated memory heaps." But the spec also states: "If a worker is killed by the V8 GC due to memory limits, it is treated as a crash and crash recovery kicks in."

This is **technically incorrect**. When V8 kills a worker thread due to memory limits (`maxOldGenerationSizeMb`), it can crash the **entire Node.js process**, not just the worker thread. The `resourceLimits` setting only sets a *hint* — V8 may choose to kill the process if the limit is exceeded.

Additionally, worker threads share the Node.js event loop. If a worker thread enters an infinite loop (CPU-bound), it starves the entire process — there's no CPU isolation.

**Recommended Decision**:
1. **Replace `resourceLimits` with a child process model** for v1, or
2. **Use `worker.terminate()` aggressively** with a watchdog that monitors memory usage externally (e.g., via `process.memoryUsage()` on the main thread comparing against worker-specific tracking), or
3. **Explicitly document** that `resourceLimits` is a best-effort hint and crashes can still take down BOS, making NFR-002 a soft guarantee

The cleanest solution for v1 is **Option 1**: run services as child processes (via `child_process.fork()` or `spawn`) instead of worker threads. This gives true process isolation and makes NFR-002 a hard guarantee.

**Risk if Unresolved**: The spec makes a promise (NFR-002) that the implementation cannot guarantee with worker threads. A memory-leaking service will crash the entire BOS instance, violating the core design goal.

---

### CH-005: No Mechanism for Services to Publish Their Runtime Binding Info

**Question**: The spec says services publish their binding information (port, host) in their runtime config (`dataDir()/config/<id>.json`). But how does this work in practice?

1. **Port 0 support**: The spec says "Port 0 (random available port) is supported." But if a service binds to port 0, the OS assigns a random port. The service needs to **discover its actual port** after binding and write it back to the config. The spec doesn't define this protocol.

2. **Dynamic port assignment**: If the service discovers its port at runtime, who writes it back to the config file? The worker thread? The service manager? How does the config get updated?

3. **Non-port services**: Not all services bind ports. How do they communicate with consumers? The spec only addresses port-based discovery.

4. **Config write race**: If the service writes to `dataDir()/config/<id>.json` while the user is editing it in Settings, there's a race condition.

**Recommended Decision**:
1. **Add a postMessage protocol extension**: Workers send `{ type: 'bound'; port: number; host: string }` after binding. The service manager writes this to the runtime config.
2. **For port 0**: The service manager reads the actual port from the server after `server.listen(0)`, sends it back to the worker, and the worker uses that port.
3. **For non-port services**: Define a "service address" concept — services can publish any key-value pairs (not just ports) via the same bound protocol.
4. **Config write ownership**: Only the service manager writes to runtime config (not the worker thread). The worker thread reads config via the `initialize` message.

**Risk if Unresolved**: Services using port 0 will be unable to communicate their actual port to consumers. The config write race will cause corrupted config files.

---

### CH-006: Crash Recovery State Persistence

**Question**: The spec defines crash recovery with exponential backoff and a restart counter. But where is this state persisted?

1. **If BOS restarts** during crash recovery, what happens? Does the service attempt to restart immediately? Does it resume the backoff? Does it start fresh?
2. **The restart counter resets on `initialized`** — but what if the service crashes between the `dispose` response and the new `initialized`? Is the counter reset or not?
3. **Max restarts** — is this a hard limit (e.g., 5 total restarts ever) or a sliding window (e.g., 5 restarts within N minutes)?

**Recommended Decision**:
1. **Persist crash state** in `dataDir()/config/<id>.json` or a separate `dataDir()/state/services/<id>.json`:
   - `restartCount`
   - `lastCrashTimestamp`
   - `lastBackoffDuration`
2. **On BOS restart**: If a service was in crash recovery, resume from the persisted state (don't start fresh).
3. **Restart counter**: Hard limit (e.g., 5 total restarts), but reset on successful initialization. Document this clearly.
4. **Define the crash-to-restart gap**: The service must receive `initialized` BEFORE the counter resets. If it crashes during initialization, the counter does NOT reset.

**Risk if Unresolved**: Crash recovery state is lost on BOS restart, causing services to restart immediately after a crash instead of waiting for backoff. This creates a crash loop that prevents BOS from stabilizing.

---

### CH-007: Dependency Resolution — Missing Failure Handling

**Question**: The spec says "If a dependency fails to start, the dependent service MUST NOT start" (FR-022). But what about **runtime dependency failures**?

1. **What if a dependency crashes AFTER the dependent service has started?** Does the dependent service crash too? Does it enter a degraded state?
2. **What if a dependency's port changes** (e.g., due to a restart)? Does the dependent service reconnect?
3. **What if a dependency's config is deleted** while the dependent service is running?

The spec only handles **startup-time** dependency validation, not **runtime** dependency health.

**Recommended Decision**: Define a **runtime dependency health check**:
1. **Option A (simpler)**: If a dependency crashes, the dependent service is automatically stopped (cascading failure).
2. **Option B (more complex)**: Services watch their dependencies' status (via the service registry or file system) and enter a degraded state if a dependency is unavailable.
3. **Document the chosen approach** and add it to the spec.

**Risk if Unresolved**: Dependent services will continue running with broken dependencies, leading to inconsistent behavior and hard-to-debug errors.

---

### CH-008: Log Rotation — Missing Details

**Question**: The spec says "Log file is append-only, rotated daily" (Log capture section). But:

1. **What rotation strategy?** Size-based? Time-based? Both?
2. **What happens if rotation fails?** The spec says "Log rotation failures MUST be logged to a central log" but doesn't define what "central log" means in this context.
3. **How many rotated files are kept?** Is there a retention policy?
4. **What if the log directory doesn't exist?** Who creates it?

**Recommended Decision**:
1. **Use size-based rotation**: Rotate when log file exceeds 10 MB (configurable).
2. **Keep last 5 rotated files** (e.g., `terminal.log.1`, `terminal.log.2`, etc.).
3. **Log rotation failures** go to `dataDir()/logs/bos.log` (the central BOS log).
4. **Create log directory** on BOS startup if it doesn't exist.

**Risk if Unresolved**: Log files will grow unbounded, potentially filling the disk. Implementation will be inconsistent across services.

---

### CH-009: Settings UI — Missing Real-time Update Mechanism

**Question**: NFR-006 says "Service status changes MUST be reflected in the Settings UI in real-time." But:

1. **How does the worker thread communicate status changes** to the UI? The spec defines `postMessage` for initialize/dispose, but not for status updates.
2. **Is there a polling mechanism?** If the UI polls the service registry, what's the interval?
3. **What if the UI is closed** when a status change occurs? Does it miss the update?

**Recommended Decision**: Add a **status event protocol** to the `postMessage` messages:
```typescript
type WorkerToMainMessage =
  | { type: 'status'; status: 'running' | 'stopped' | 'crashed' }
  | ...;
```
The service manager forwards these events to the Settings UI via a WebSocket or SSE (Server-Sent Events) connection.

**Risk if Unresolved**: The Settings UI will be out of sync with actual service state, leading to confusing user experiences (e.g., button says "Start" but service is already running).

---

### CH-010: Hooks Integration — Ambiguous Relationship with Plugins

**Question**: The spec says "Hooks MUST be registered via `registerHook()` (not `registerPlugin()`)" (FR-034). But the existing plugin system (026-plugin-pipeline) uses `registerPlugin()` and `plugin.json` manifests.

1. **Are service hooks and plugins the same thing?** The spec seems to treat them as separate (services have `hooks/`, plugins have `plugin.json`).
2. **How do service hooks interact with plugin hooks?** If a service provides hooks via `hooks/` and a plugin provides hooks via `plugin.json`, do they both register with the same `registerHook()` system?
3. **The migration note** says "Existing `dataDir()/plugins/<id>/` will be migrated to `dataDir()/user-apps/<id>/hooks/`" — this suggests they ARE the same, but the spec doesn't clarify the relationship.

**Recommended Decision**: Clarify the relationship:
- **Service hooks** = hooks provided by services (via `hooks/` directory)
- **Plugin hooks** = hooks provided by plugins (via `plugin.json` manifest)
- Both use the same `registerHook()` system
- The migration is renaming `plugins/<id>/` to `user-apps/<id>/hooks/` — the underlying mechanism is the same

**Risk if Unresolved**: Implementers will be confused about whether to use `registerHook()` or `registerPlugin()` for service hooks. The migration path is unclear.

---

### CH-011: Service Manifest Validation — Incomplete Error Handling

**Question**: The spec says "Validate required fields (id, name, version, entry)" during installation. But:

1. **What if `entry` is a relative path that resolves to a non-existent file?** The spec says "If the service entrypoint file does not exist, installation MUST fail" but doesn't say when this check happens (install time or start time?).
2. **What if the entrypoint file exists but is malformed JS?** The spec doesn't define how to validate the entrypoint code.
3. **What if `configSchema` is invalid JSON?** The spec says "Validate configSchema if present" but doesn't define what "valid" means (valid JSON? Valid JSON Schema?).

**Recommended Decision**:
1. **Validate entrypath at install time** (file must exist) AND **validate entrypoint loadability at start time** (try `require()` or `import()` and catch errors).
2. **Validate `configSchema` at install time** as valid JSON and valid JSON Schema (use a schema validator library).
3. **Define clear error messages** for each validation failure.

**Risk if Unresolved**: Broken entrypoints will be detected late (at start time), causing confusing errors for users. Invalid schemas will cause runtime crashes.

---

### CH-012: Port Conflict Detection — Race Condition

**Question**: The spec defines a `checkPortAvailable()` function that creates a temporary server to check if a port is free. But:

1. **What if the port becomes free between the check and the actual bind?** The spec doesn't handle this race condition.
2. **What if the service binds to port 0?** The spec says this is supported but doesn't define the protocol for discovering the actual port.

**Recommended Decision**:
1. **Accept that port conflicts can still occur** after the check. The service manager should catch the bind error and return a clear message: "Port 3001 became unavailable between check and bind. Try again or use a different port."
2. **For port 0**: Define the protocol in CH-005 (see CH-005 recommendation).

**Risk if Unresolved**: Users will see confusing "port in use" errors even when they just checked the port and it was free.

---

### CH-013: Service Registry — Source vs. Installed State Ambiguity

**Question**: FR-006 says the registry scans "both source and installed states" for discovery. But:

1. **What's the difference between source and installed states?** The spec mentions `dataDir()/user-apps/` (source) and `dataDir()/system/services/` (installed), but doesn't define the relationship.
2. **Can a service be in source state but not installed?** If so, what happens when the user tries to start it?
3. **What if a service is uninstalled but the source still exists?** The spec says the marketplace clone is NOT deleted on uninstall, but the user-apps entry is. So the source still exists — does the registry still discover it?

**Recommended Decision**:
1. **Define clearly**: Source = available to install, Installed = currently installed and available to start.
2. **Source-only services** can be started (they're effectively installed on demand).
3. **Uninstalled services** are removed from the installed state but remain in source (if from marketplace).

**Risk if Unresolved**: The registry will have undefined behavior for source-only or uninstalled services. Users will be confused about what they can and can't start.

---

### CH-014: Dependency Order — Missing Self-Dependency Check at Startup

**Question**: FR-025 says "Self-dependencies MUST be rejected at install time." But:

1. **What if a service dynamically adds itself as a dependency at runtime?** The spec doesn't prevent this.
2. **What if the dependency graph changes** (e.g., a dependency is uninstalled and reinstalled with different dependencies)?

**Recommended Decision**:
1. **Validate dependency graph at startup time** (not just install time).
2. **Rebuild dependency graph** if any service is uninstalled or reinstalled.

**Risk if Unresolved**: Dynamic dependency changes will cause undefined behavior.

---

### CH-015: Worker Thread — No Heartbeat Mechanism

**Question**: The spec defines a 30-second timeout for `dispose` but doesn't define a **heartbeat mechanism** to detect hung services.

1. **What if a service is responsive but slow?** The 30-second timeout will force-kill it unnecessarily.
2. **What if a service enters an infinite loop?** The spec says worker threads are isolated, but an infinite loop will starve the event loop.

**Recommended Decision**:
1. **Add a heartbeat mechanism**: Workers send periodic `{ type: 'heartbeat' }` messages. If the main process doesn't receive a heartbeat for N seconds, the service is considered hung.
2. **Define heartbeat interval** (e.g., every 5 seconds).
3. **Define hang timeout** (e.g., 10 seconds without heartbeat = force kill).

**Risk if Unresolved**: Slow but healthy services will be force-killed. Hung services will starve the process.

---

## Risks and Gaps Summary

| # | Category | Risk Level | Description |
|---|----------|------------|-------------|
| 1 | Config Path | **HIGH** | Two conflicting "runtime config" paths (`dataDir()/config/<id>.json` vs `dataDir()/system/config/<id>`) |
| 2 | Boot Sequence | **HIGH** | Circular dependency between services, plugins, and chat pipeline |
| 3 | Concurrency | **HIGH** | No mutex for lifecycle operations; race conditions will produce inconsistent state |
| 4 | Worker Isolation | **HIGH** | V8 memory limit kills can crash the entire process, violating NFR-002 |
| 5 | Port Discovery | **MEDIUM** | No protocol for services to publish dynamic ports (port 0) |
| 6 | Crash State | **MEDIUM** | Crash recovery state not persisted; lost on BOS restart |
| 7 | Runtime Dependencies | **MEDIUM** | No handling for dependency failures after service startup |
| 8 | Log Rotation | **LOW** | Missing rotation strategy, retention policy, failure handling |
| 9 | UI Updates | **MEDIUM** | No mechanism for real-time status updates to Settings UI |
| 10 | Hooks/Plugins | **LOW** | Ambiguous relationship between service hooks and plugin hooks |
| 11 | Manifest Validation | **LOW** | Incomplete error handling for malformed entrypoints/schemas |
| 12 | Port Race | **LOW** | Race condition between port check and bind |
| 13 | Registry Ambiguity | **MEDIUM** | Unclear what "source state" vs "installed state" means for discovery |
| 14 | Dynamic Deps | **LOW** | No validation of dependency graph changes at runtime |
| 15 | Heartbeat | **MEDIUM** | No mechanism to detect hung services (only timeout for dispose) |

---

## Suggested Spec Changes

### Change 1: Eliminate `dataDir()/system/config/<id>` Symlink

**Where**: Installation Process, Uninstallation Process, FR-004/FR-005

**Change**: Remove the symlink creation/removal for `dataDir()/system/config/<id>`. Config is user data, not a system resource.

**Justification**: Eliminates the path contradiction (CH-001) and simplifies the config system.

---

### Change 2: Define Phased Boot Sequence

**Where**: New section "Boot Sequence" before "Requirements"

**Change**: Add a phased boot sequence:
1. Core initializes (hooks system, service registry)
2. Services start (in dependency order)
3. Plugins initialize
4. Chat pipeline starts

**Justification**: Resolves the circular dependency (CH-002) by making hooks a primitive, not a service feature.

---

### Change 3: Add Per-Service Mutex for Lifecycle Operations

**Where**: Concurrency Control section

**Change**: Add a per-service mutex (or global lock) that serializes all lifecycle operations (Start/Stop/Restart/Crash-restart). Define operation rejection rules.

**Justification**: Prevents race conditions (CH-003) that will produce inconsistent state.

---

### Change 4: Replace Worker Threads with Child Processes

**Where**: NFR-001, FR-009, Worker Thread IPC Protocol section

**Change**: Change "Services MUST run as worker threads" to "Services SHOULD run as child processes (via `child_process.fork()` or `spawn`) for v1." Update IPC protocol to use `process.send()`/`process.on('message')`.

**Justification**: Provides true process isolation (CH-004), making NFR-002 a hard guarantee.

**Alternative**: If worker threads are preferred, explicitly document that `resourceLimits` is a best-effort hint and crashes can still take down BOS.

---

### Change 5: Add Post-Bind Port Discovery Protocol

**Where**: Worker Thread IPC Protocol section, Port Conflict Detection section

**Change**: Add new message types:
```typescript
type WorkerToMainMessage =
  | { type: 'bound'; port: number; host: string }
  | ...;
```
The service manager writes the bound port to the runtime config.

**Justification**: Enables port 0 support and dynamic port discovery (CH-005, CH-012).

---

### Change 6: Persist Crash Recovery State

**Where**: Crash Recovery section

**Change**: Define crash state persistence in `dataDir()/state/services/<id>.json`:
- `restartCount`
- `lastCrashTimestamp`
- `lastBackoffDuration`

On BOS restart, resume from persisted state.

**Justification**: Prevents crash loops (CH-006).

---

### Change 7: Define Runtime Dependency Health

**Where**: Dependencies section

**Change**: Define behavior when a dependency crashes after the dependent service has started:
- **Option A**: Cascading stop (dependent service is also stopped)
- **Option B**: Degraded state (dependent service continues but logs warnings)

**Justification**: Handles runtime dependency failures (CH-007).

---

### Change 8: Define Log Rotation Strategy

**Where**: Log capture section

**Change**: Specify:
- Size-based rotation (10 MB)
- Keep last 5 rotated files
- Rotation failures go to `dataDir()/logs/bos.log`
- Create log directory on startup

**Justification**: Prevents unbounded log growth (CH-008).

---

### Change 9: Add Status Event Protocol

**Where**: Worker Thread IPC Protocol section

**Change**: Add status event messages:
```typescript
type WorkerToMainMessage =
  | { type: 'status'; status: 'running' | 'stopped' | 'crashed' }
  | ...;
```
The service manager forwards these to the Settings UI via WebSocket/SSE.

**Justification**: Enables real-time UI updates (CH-009).

---

### Change 10: Clarify Hooks vs. Plugins Relationship

**Where**: Hooks Integration section

**Change**: Add a clarification:
- Service hooks = hooks provided by services (via `hooks/` directory)
- Plugin hooks = hooks provided by plugins (via `plugin.json` manifest)
- Both use the same `registerHook()` system
- Migration: `plugins/<id>/` → `user-apps/<id>/hooks/` (same mechanism, different location)

**Justification**: Resolves ambiguity (CH-010).

---

### Change 11: Enhance Manifest Validation

**Where**: Installation Process, Error Handling section

**Change**: 
- Validate entrypoint file exists at install time
- Validate entrypoint loadability at start time
- Validate `configSchema` as valid JSON Schema at install time
- Define clear error messages for each failure

**Justification**: Catches errors early (CH-011).

---

### Change 12: Clarify Service Registry States

**Where**: Service Registry section, FR-006

**Change**: Define clearly:
- Source state = available to install
- Installed state = currently installed and available to start
- Source-only services can be started (installed on demand)
- Uninstalled services remain in source (if from marketplace)

**Justification**: Removes ambiguity (CH-013).

---

### Change 13: Add Heartbeat Mechanism

**Where**: Worker Thread IPC Protocol section

**Change**: Add heartbeat protocol:
```typescript
type WorkerToMainMessage =
  | { type: 'heartbeat' }
  | ...;
```
Workers send heartbeat every 5 seconds. If no heartbeat for 10 seconds, force kill.

**Justification**: Detects hung services (CH-015).

---

## Priority of Changes

| Priority | Changes | Rationale |
|----------|---------|-----------|
| **P0 (Blocker)** | CH-001, CH-002, CH-003, CH-004 | These are architectural contradictions or missing mechanisms that will cause implementation to fail or produce incorrect behavior. |
| **P1 (High)** | CH-005, CH-006, CH-007, CH-009 | These are missing mechanisms that will cause runtime issues. |
| **P2 (Medium)** | CH-008, CH-011, CH-012, CH-013, CH-015 | These are edge cases or incomplete specifications. |
| **P3 (Low)** | CH-010, CH-014 | These are ambiguities that can be resolved during implementation. |

---

## Conclusion

The spec is a strong foundation but needs **four critical changes** before implementation can begin:

1. **Eliminate the config path contradiction** (CH-001)
2. **Define a phased boot sequence** (CH-002)
3. **Add concurrency controls for lifecycle** (CH-003)
4. **Replace worker threads with child processes** (CH-004) or explicitly document the isolation limitation

Without these changes, the implementation will either deadlock, produce inconsistent state, or crash the entire BOS process — all of which are high-severity bugs that will be very difficult to fix retroactively.

The remaining 11 challenges are important but can be addressed during implementation with careful attention to the recommended decisions.
