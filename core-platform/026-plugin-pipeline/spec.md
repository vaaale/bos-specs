# Feature Specification: Plugin Pipeline Architecture

**Feature Branch**: `026-plugin-pipeline`

**Created**: 2026-07-20

**Status**: Draft

**Input**: "Extract compaction and memory into plugins with a generalized hook-based plugin API that can be extended to all BOS subsystems."

## Overview

This spec defines a plugin pipeline architecture for BrowserOS that:

1. **Generalizes** the chat pipeline into a set of hooks that plugins can register at
2. **Extracts** compaction and memory into optional plugins (not hard-coded)
3. **Enables** marketplace distribution of server-side plugins
4. **Provides** configuration UI in Settings for plugin management

### Goals

- Plugins are optional — BOS ships with default implementations but users can swap them out
- Plugin API is generalized — works for any subsystem (memory, compaction, telemetry, integrations, etc.)
- Marketplace integration — plugins install from Marketplace app
- Configuration — plugins register config pages in Settings or provide their own config app

### Non-Goals

- Client-side plugins (browser extensions) — out of scope
- Plugin marketplace backend — uses existing Marketplace app infrastructure
- Plugin approval workflow — trust model is user-installed only

## User Scenarios & Testing

### User Story 1 - I can swap out the compaction plugin (Priority: P1)

The user installs a different compaction plugin from the Marketplace and configures it in Settings.

**Acceptance Scenarios**:

1. **Given** the default compaction plugin is active, **When** the user installs a marketplace compaction plugin, **Then** they can deactivate the default and activate the new one in Settings → Plugins.
2. **Given** a marketplace compaction plugin is installed, **When** the user opens Settings → Plugins, **Then** they see the plugin in the pipeline list with an active toggle and config button.

### User Story 2 - I can configure plugins in Settings (Priority: P1)

The user opens Settings → Plugins and sees a reorderable list of installed plugins with individual configuration.

**Acceptance Scenarios**:

1. **Given** plugins are installed, **When** the user opens Settings → Plugins, **Then** they see a master-detail view with pipeline order, active toggle, and config panel.
2. **Given** a plugin has a config schema, **When** the user clicks Configure, **Then** they see a form rendered from the schema.
3. **Given** a plugin has a bundled config app, **When** the user clicks Open Config App, **Then** the app opens in a sidebar or modal.

### User Story 3 - Plugins run in the chat pipeline (Priority: P1)

The chat pipeline runs through all active plugins in order, with each plugin able to modify messages, intercept tool calls, or handle errors.

**Acceptance Scenarios**:

1. **Given** plugins are registered and active, **When** a user message is sent, **Then** the beforeRun hooks run in registration order before the LLM call.
2. **Given** a plugin throws in beforeRun, **When** the error propagates, **Then** the onError hooks run and the user sees a graceful error message.
3. **Given** a plugin modifies messages in beforeRun, **When** the LLM responds, **Then** the modified messages are sent to the model.

### User Story 4 - BOS ships with default plugins (Priority: P2)

BOS ships with default memory and compaction plugins that can be swapped out.

**Acceptance Scenarios**:

1. **Given** a fresh BOS install, **When** the user opens Settings → Plugins, **Then** they see "Memory System" and "Context Compaction" as active plugins.
2. **Given** the default plugins are active, **When** the user deactivates one, **Then** the corresponding functionality is disabled (memory stops working, compaction falls back to mechanical).

## Requirements

### Functional Requirements

#### Plugin Pipeline

- **FR-001**: The chat pipeline MUST support a set of hooks that plugins can register at: `beforeRun`, `extendSystemPrompt`, `beforeToolCall`, `afterToolCall`, `afterRun`, `onError`, `onRunFinished`.
- **FR-002**: Plugins MUST be loaded from `dataDir()/plugins/<id>/` and registered via a global plugin registry.
- **FR-003**: The pipeline MUST execute hooks in registration order (plugins registered first run first).
- **FR-004**: Plugins MUST be optional — BOS MUST NOT hard-code any plugin functionality. Default plugins (memory, compaction) are still plugins loaded via the plugin system, not hard-coded.
- **FR-005**: BOS MUST ship with default plugin implementations for memory and compaction that are active by default but can be deactivated/swapped.

