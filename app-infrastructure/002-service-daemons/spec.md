# Feature Specification: Service Daemons

**Feature Branch**: `bos/plugin-system-services`

**Created**: 2026-07-25

**Status**: Implemented (backend) — see "Implementation Status" below

**Input**: User description: "Extend BOS plugin system to support system services/daemons installable through the marketplace. Services are long-running background processes (e.g., terminal WebSocket daemon) that run as worker threads with crash recovery, configurable ports, and startup ordering."

## Overview

This spec defines the **Service Daemons** architecture for BrowserOS, which enables long-running background processes as installable marketplace items. Services run as worker threads with automatic crash recovery and integrate into the Settings UI for lifecycle management.

### Goals

- **Long-running services**: BOS can run background daemons (terminal, monitoring, etc.) as worker threads
- **Marketplace integration**: Services are installable from existing marketplace infrastructure
- **Crash recovery**: Automatic restart with configurable retry policy and backoff
- **Settings integration**: Services appear in Settings → Plugins → [Services] section
- **Consistent item structure**: Services follow the same layout as apps, specs, and hooks

### Non-Goals

- **Client-side services** (browser extensions) — out of scope
- **Container-based services** (Docker) — worker threads only for v1
- **Service mesh** (Kubernetes) — no orchestration layer

## User Scenarios & Testing

### User Story 1 - I can install a terminal service from the marketplace (Priority: P1)

The user installs the Terminal service from the marketplace and starts it.

**Why this priority**: This is the canonical use case — demonstrates the full service lifecycle from install to running.

**Independent Test**: Install the Terminal service, start it, connect to its WebSocket endpoint, and verify the terminal app can communicate with it.

**Acceptance Scenarios**:

1. **Given** a marketplace with Terminal service, **When** the user installs it, **Then** symlinks are created in `dataDir()/system/services/terminal` and `dataDir()/config/terminal` (directory).
2. **Given** the Terminal service is installed, **When** the user starts it, **Then** a worker thread is created and the service binds to a configurable port (default: 3001).
3. **Given** the Terminal service is running, **When** the terminal app reads port from `dataDir()/config/terminal/terminal.json`, **Then** the WebSocket connection is established and shell commands execute.

### User Story 2 - I can manage service lifecycle in Settings (Priority: P1)

The user opens Settings → Plugins → [Services] and manages service lifecycle.

**Why this priority**: Users need a consistent interface to start, stop, restart, and configure services.

**Independent Test**: Start, stop, and restart a service from Settings UI and verify the lifecycle state changes correctly.

**Acceptance Scenarios**:

1. **Given** services are installed, **When** the user opens Settings → Plugins → [Services], **Then** they see a list of services with status indicators (running/stopped).
2. **Given** a service is stopped, **When** the user clicks Start, **Then** the service starts and the status changes to "Running".
3. **Given** a service is running, **When** the user clicks Stop, **Then** the service stops and the status changes to "Stopped".
4. **Given** a service is running, **When** the user clicks Restart, **Then** the service restarts and the status briefly shows "Restarting".

### User Story 3 - I can configure service ports (Priority: P2)

The user changes the port a service binds to from Settings.

**Why this priority**: Users may need to avoid port conflicts or run multiple service instances.

**Independent Test**: Change the port in Settings, restart the service, and verify it binds to the new port.

**Acceptance Scenarios**:

1. **Given** a service with default port 3001, **When** the user changes it to 3002 in Settings, **Then** `dataDir()/config/terminal/terminal.json` is updated with `port: 3002` (auto-saved).
2. **Given** a service with port 3002, **When** the service restarts, **Then** it binds to port 3002 (not 3001).
3. **Given** a service with port 3002, **When** the user changes it back to 3001, **Then** `dataDir()/config/terminal/terminal.json` is updated with `port: 3001` (auto-saved).

### User Story 4 - I can see service logs in Settings (Priority: P2)

The user views service logs (stdout/stderr) from Settings.

**Why this priority**: Users need to debug service issues and monitor service health.

**Independent Test**: Trigger a service error, check that the error appears in the Settings UI logs panel.

**Acceptance Scenarios**:

1. **Given** a service that writes to stderr, **When** the error occurs, **Then** the error message appears in the Settings logs panel.
2. **Given** a service that writes to stdout, **When** the service starts, **Then** the startup message appears in the Settings logs panel.
3. **Given** service logs, **When** the user clears logs, **Then** the logs panel is empty.

### User Story 5 - I can configure crash recovery settings (Priority: P3)

The user adjusts the service restart policy (max restarts, backoff time).

**Why this priority**: Advanced users may want to tune crash recovery behavior.

**Independent Test**: Set max restarts to 3, trigger a crash, verify the service restarts 3 times then stops.

**Acceptance Scenarios**:

1. **Given** default settings (max restarts: 5, backoff: exponential), **When** a service crashes, **Then** it restarts up to 5 times with exponential backoff.
2. **Given** custom settings (max restarts: 3), **When** a service crashes 4 times, **Then** it stops after 3 restarts.
3. **Given** a service with crashes, **When** the service recovers, **Then** the restart counter resets to 0.

### Edge Cases

- What happens when a service crashes during shutdown?
- How does the system handle port conflicts (port already in use)?
- What happens if a service's config file is malformed?
- How does the system handle service dependencies that fail to start? (Continue starting the next service, always log.)

## Requirements

### Functional Requirements

#### Complete Item Directory Structure

Every service item — whether from a marketplace, user-apps (local marketplace), or the Terminal app itself — follows the same directory structure. This ensures consistency across all item sources.