#### Plugin Manifest

- **FR-006**: Every plugin MUST have a `plugin.json` manifest with: `id`, `name`, `version`, `type` (always "server-plugin"), `provides` (array of hook types), `configSchema` (optional), `entry` (JS module path), `configApp` (optional), `settingsRegistration` (optional).
- **FR-007**: The `provides` field MUST declare which hook types the plugin implements (e.g., `["beforeRun", "extendSystemPrompt"]`).
- **FR-008**: Plugins MUST be CommonJS modules (no bundling, no build step).

#### Plugin Registry

- **FR-009**: The plugin registry MUST be global (using `globalThis.__bosPlugins` like the existing hook system).
- **FR-010**: Plugins MUST register via `registerPlugin(plugin: PluginDefinition)` and can be unregistered via `unregisterPlugin(id)`.
- **FR-011**: The registry MUST support composing hooks: `composeHooks<K extends keyof BosPluginHooks>(hookName: K, context: RunContext): BosPluginHooks[K][]`.
- **FR-012**: The registry MUST be initialized at BOS startup by loading all active plugins from `dataDir()/config/plugins.json`.

#### Plugin Lifecycle

- **FR-013**: Plugins MUST implement `initialize(context: PluginContext): Promise<void>` for setup (called once at startup).
- **FR-014**: Plugins MUST implement `dispose(): Promise<void>` for cleanup (called on shutdown or deactivation).
- **FR-015**: Plugins MUST be able to read/write their own state via `PluginContext` (transcript read, sidecar write, model calls).
- **FR-016**: Plugins MUST implement `getConfig(): Promise<Record<string, unknown>>` and `setConfig(config: Record<string, unknown>): Promise<void>` for configuration.

#### Configuration

- **FR-017**: Plugin configuration MUST be stored in `dataDir()/config/plugins.json` with `active` (array of plugin IDs) and `config` (map of plugin ID to config).
- **FR-018**: Plugins MUST declare a `configSchema` (JSON Schema) in their manifest for form-based configuration in Settings.
- **FR-019**: Plugins MAY bundle a configuration app (HTML/JS) that opens in a sidebar or modal.
- **FR-020**: Plugins MAY register a settings row in Settings → Plugins via `settingsRegistration` (label, icon, order, description).

#### Marketplace Integration

- **FR-021**: Marketplace MUST support a new item type "server-plugin" with a plugin manifest and entry JS module.
- **FR-022**: Installing a server plugin MUST copy files to `dataDir()/plugins/<id>/` and validate the manifest.
- **FR-023**: Activating a plugin MUST add it to the `active` array in `dataDir()/config/plugins.json` and call `registerPlugin()`.
- **FR-024**: Deactivating a plugin MUST remove it from the `active` array and call `unregisterPlugin()`.
- **FR-025**: Uninstalling a plugin MUST remove files from `dataDir()/plugins/<id>/` and call `unregisterPlugin()`.

#### Settings UI

- **FR-026**: Settings MUST have a "Plugins" tab with a master-detail layout showing pipeline order, active toggle, and config panel.
- **FR-027**: The plugin list MUST be reorderable (drag handles) to control execution order.
- **FR-028**: Each plugin row MUST show: name, version, description, active toggle, config button, uninstall button.
- **FR-029**: Clicking Configure MUST open the plugin's config panel (inline form or bundled app).
- **FR-030**: The config panel MUST render a form from the plugin's `configSchema` and save to `dataDir()/config/plugins.json`.

#### Migration

- **FR-031**: The existing compaction middleware MUST be refactored to use plugin hooks (no behavior change).
- **FR-032**: The existing memory system MUST be refactored to use plugin hooks (no behavior change).
- **FR-033**: Existing config files (`dataDir()/config/compaction.json`, `dataDir()/config/memory.json`) MUST be migrated to `dataDir()/config/plugins.json`.