```
<item-id>/
├── spec/           → Spec template (read-only when from external marketplace, modifiable in user-apps)
│                     If the user wants to modify a marketplace spec, they adopt/fork it to
│                     their own specs/user-specs/<id>/ directory.
│                     If the item is in user-apps/, the user can edit spec/ directly.
│
├── doc/            → Documentation files (markdown, etc.)
│                     Read by the docs provider to surface in the docs app.
│
├── app/            → Application code (React components, etc.)
│                     Read by the AppRegistry to make available in the app launcher.
│
├── services/       → Service daemon definitions
│                     Contains service.json manifest that describes the long-running daemon.
│                     Read by the ServiceRegistry to register the service.
│
├── settings/       → Settings UI components (React components)
│                     React components that integrate into the Settings app's sidebar.
│                     Registered via settingsRegistration in service.json.
│
├── config/         → Configuration files (always present, may be empty)
│                     Contains default/template config files (any number, determined by the item).
│                     Read-only when from marketplace (external), writable in user-apps.
│                     On install, this directory is symlinked to dataDir()/config/<id>.
│
├── hooks/          → Hook-based plugin definitions (optional)
│                     CommonJS modules that register hooks via registerHook().
│                     Same format as existing hook-based plugins.
│                     Renamed from "pipeline/" to better describe the purpose.
│
└── (other/)        → Future item types can extend this structure
```

**Key principles**:
- The `config/` directory MUST always be present (even if empty) to avoid "directory does not exist" errors.
- The `hooks/` directory is optional — only included if the item provides hook-based plugins.
- All directories follow the same structure regardless of source (marketplace vs user-apps).

#### Item-to-System Symlink Mapping

When an item is "installed," symlinks are created from `dataDir()/system/` and `dataDir()/config/` to point at the item's directories:

```
# Item source (uninstalled / available to install)
dataDir()/user-apps/                     ← a marketplace repo, structurally identical to any other
├── marketplace.json                     ← BOS-maintained manifest (034)
└── items/<item-id>/
    ├── spec/
    ├── doc/
    ├── app/
    ├── services/
    ├── settings/
    ├── config/
    │   ├── terminal.json    ← default/template config files (determined by item)
    │   └── monitoring.json  ← (optional, determined by item)
    └── hooks/

# Installed state — ONE symlink per item (035-install-by-symlink)
dataDir()/system/
├── <item-id>  -> the item directory, wherever it lives:
│                   dataDir()/marketplace/<mktId>/items/<item-id>   (a marketplace item)
│                or dataDir()/user-apps/items/<item-id>             (the user's own)
└── config/<item-id>/   ← REAL directory, seeded from the item's config/ defaults
```

> **Superseded by `bos-system-specs/035-install-by-symlink`.** Earlier revisions
> specified up to six symlinks per item (`system/app`, `system/services`,
> `system/hooks`, `system/settings`, `config/<id>`, `specs/external-specs/<id>`,
> `docs/external-docs/<id>`) plus a **copy** of the item into `user-apps/`.
> Installing now copies nothing: it creates one symlink and seeds config.
> Facets are found by a depth-2 scan of `dataDir()/system/` — `<id>/app/`,
> `<id>/services/service.json`, `<id>/plugin/bos-plugin.json`, `<id>/spec/`,
> `<id>/hooks/` — through a single shared scanner. Uninstalling is one `rm`.
>
> Two consequences worth stating plainly: a `git pull` on a marketplace updates
> every item installed from it with no reinstall; and `user-apps` no longer
> accumulates copies of other people's items, which matters because it is a
> repository the user may publish (034).

> **Layout parity is the whole point** — and an earlier revision of this spec
> broke it while asserting it. It claimed user-apps has the "same layout as
> marketplace items" but then specified `user-apps/<item-id>/` against
> `marketplace/<id>/items/<item-id>/`. The implementation followed the paths,
> not the claim, and the two diverged: the local marketplace was auto-scanned
> flat with a synthesized in-memory manifest, while every other marketplace was
> read from an on-disk `marketplace.json` with items under `items/`. See
> `bos-system-specs/034-user-apps-marketplace-parity` for the corrected model.

**Where each lives**:
- `dataDir()` = `/workspace/data/` (VFS user data) — NOT in git, NOT in `src/`
- `dataDir()/user-apps/items/<item-id>/` = an item in the user's own marketplace — **byte-for-byte the same layout** as an item in any external marketplace, so the same repository can serve as either (034)
- `dataDir()/marketplace/<market-place-id>/items/<item-id>/` = marketplace items cloned from external repos
- `dataDir()/system/<type>/<id>` = symlinks pointing to user-apps (installed state)
- `dataDir()/config/<id>` = symlink to the item's `config/` directory (read by services at runtime)
- `dataDir()/specs/external-specs/<id>` = spec templates (read-only for external, modifiable for user-apps)
- `dataDir()/docs/external-docs/<id>` = documentation (read-only for external, modifiable for user-apps)

**Config directory semantics**:
- `dataDir()/config/<id>` is a symlink pointing to `dataDir()/user-apps/<id>/config/`.
- The config directory contains whatever config files the item defines (e.g., `terminal.json`, `monitoring.json`). The number and names of config files are determined by the item.
- The service reads its configuration from `dataDir()/system/config/<id>/`.
- When the user changes a config file in Settings, the change is written directly to the file under `dataDir()/config/<id>/`.
- Config is always writable: it is BOS-owned state under `dataDir()/system/config/<id>/`, never the item's packaged defaults (035 FR-004).
- **Separation of user config and runtime state** (CH-002 fix): User-configurable values live in the item's config files (e.g., `terminal.json`). Runtime-discovered values (e.g., the actual bound port when `port: 0`) live in a separate `runtime.json` file under `dataDir()/config/<id>/`. The service manager writes `runtime.json`; users never edit it. The Settings UI reads both files: user values from the item config files, and app binding info from `runtime.json`.