### Non-Functional Requirements

- **NFR-001**: Plugins MUST run in the same process as BOS (no sandboxing, no isolation).
- **NFR-002**: Plugins MUST NOT be able to access BOS source code (only `dataDir()` and provided APIs).
- **NFR-003**: Plugin initialization MUST be async and await all plugins before starting the pipeline.
- **NFR-004**: Plugin errors MUST not crash BOS — `onError` hooks MUST handle exceptions gracefully.
- **NFR-005**: Plugin configuration changes MUST take effect on next pipeline call (no restart required).

## Key Entities

- **PluginDefinition** — Plugin metadata and hook implementations.
- **PluginContext** — BOS APIs provided to plugins (transcript read, sidecar write, model calls).
- **RunContext** — Information about the current run (conversation ID, model, token budget).
- **Plugin Registry** — Global registry of active plugins.
- **Plugin Pipeline** — The chat pipeline that runs hooks in order.
- **Plugin Manifest** — `plugin.json` file declaring plugin metadata and config schema.

## Success Criteria

### Measurable Outcomes

- **SC-001**: A marketplace compaction plugin can be installed, activated, and configured via Settings.
- **SC-002**: The default compaction plugin can be deactivated and replaced by a marketplace plugin.
- **SC-003**: Plugin hooks run in registration order and can modify messages in the pipeline.
- **SC-004**: Plugin configuration changes take effect on next pipeline call without restart.
- **SC-005**: BOS ships with default memory and compaction plugins that work out of the box.

## Design Decisions

### 1. Plugin API Design

**Decision**: Use a generalized hook-based API (`BosPluginHooks`) with the following hooks:

```typescript
interface BosPluginHooks {
  beforeRun?: (messages: Message[], context: RunContext) => Promise<Message[]>;
  extendSystemPrompt?: (context: RunContext) => Promise<string | undefined>;
  beforeToolCall?: (call: TurnToolCall, context: RunContext) => Promise<ToolCallDecision | void>;
  afterToolCall?: (call: TurnToolCall, result: string, context: RunContext) => Promise<void>;
  afterRun?: (response: LLMResponse, context: RunContext) => Promise<LLMResponse>;
  onError?: (error: Error, context: RunContext) => Promise<void>;
  onRunFinished?: (summary: RunFinishSummary, context: RunContext) => Promise<void>;
}
```

**Rationale**: This generalizes to any subsystem (memory, compaction, telemetry, integrations) and follows the existing hook composition pattern in BOS.

### 2. Plugin Storage

**Decision**: Plugins live in `dataDir()/plugins/<id>/` and are loaded from `dataDir()/config/plugins.json`.

**Rationale**: User-owned, survives BOS updates, follows existing BOS patterns (memory, skills, apps all live in `dataDir()`).

### 3. Plugin Format

**Decision**: Plugins are CommonJS modules (no bundling, no build step).

**Rationale**: Plugin authors write plain JS, no tooling required, follows existing BOS patterns (skills are plain files).

### 4. Configuration UI

**Decision**: Hybrid approach — plugins can register a settings row in Settings → Plugins AND/OR bundle a configuration app.

**Rationale**: Settings provides a single place to manage all plugins, but plugins with complex config can provide their own rich UI.

### 5. Default Plugins

**Decision**: BOS ships with default memory and compaction plugins that are active by default but can be deactivated/swapped.

**Rationale**: Users get a working system out of the box, but can customize if they want.

## Touched Specs

| Spec | Path | What needs updating |
|------|------|-------------------|
| **000-browseros-core** | `bos-system-specs/000-browseros-core/` | Add plugin architecture section |
| **002-memory** | `bos-system-specs/002-memory/` | Memory system becomes optional/plugin |
| **003-self-improvement** | `bos-system-specs/003-self-improvement/` | Dependencies on memory system become soft |
| **009-installed-apps** | `bos-system-specs/009-installed-apps/` | Plugins can register config apps |
| **017-central-logging** | `bos-system-specs/017-central-logging/` | Plugins log to central logger |
| **021-memory-loops** | `bos-system-specs/021-memory-loops/` | Loops become plugin handlers |
| **022-context-compaction** | `bos-system-specs/022-context-compaction/` | Middleware becomes plugin hooks |
| **027-vfs-specfs-marketplace** | `bos-system-specs/027-vfs-specfs-marketplace/` | Server plugins as new marketplace item type |

## Implementation Plan

### Phase 1: Plugin Registry + Types

- [ ] Create `src/lib/plugins/types.ts` — Plugin interfaces
- [ ] Create `src/lib/plugins/registry.ts` — Global registry
- [ ] Create `src/lib/plugins/loader.ts` — Load from `dataDir()/plugins/`

### Phase 2: Extract Compaction

- [ ] Create `src/plugins/compaction/index.ts` — Compaction as plugin
- [ ] Refactor `src/lib/agent/compaction/middleware.ts` — Use plugin hooks
- [ ] Create `src/plugins/compaction/init.ts` — Register default plugin
- [ ] Migrate `dataDir()/config/compaction.json` to `dataDir()/config/plugins.json`

### Phase 3: Extract Memory

- [ ] Create `src/plugins/memory/index.ts` — Memory as plugin
- [ ] Refactor `src/lib/memory/memory.ts` — Use plugin hooks
- [ ] Create `src/plugins/memory/init.ts` — Register default plugin
- [ ] Migrate `dataDir()/config/memory.json` to `dataDir()/config/plugins.json`

### Phase 4: Settings Integration

- [ ] Create `src/components/apps/settings/PluginsTab.tsx` — Plugin management UI
- [ ] Add "Plugins" tab to Settings navigation
- [ ] Wire up plugin list, active toggle, reorder, config panel

### Phase 5: Marketplace Integration

- [ ] Update `src/lib/marketplace/client.ts` — Add `installServerPlugin()`
- [ ] Validate plugin manifest on install
- [ ] Add "Server Plugin" item type to Marketplace

### Phase 6: Testing + Migration

- [ ] Write tests for plugin registry
- [ ] Write tests for plugin pipeline
- [ ] Migrate existing config files
- [ ] Update documentation

## Acceptance Criteria

### Plugin Registry

- [ ] Plugins can be registered and unregistered via `registerPlugin()` / `unregisterPlugin()`
- [ ] Hooks are composed in registration order
- [ ] Plugin initialization is async and awaited

### Plugin Pipeline

- [ ] `beforeRun` hooks run before LLM call
- [ ] `extendSystemPrompt` hooks append to system prompt
- [ ] `beforeToolCall` / `afterToolCall` hooks intercept tool calls
- [ ] `afterRun` hooks process LLM response
- [ ] `onError` hooks handle exceptions
- [ ] `onRunFinished` hooks run after each call

### Settings UI

- [ ] Settings → Plugins shows pipeline list
- [ ] Plugin rows show name, version, description, active toggle, config button, uninstall button
- [ ] Pipeline list is reorderable (drag handles)
- [ ] Config panel renders form from `configSchema`
- [ ] Config changes save to `dataDir()/config/plugins.json`

### Marketplace Integration

- [ ] Marketplace supports "Server Plugin" item type
- [ ] Installing a plugin copies files to `dataDir()/plugins/<id>/`
- [ ] Activating a plugin adds it to `active` array and calls `registerPlugin()`
- [ ] Deactivating a plugin removes it from `active` array and calls `unregisterPlugin()`
- [ ] Uninstalling a plugin removes files and calls `unregisterPlugin()`

### Migration

- [ ] Existing compaction middleware uses plugin hooks
- [ ] Existing memory system uses plugin hooks
- [ ] Existing config files migrated to `dataDir()/config/plugins.json`
- [ ] Default plugins are active out of the box

## Notes

- This spec generalizes the chat pipeline into a hook-based architecture
- Plugins are optional — BOS ships with defaults but users can swap them out
- Marketplace integration enables third-party plugin distribution
- Settings UI provides configuration and management