#### Service `service.json` Manifest

The `services/<id>/service.json` manifest declares the service daemon:

```json
{
  "id": "terminal",
  "name": "Terminal Service",
  "version": "1.0.0",
  "description": "WebSocket-based terminal service for shell access",
  "entry": "./index.js",
  "configSchema": {
    "type": "object",
    "properties": {
      "port": { "type": "number", "default": 3001, "description": "WebSocket port" },
      "shell": { "type": "string", "default": "/bin/bash", "description": "Shell executable" }
    }
  },
  "dependencies": [],
  "settingsRegistration": {
    "label": "Terminal",
    "icon": "terminal",
    "order": 100,
    "configApp": "terminal-config"
  }
}
```

**Manifest fields**:
- `id`: Service identifier (must match directory name)
- `name`: Human-readable name
- `version`: Semantic version
- `description`: Service description
- `entry`: Path to the service entrypoint (JS/TS file relative to services/ directory)
- `configSchema`: Optional JSON Schema for configuration (used to render Settings UI)
- `dependencies`: Optional array of service IDs that must start before this one
- `settingsRegistration`: Optional registration for Settings UI (label, icon, order, configApp)

#### Installation Process (Server-Side)

When a user installs a service from the marketplace or from their local user-apps, the following server-side operations occur:

**Step 1: Clone/Copy the item to user-apps**
```
dataDir()/user-apps/<item-id>/
```
- If from marketplace: clone the marketplace repo to `dataDir()/marketplace/<market-place-id>/items/<item-id>/`
- If from user-apps: the item already exists in `dataDir()/user-apps/<item-id>/`
- The item is NOT copied to `dataDir()/system/` — it stays in user-apps or marketplace

**Step 2: Create symlinks**
```bash
# Create dataDir()/system/<type>/<id> symlinks
ln -s <absolute-path-to-user-apps>/<item-id>/services dataDir()/system/services/<id>

# Create config directory symlink (NOT a single JSON file)
ln -s <absolute-path-to-user-apps>/<item-id>/config dataDir()/config/<id>

# Create spec and doc symlinks (if present)
ln -s <absolute-path-to-user-apps>/<item-id>/spec dataDir()/specs/external-specs/<id>
ln -s <absolute-path-to-user-apps>/<item-id>/doc dataDir()/docs/external-docs/<id>

# Create hooks symlink (if hooks/ exists)
if [ -d "<absolute-path-to-user-apps>/<item-id>/hooks" ]; then
  ln -s <absolute-path-to-user-apps>/<item-id>/hooks dataDir()/system/hooks/<id>
fi

# Create app symlink (if app/ exists)
if [ -d "<absolute-path-to-user-apps>/<item-id>/app" ]; then
  ln -s <absolute-path-to-user-apps>/<item-id>/app dataDir()/system/app/<id>
fi
```

**Step 3: Validate service manifest**
- Read `dataDir()/system/services/<id>/service.json`
- Validate required fields (id, name, version, entry)
- Validate `configSchema` if present (must be valid JSON Schema)
- Detect self-dependencies — reject if service declares itself as a dependency
- If validation fails: abort installation, log error

**Step 4: Register service in registry**
- Load service manifest from `dataDir()/system/services/<id>/service.json`
- Add to service registry (both source and installed state)
- Resolve dependencies and determine startup order
- Register settings component if settingsRegistration is present

**Step 5: Notify Settings UI**
- Settings app refreshes to show the new service in Settings → Plugins → [Services]

#### Uninstallation Process (Server-Side)

When a user uninstalls a service, the following server-side operations occur:

**Step 1: Stop the service (if running)**
- If the service is currently running, stop it first
- This sends `dispose` to the worker thread and terminates it
- Updates status to "Stopped"

**Step 2: Remove symlinks**
```bash
# Remove dataDir()/system/<type>/<id> symlinks
rm dataDir()/system/services/<id>
rm dataDir()/system/hooks/<id>  # if present
rm dataDir()/system/app/<id>     # if present

# Remove config directory symlink
rm dataDir()/config/<id>

# Remove spec and doc symlinks (if present)
rm dataDir()/specs/external-specs/<id>
rm dataDir()/docs/external-docs/<id>
```

**Step 3: Remove from registry**
- Remove service from registry (both source and installed state)
- Remove settings component registration

**Step 4: Remove item from user-apps**
```bash
# Remove the item directory
rm -rf dataDir()/user-apps/<item-id>/
```

**Note**: If the item was cloned from a marketplace, the marketplace clone itself (`dataDir()/marketplace/<market-place-id>/items/<item-id>/`) is NOT deleted — it remains available for re-installation. Only the user-apps entry is removed.

#### Worker Thread IPC Protocol

Worker threads cannot directly export functions — they communicate via `postMessage`/`onmessage`. The following message protocol is used:

**Main process → Worker thread**:
```typescript
type MainToWorkerMessage =
  | { type: 'initialize'; configDirPath: string; logsPath: string; serviceId: string }
  | { type: 'dispose' }
  | { type: 'restart'; reason: string };
```

**Worker thread → Main process**:
```typescript
type WorkerToMainMessage =
  | { type: 'initialized' }
  | { type: 'bound'; port: number; host: string }     // CH-005: publish actual bound port
  | { type: 'error'; message: string; stack?: string }
  | { type: 'disposed' }
  | { type: 'log'; level: 'info' | 'warn' | 'error'; message: string }
  | { type: 'crash'; error: string; stack?: string };
```

**Main process worker lifecycle management** (CH-001 fix):
The main process listens for the Worker `exit` and `error` events. If the worker exits unexpectedly (non-zero exit code, or `error` event on the Worker object), the main process treats it as a crash — regardless of whether the worker sent a `crash` message. This is critical because if the worker dies during `initialize` (e.g., malformed config, broken symlink, entrypoint error), it cannot send a `crash` or `initialized` message. The main process MUST:
- Increment the restart counter
- Log the crash with the exit code and stderr captured by the Worker
- Trigger crash recovery (respecting max restarts and backoff)
The startup timeout mechanism is separate from crash detection: the timeout only applies if the worker is alive but unresponsive. A crashed worker is detected via the `exit`/`error` events.

**Implementation**:
```typescript
import { Worker } from 'node:worker_threads';

const worker = new Worker(entryPath, {
  workerData: { configDirPath, logsPath, serviceId },
  resourceLimits: {
    maxOldGenerationSizeMb: 256,
    maxYoungGenerationSizeMb: 64,
  }
});

// Detect unexpected worker exit (CH-001 fix)
worker.on('exit', (code) => {
  if (code !== 0) {
    // Worker crashed — increment restart counter, trigger recovery
    handleCrash(serviceId, `Worker exited with code ${code}`);
  }
});
worker.on('error', (err) => {
  // Worker encountered an error — same as crash
  handleCrash(serviceId, err.message);
});

// Wait for initialization
worker.postMessage({ type: 'initialize', configDirPath, logsPath, serviceId });
await waitForMessage(worker, 'initialized');

// On dispose
worker.postMessage({ type: 'dispose' });
await waitForMessage(worker, 'disposed');
```

**Worker thread entrypoint** must implement:
```typescript
// Worker entrypoint (e.g., services/terminal/index.js)
// Parameterized by serviceId from workerData — NOT hardcoded
onmessage = async (event) => {
  const { type } = event.data;
  const { configDirPath, logsPath, serviceId } = event.data;

  if (type === 'initialize') {
    // Load config from dataDir()/config/<serviceId>/<serviceId>.json
    const configPath = path.join(configDirPath, `${serviceId}.json`);
    try {
      const config = JSON.parse(fs.readFileSync(configPath));
      // Bind port, start service
      const host = config.host || 'localhost';
      const port = config.port || 3001;
      const server = net.createServer(handler);
      await new Promise<void>((resolve) => {
        server.listen(port, host, () => resolve());
      });
      const actualPort = server.address().port;
      // Log to dataDir()/logs/services/<id>.log
      postMessage({ type: 'initialized' });
      // Publish actual bound port (CH-005)
      postMessage({ type: 'bound', port: actualPort, host });
    } catch (err) {
      // If we crash before initialized, main process detects via 'exit' event (CH-001)
      throw err;
    }
  } else if (type === 'dispose') {
    // Close WebSocket connections, release resources
    server?.close();
    postMessage({ type: 'disposed' });
  } else if (type === 'restart') {
    // Same as initialize but with restart reason
    postMessage({ type: 'initialized' });
    postMessage({ type: 'bound', port: actualPort, host });
  }
};
```

**Key design decisions**:
- Worker threads run in the same Node.js process as BOS (no container isolation for v1).
- Worker threads provide memory isolation (separate heaps) but NOT CPU isolation — a runaway CPU worker can starve the process. Resource limits and aggressive `worker.terminate()` mitigate this risk.
- Resource limits are set per worker thread (see Resource Limits below).
- The `serviceId` is passed via `workerData` so the worker constructs the config file path dynamically (`<serviceId>.json`), avoiding hardcoded names.
- The worker tries to bind to a configurable `host` (from config, default `localhost`) to support external interfaces.
- **CH-001**: If the worker crashes during `initialize` (before sending `initialized`), the main process detects it via the Worker `exit`/`error` events and triggers crash recovery.
- **CH-005**: When the service binds to a port, it MUST publish the actual port via `bound` message. This is required for port 0 (random port) support — the OS assigns the port and the worker reports it back. The service manager writes the bound port to `runtime.json` (not the user config file).

**Service lifecycle states**:

```
┌─────────────────────────────────────────────────────────────┐
│                      Service Lifecycle                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Stopped ──Start──→ Running                                 │
│    ↑                        │                               │
│    │                        ↓                               │
│    │                  Stopped (manual)                      │
│    │                        │                               │
│    │                        ↓                               │
│    └─────Restart──→ Restarting ──→ Running                  │
│                                                             │
│  Crash ──→ Crash Recovery (exponential backoff)            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Lifecycle operations**:

1. **Start**:
   - Validate service manifest (required fields present, entry path exists)
   - Resolve entrypoint path: `path.resolve(dataDir(), 'system/services', serviceId, serviceManifest.entry)`
   - Validate entrypoint file exists
   - Resolve config directory: `dataDir()/config/<id>` (symlink to user-apps config)
   - If config directory is empty or missing expected files, log a warning (not an error)
   - Create worker thread with resource limits
   - Send `initialize` message to worker (pass config directory path, not individual config file path)
   - Wait for `initialized` response (timeout configurable, see CH-015)
   - Set status to "Running"
   - Listen for `bound` messages (CH-005) — write actual bound port to runtime config

2. **Stop**:
   - Send `dispose` message to worker
   - Wait for `disposed` response (timeout configurable, see CH-015)
   - Terminate worker thread
   - Set status to "Stopped"
   - Reset restart counter to 0

3. **Restart**:
   - Call Stop followed by Start
   - Status briefly shows "Restarting" during the transition

4. **Crash Recovery**:
   - If the worker thread exits unexpectedly (crash), the crash handler fires
   - Check if `restartCount < maxRestarts`
   - If yes: wait for backoff time, then call Start
   - If no: stop permanently, log error, status remains "Stopped"
   - The restart counter resets to 0 when `initialized` message is received from worker

#### Resource Limits

Worker threads run in the same Node.js process as BOS. To prevent a runaway service from starving the process:

```typescript
const worker = new Worker(entryPath, {
  workerData: { configDirPath, logsPath },
  resourceLimits: {
    maxOldGenerationSizeMb: 256,
    maxYoungGenerationSizeMb: 64,
  }
});
```

**Defaults**:
- `maxOldGenerationSizeMb`: 256 MB per worker thread
- `maxYoungGenerationSizeMb`: 64 MB per worker thread
- These limits are applied per worker thread (each service gets its own worker)

**Monitoring**:
- The service manager monitors worker thread memory usage
- If a worker exceeds 90% of its memory limit, an alert is logged
- If a worker is killed by the V8 GC due to memory limits, it is treated as a crash and crash recovery kicks in

#### Port Conflict Detection

Port availability is checked before the service attempts to bind:

```typescript
function checkPortAvailable(port: number): Promise<boolean> {
  return new Promise((resolve) => {
    const server = net.createServer();
    server.once('error', () => resolve(false)); // Port in use
    server.once('listening', () => {
      server.close();
      resolve(true);
    });
    server.listen(port, '127.0.0.1');
  });
}
```

- If port is in use: service refuses to start, logs error "Port <N> is already in use. Configure a different port in Settings."
- No auto-retry on a different port (the user must explicitly change the port in Settings)
- Port 0 (random available port) is supported — the OS assigns a free port, and the worker publishes the actual port via the `bound` message (CH-005)

#### Service Discovery (Dependencies)

Dependent services discover their dependencies by reading the dependency's config directory. Binding information (actual port, host) is published in `dataDir()/config/<depId>/runtime.json` (the runtime state file, separate from user config):

```typescript
// Service A depends on Service B
// Service A reads B's runtime state to discover the actual bound port
const runtime = JSON.parse(
  fs.readFileSync(path.join(dataDir(), 'config', 'b', 'runtime.json'))
);
const ws = new WebSocket(`ws://${runtime.host}:${runtime.port}`);
```

- `runtime.json` contains: `{ "port": number, "host": string }` — populated by the service manager after the worker publishes the `bound` message
- User config values (e.g., `port: 0` or `port: 3001`) live in the item's config files (e.g., `terminal.json`)
- Dependent services read `runtime.json` (not user config) to discover actual binding info
- If `runtime.json` does not exist or is missing `port`, the dependent service logs a warning and cannot connect

#### Startup Ordering

Services start in dependency order (services with no dependencies start first). Startup ordering is determined by topological sort of the dependency graph.

**Behavior on dependency failure (CH-007)**:
- If a service fails to start, the system continues starting the next service as if nothing happened.
- **ALWAYS LOG** the failure with full context (service ID, error, timestamp) at `warn` level.
- The dependent service still starts (ordering is advisory, not blocking).
- The dependent service reads the dependency's `runtime.json` to discover its binding — if the dependency is not running, `runtime.json` will be absent and the dependent service logs a warning but continues.
- **CH-003 fix — Crash loop prevention**: Before starting a dependent service, the system checks whether its declared dependencies are currently in `Running` state. If any dependency is not running, the system logs a `warn` ("Dependency X is not running — skipping service Y to avoid crash loop") and does NOT start the dependent service. The dependent service remains in `Stopped` state until its dependencies are started manually.

**Dependency validation**:
- The dependency resolver detects circular dependencies and logs a warning (does NOT prevent startup)
- The dependency resolver detects missing dependencies (dependency ID not found in installed services) and logs a warning (does NOT prevent startup)
- Self-dependencies are rejected at install time with an error (service cannot declare itself as a dependency)

#### Crash Recovery

```typescript
interface CrashRecoveryPolicy {
  maxRestarts: number;       // Maximum number of restart attempts (default: 5)
  backoffMs: number;         // Initial backoff time in milliseconds (default: 1000)
  backoffMultiplier: number; // Multiplier for exponential backoff (default: 2)
}
```

**Backoff calculation**:
- Restart 1: wait `backoffMs` (1s)
- Restart 2: wait `backoffMs * backoffMultiplier` (2s)
- Restart 3: wait `backoffMs * backoffMultiplier^2` (4s)
- Restart 4: wait `backoffMs * backoffMultiplier^3` (8s)
- Restart 5: wait `backoffMs * backoffMultiplier^4` (16s)
- If restart 5 also fails: stop permanently

**Crash detection sources**:
1. Worker sends `crash` message via IPC (worker detected its own crash condition)
2. Worker `exit` event with non-zero code (CH-001: crash during `initialize` or other phases)
3. Worker `error` event on the Worker object (CH-001: unexpected error)
4. Startup timeout exceeded — worker alive but unresponsive (distinct from crash)

All sources increment the restart counter and trigger the recovery flow.

**Log capture**:
- Worker thread stdout/stderr is captured and written to `dataDir()/logs/services/<id>.log`
- Log rotation is handled by the system logging layer (not by the service manager)
- Settings UI can display the log content in a viewer

**CH-015 — Configurable timeouts**:
- Startup timeout (wait for `initialized` response): configurable, default 30s, minimum 0 (disabled)
- Shutdown timeout (wait for `disposed` response): configurable, default 30s, minimum 0 (disabled)
- If set to 0, the operation proceeds without waiting
- If the timeout is exceeded, the worker is force-killed via `worker.terminate()` and the event is logged at `warn` level

#### Manifest Validation (CH-011)

**At install time**:
- Validate `service.json` is valid JSON
- Validate required fields (id, name, version, entry) are present
- Validate `configSchema` (if present) is valid JSON Schema
- Validate entrypoint file exists (resolve path and check)
- Validate self-dependencies are not declared

**At start time**:
- Validate entrypoint can be loaded (try `require()`, catch errors)
- If the entrypoint fails to load, log a clear error and do NOT start the service
- If `dataDir()/config/<id>/` exists but is empty, log a warning (not an error)

#### Error Handling

- Permission denied errors during symlink creation MUST be caught and reported with actionable messages
- If a service manifest (`service.json`) is deleted after install, the service MUST be marked as "corrupted" and refuse to start until the manifest is restored

### Non-Functional Requirements

- **NFR-001**: Services MUST run in the same process as BOS (no container isolation for v1).
- **NFR-002**: A service crash MUST NOT crash BOS — worker threads have isolated memory heaps. However, all workers share the Node.js process, so resource limits are enforced per worker to prevent one runaway service from starving the process.
- **NFR-003**: Service startup MUST be awaited before the chat pipeline starts (services are ready when BOS is ready).
- **NFR-004**: Service logs MUST be written to `dataDir()/logs/services/<id>.log` (handled by the system logging layer).
- **NFR-005**: Service config changes MUST NOT require BOS restart — only service restart.
- **NFR-006**: Service status changes MUST be reflected in the Settings UI in real-time.

## Requirements (Numbered)

### Service Item Structure

- **FR-001**: Every service item MUST follow the directory structure defined in "Complete Item Directory Structure".
- **FR-002**: The `services/<id>/` directory MUST contain a `service.json` manifest with all fields defined in "Service `service.json` Manifest".
- **FR-003**: The `config/` directory MUST always be present (even if empty) in the item.

### Installation & Symlinks

- **FR-004**: Installing a service MUST create symlinks in `dataDir()/system/`:
  ```
  dataDir()/system/services/<id> -> dataDir()/user-apps/<id>/services
  dataDir()/config/<id> -> dataDir()/user-apps/<id>/config  # config directory symlink
  ```
- **FR-005**: Uninstalling a service MUST delete the symlinks in `dataDir()/system/` and `dataDir()/config/<id>` (directory symlink, NOT a single .json file).

### Service Registry

- **FR-006**: The service registry MUST distinguish between two states:
  - **Source state** (available to install): Items in `dataDir()/user-apps/` and `dataDir()/marketplace/<id>/items/` that have a `services/<id>/service.json` manifest. These are discovered during registry scan but are NOT yet installed.
  - **Installed state** (runtime): Services whose symlinks exist in `dataDir()/system/services/<id>/`. These can be started, stopped, and restarted.
  - A service can exist in source state without being installed. A service in installed state has its symlinks active and can be managed via lifecycle operations.
- **FR-007**: The registry MUST load service manifests from `dataDir()/system/services/<id>/service.json`.
- **FR-008**: The registry MUST resolve dependencies and determine startup order before starting services.

### Service Lifecycle

- **FR-009**: Services MUST run as **worker threads** (not child processes or containers).
- **FR-010**: Services MUST support this lifecycle: `Stopped → Running → Stopped → Restarting → Running`.
- **FR-011**: Starting a service MUST follow the lifecycle operations defined in "Lifecycle operations" step 1.
- **FR-012**: Stopping a service MUST follow the lifecycle operations defined in "Lifecycle operations" step 2.
- **FR-013**: Restarting a service MUST call Stop followed by Start.

### Resource Limits

- **FR-014**: Worker threads MUST have `resourceLimits` set: `maxOldGenerationSizeMb: 256`, `maxYoungGenerationSizeMb: 64`.

### Port & Network

- **FR-015**: Services MAY bind to network ports. The port MUST be configurable via the item's config files under `dataDir()/config/<id>/`.
- **FR-016**: If no port is configured, the service MUST bind to a default port (e.g., 3001 for terminal).
- **FR-017**: The consuming app MUST fetch the service config from `dataDir()/config/<id>/` to discover the port.
- **FR-018**: If a port is already in use, the service MUST fail to start and log an error.
- **FR-019**: (CH-005) After binding, the worker MUST publish the actual bound port via `{ type: 'bound', port, host }` message. The service manager MUST write the bound port to the config file so dependent services and apps can discover it.

### Dependencies

- **FR-020**: Services MAY declare dependencies on other services via the `dependencies` field in `service.json`.
- **FR-021**: Services MUST start in dependency order (services with no dependencies start first). Startup ordering is determined by topological sort of the dependency graph.
- **FR-022**: If a dependency fails to start, the system MUST continue starting the next service (do NOT block). **ALWAYS LOG** the failure with full context.
- **FR-023**: Circular dependency detection logs a warning at startup but does NOT prevent startup.
- **FR-024**: Missing dependency detection logs a warning at startup but does NOT prevent startup.
- **FR-025**: Self-dependencies MUST be rejected at install time with an error.
- **FR-026**: Dependent services discover their dependencies by reading the dependency's config directory (`dataDir()/config/<depId>/`).
- **FR-027**: (CH-005) Services publish their binding information (port, host) via the `bound` IPC message, and the service manager writes it to the config directory.

### Settings UI

- **FR-028**: Services MUST appear in Settings → Plugins → [Services] section (below [Plugin Pipeline]).
- **FR-029**: The Services section MUST show: service name, status indicator (running/stopped), version, Start/Stop/Restart buttons, Config button, Logs button.
- **FR-030**: Services MAY register settings UI components via `settingsRegistration` in `service.json`.
- **FR-031**: Settings UI components MUST be React components that integrate into the Settings app.
- **FR-032**: Config changes in Settings MUST be auto-saved (no Save button). Changes are written directly to `dataDir()/config/<id>/<configFile>.json` and take effect on next service restart.
- **FR-033**: (CH-009) Service status changes MUST be reflected in the Settings UI in real-time. The service manager MUST broadcast status events (running/stopped/crashed) to the Settings UI.

### Hooks Integration

- **FR-034**: Services MAY provide hooks via the `hooks/` directory in the service item.
- **FR-035**: Hooks MUST be registered via `registerHook()` (not `registerPlugin()`). The hooks system replaces the plugin system — all references to plugins, pipeline, and related terminology are replaced with hooks.
- **FR-036**: Hooks MUST follow the same format as existing hook-based plugins (CommonJS modules).
- **FR-037**: The plugin system (026-plugin-pipeline) is being renamed to the hooks system. `registerPlugin()` becomes `registerHook()`. `plugin.json` becomes `hook.json`. All plugin-related methods, functions, and texts are updated accordingly. This spec assumes the rename is complete. If the rename is not complete, the service hooks feature MUST NOT be implemented until the plugin system rename is finished.

## Key Entities

- **ServiceManifest** — Service metadata and entrypoint definition (from `service.json`).
- **ServiceDefinition** — Runtime service state (status, worker thread, config directory path).
- **ServiceConfig** — User-configurable settings stored as files in `dataDir()/config/<id>/`.
- **Service Registry** — Discovers and manages service lifecycle across source and installed states.
- **Service Worker** — The worker thread running the service.

## Success Criteria

### Measurable Outcomes

- **SC-001**: A terminal service can be installed from the marketplace, started, and connected to via WebSocket.
- **SC-002**: A service can be stopped and restarted from Settings UI.
- **SC-003**: A service port can be changed in Settings (auto-saved) and takes effect on restart.
- **SC-004**: A crashing service automatically restarts up to the configured max restarts.
- **SC-005**: Services start in dependency order.

### Acceptance Criteria

#### Service Installation

- [x] Installing a service creates symlinks in `dataDir()/system/services/` and `dataDir()/config/<id>` (directory)
- [x] Uninstalling a service deletes the symlinks
- [x] User-apps mirror the marketplace structure

#### Service Lifecycle

- [x] Services can be started, stopped, and restarted
- [x] Status changes are reflected in Settings UI in real-time
- [x] Services run as worker threads
- [x] Services can be configured via Settings (auto-saved)

#### Crash Recovery

- [x] Crashing services automatically restart
- [x] Restart policy is configurable (max restarts, backoff)
- [x] Restart counter resets on successful start
- [x] Service logs capture stdout/stderr

#### Settings UI

- [x] Services appear in Settings → Plugins → [Services]
- [x] Start/Stop/Restart buttons work
- [x] Config panel auto-saves to `dataDir()/config/<id>/`
- [x] Logs viewer shows service output
- [x] Status updates are real-time (no polling delay)

#### Dependencies

- [x] Services start in dependency order
- [x] Failed dependencies are logged but do NOT block other services from starting

## Implementation Status

Phases 1, 2, 3, 4, 5, 6, and 8 (backend, UI, polish) are implemented. Phase 7 (automated test coverage) is not.

| Phase | Area | Status |
|---|---|---|
| 1 | Service item structure, symlink layout | ✅ Implemented — `src/system/marketplace/install/symlinkManager.ts`, `serviceInstaller.ts` |
| 2 | Service registry (source/installed split, discovery) | ✅ Implemented — `src/core/service/ServiceRegistry.ts` |
| 3 | Service lifecycle (worker threads, start/stop/restart, crash recovery, IPC) | ✅ Implemented — `src/core/service/ServiceManager.ts`, `CrashRecovery.ts`, `workerIpc.ts` |
| 4 | Settings UI (service list, Start/Stop/Restart, config panel, logs viewer) | ✅ Implemented — `src/components/apps/settings/ServicesTab.tsx`, `ServiceCard.tsx`, `ServiceConfigPanel.tsx`, `ServiceLogViewer.tsx` wired into `PluginsTab.tsx` (Services section below Plugin Pipeline; right pane switches between plugin detail / service config / service logs). `npx tsc --noEmit` + `npx eslint` pass clean. |
| 5 | Network binding (configurable ports, port-conflict detection, `bound` message) | ✅ Implemented — `src/core/service/PortChecker.ts`, `ServiceManager.ts` |
| 6 | Dependencies (topological sort, circular/missing detection, crash-loop prevention) | ✅ Implemented — `src/core/service/DependencyResolver.ts` |
| 7 | Automated tests | ⚠️ **Partial.** Stub test files exist under `tests/services/` (`ServiceManager.test.ts`, `manifestValidator.test.ts`, `integration.test.ts`) but currently fail lint (misuse of a test-data-dir hook outside a component, `any` types) and are not a reliable coverage signal yet. |
| 8 | Polish & cross-cutting concerns | ✅ Implemented — see "Known Limitations" below for what polish intentionally does NOT cover |

## Known Limitations

- **No CPU isolation between services**: worker threads give each service its own V8 heap (crash/memory isolation, NFR-002), but all workers share the single Node.js process and its single-threaded event loop for JS execution. A CPU-bound worker can still starve BOS and other services. `resourceLimits` (256MB old-gen / 64MB young-gen) bound memory only, not CPU time.
- **No container-based services (Docker) for v1**: worker threads are the only supported isolation mechanism. True process/CPU isolation would require a container runtime, which is explicitly out of scope (see "Non-Goals").
- **No port auto-retry on conflict**: if a configured port is already in use, the service fails to start and logs an actionable error ("Port <N> is already in use. Configure a different port in Settings."). The user must change the port manually — there is no automatic fallback to another port.
- **No direct function exports from services**: all main-process ↔ worker communication is via the `postMessage`/`onmessage` IPC protocol (`initialize`/`dispose`/`restart` in, `initialized`/`bound`/`error`/`disposed`/`log`/`crash` out). A service cannot expose a callable API to the main process beyond this message set.
- **Hooks integration (FR-034–FR-037) is inert pending the plugin→hooks rename**: the assumption "the plugin system rename from 'pipeline' to 'hooks' is a prerequisite for the hooks integration feature" is **not yet satisfied** — `src/lib/plugins/*` still uses `registerPlugin()`/`plugin.json` (026-plugin-pipeline has not been renamed). Consequently, while `serviceInstaller`/`symlinkManager` DO create the `dataDir()/system/hooks/<id>` symlink for a service's optional `hooks/` directory, there is no `registerHook()` and nothing loads or registers hooks from that symlink today. Service items may ship a `hooks/` directory, but it has no effect until the rename lands.
- **Corrupted-state recovery requires a fresh scan**: a service stays `"corrupted"` (service.json missing/invalid) until the next `discoverServices()` call observes a valid manifest again (e.g. the next `GET /api/services` or `GET /api/services/[id]`); there is no filesystem watcher, so recovery is detected on next poll, not instantaneously.

## Assumptions

- **Worker threads**: Services run as worker threads (not child processes) for v1, with crash isolation via isolated memory heaps.
- **No containerization**: Docker or other container runtimes are out of scope for v1.
- **Port conflicts**: If a port is already in use, the service fails to start (no auto-retry on different ports).
- **Settings UI**: Settings components are React components, not standalone apps. All config changes auto-save (no Save button).
- **Hooks**: Hooks replace plugins. `registerHook()` replaces `registerPlugin()`. The plugin system rename from "pipeline" to "hooks" is a prerequisite for the hooks integration feature.
- **Migration**: Existing `dataDir()/plugins/<id>/` will be migrated to `dataDir()/user-apps/<id>/hooks/`.
- **Config**: Config is a directory at `dataDir()/config/<id>` (symlinked to `dataDir()/user-apps/<id>/config/`). The directory can contain any number of config files determined by the item. Services read from this directory.
- **IPC**: All service-worker communication is via `postMessage` protocol — no direct function calls. Workers publish their bound port via `bound` message.
- **Timeouts**: Startup and shutdown timeouts are configurable and optional.

## Touched Specs

| Spec | Path | What needs updating |
|------|------|---------------------|
| **026-plugin-pipeline** | `bos-system-specs/026-plugin-pipeline/` | Rename "pipeline" to "hooks", `registerPlugin()` to `registerHook()` |
| **027-vfs-specfs-marketplace** | `bos-system-specs/027-vfs-specfs-marketplace/` | Add service item type to marketplace |
| **028-marketplace-sandbox** | `bos-system-specs/028-marketplace-sandbox/` | Extend app provider registry for services |
| **009-installed-apps** | `bos-system-specs/009-installed-apps/` | Update app directory structure |

## Implementation Plan

### Phase 1: Service Item Structure
- [x] Define `service.json` manifest format
- [x] Create `dataDir()/user-apps/<id>/services/` layout
- [x] Create `dataDir()/system/services/<id>/` symlink structure
- [x] Create `dataDir()/config/<id>` config directory symlink
- [x] Update marketplace schema to support service items

### Phase 2: Service Registry
- [x] Create service registry with source + installed state distinction
- [x] Load service manifests from `dataDir()/system/services/<id>/service.json`
- [x] Resolve dependencies and determine startup order (topological sort)
- [x] Expose registry API for lifecycle management

### Phase 3: Service Lifecycle
- [x] Implement worker thread creation for services
- [x] Implement `initialize()` and `dispose()` lifecycle via postMessage
- [x] Implement Start/Stop/Restart operations
- [x] Implement crash recovery with exponential backoff
- [x] Implement `bound` message handling (CH-005) — write actual port to config
- [x] Log service stdout/stderr to `dataDir()/logs/services/<id>.log`

### Phase 4: Settings UI
✅ Implemented — `src/components/apps/settings/ServicesTab.tsx`, `ServiceCard.tsx`, `ServiceConfigPanel.tsx`, `ServiceLogViewer.tsx`, wired into `PluginsTab.tsx`.
- [x] Add [Services] section to Settings → Plugins
- [x] Show service list with real-time status indicators (CH-009)
- [x] Implement Start/Stop/Restart buttons
- [x] Implement auto-saving config panel (no Save button) — writes to `dataDir()/config/<id>/`
- [x] Implement logs viewer
- [x] Implement status event broadcasting from service manager to UI

### Phase 5: Network Binding
- [x] Implement configurable port in service config files
- [x] Implement port conflict detection
- [x] Implement `bound` message protocol for port 0 support
- [x] Update consuming apps to read port from `dataDir()/config/<id>/`

### Phase 6: Dependencies
- [x] Implement dependency graph from `service.json`
- [x] Implement startup ordering (topological sort)
- [x] Log failures but continue starting other services (CH-007)
- [x] Detect circular/missing dependencies and log warnings

### Phase 7: Automated Testing (Not started — not yet implemented)
- [ ] Write tests for service registry
- [ ] Write tests for service lifecycle
- [ ] Write tests for crash recovery
- [ ] Write tests for Settings UI (auto-save, real-time updates)
- [ ] Write tests for dependencies and startup ordering

### Phase 8: Testing
- [ ] Write tests for service registry
- [ ] Write tests for service lifecycle
- [ ] Write tests for crash recovery
- [ ] Write tests for Settings UI (auto-save, real-time updates)
- [ ] Write tests for dependencies and startup ordering
