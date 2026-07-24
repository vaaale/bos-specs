# Plan: Plugin Pipeline Architecture

**Spec**: `user-specs/026-plugin-pipeline/spec.md`

**Created**: 2026-07-20

**Status**: Draft

## Summary

This plan breaks down the plugin pipeline architecture into implementation tasks. The goal is to create a generalized hook-based plugin API that allows subsystems (memory, compaction, telemetry, integrations) to be extracted into optional plugins.

## Tasks

### Task 1: Plugin Registry + Types

**Description**: Create the core plugin infrastructure — types, registry, and loader.

**Files**:
- `src/lib/plugins/types.ts` — Plugin interfaces (BosPluginHooks, PluginDefinition, PluginContext, RunContext)
- `src/lib/plugins/registry.ts` — Global registry (registerPlugin, unregisterPlugin, composeHooks)
- `src/lib/plugins/loader.ts` — Load plugins from `dataDir()/plugins/`

**Acceptance Criteria**:
- [ ] Plugin types are defined with all required interfaces
- [ ] Registry supports register/unregister/compose
- [ ] Loader loads plugins from `dataDir()/plugins/` on BOS startup
- [ ] Tests cover registry operations

---

### Task 2: Extract Compaction to Plugin

**Description**: Refactor the existing compaction middleware to use plugin hooks.

**Files**:
- `src/plugins/compaction/index.ts` — Compaction as plugin (beforeRun hook)
- `src/plugins/compaction/init.ts` — Register default compaction plugin
- `src/lib/agent/compaction/middleware.ts` — Refactor to use plugin hooks
- `src/lib/agent/compaction/view.ts` — Extract into plugin
- `src/lib/agent/compaction/summarize.ts` — Extract into plugin
- `src/lib/agent/compaction/sidecar.ts` — Extract into plugin

**Acceptance Criteria**:
- [ ] Compaction is implemented as a plugin with beforeRun hook
- [ ] Compaction plugin is registered at BOS startup
- [ ] Existing compaction behavior is preserved (no behavior change)
- [ ] Compaction config is migrated to `dataDir()/config/plugins.json`
- [ ] Tests cover compaction plugin lifecycle

---

### Task 3: Extract Memory to Plugin

**Description**: Refactor the existing memory system to use plugin hooks.

**Files**:
- `src/plugins/memory/index.ts` — Memory as plugin (afterRun hook for fast memory loop)
- `src/plugins/memory/init.ts` — Register default memory plugin
- `src/lib/memory/memory.ts` — Refactor to use plugin hooks
- `src/lib/memory/memory-loop.ts` — Extract into plugin

**Acceptance Criteria**:
- [ ] Memory system is implemented as a plugin with afterRun hook
- [ ] Memory plugin is registered at BOS startup
- [ ] Existing memory behavior is preserved (no behavior change)
- [ ] Memory config is migrated to `dataDir()/config/plugins.json`
- [ ] Tests cover memory plugin lifecycle

---

### Task 4: Settings Integration

**Description**: Create the Settings → Plugins tab for managing plugins.

**Files**:
- `src/components/apps/settings/PluginsTab.tsx` — Plugin management UI
- `src/components/apps/settings/index.tsx` — Add "Plugins" tab to navigation
- `src/lib/plugins/settings.ts` — Load settings registration from plugins

**Acceptance Criteria**:
- [ ] Settings → Plugins tab shows pipeline list
- [ ] Plugin rows show name, version, description, active toggle, config button, uninstall button
- [ ] Pipeline list is reorderable (drag handles)
- [ ] Config panel renders form from `configSchema`
- [ ] Config changes save to `dataDir()/config/plugins.json`
- [ ] Tests cover Settings UI interactions

---

### Task 5: Marketplace Integration

**Description**: Add server plugin support to the Marketplace.

**Files**:
- `src/lib/marketplace/client.ts` — Add `installServerPlugin()`
- `src/lib/marketplace/manifest.ts` — Validate plugin manifests
- `src/components/apps/marketplace/MarketplaceApp.tsx` — Add "Server Plugin" item type

**Acceptance Criteria**:
- [ ] Marketplace supports "Server Plugin" item type
- [ ] Installing a server plugin copies files to `dataDir()/plugins/<id>/`
- [ ] Installing validates the plugin manifest
- [ ] Activating a plugin adds it to the `active` array in `dataDir()/config/plugins.json`
- [ ] Deactivating a plugin removes it from the `active` array
- [ ] Uninstalling a plugin removes files and calls `unregisterPlugin()`
- [ ] Tests cover marketplace plugin installation

---

### Task 6: Plugin Security + Monitoring

**Description**: Add security restrictions and hang detection for plugins.

**Files**:
- `src/lib/plugins/validator.ts` — Validate plugin manifests
- `src/lib/plugins/monitor.ts` — Monitor plugin execution for hangs
- `src/lib/plugins/sandbox.ts` — Restrict plugin access to `dataDir()`

**Acceptance Criteria**:
- [ ] Plugin manifests are validated on install
- [ ] Plugins cannot access BOS source code (only `dataDir()`)
- [ ] Plugin errors are caught by `onError` hooks
- [ ] Plugin hangs are detected and terminated
- [ ] Tests cover security and monitoring

---

### Task 7: Documentation + Migration

**Description**: Update documentation and migrate existing config files.

**Files**:
- `docs/developer/plugins.md` — Plugin development guide
- `docs/user/marketplace.md` — Update marketplace documentation
- `src/lib/agent/compaction/config.ts` — Migrate to `dataDir()/config/plugins.json`
- `src/lib/memory/memory.ts` — Migrate to `dataDir()/config/plugins.json`

**Acceptance Criteria**:
- [ ] Plugin development guide is written
- [ ] Marketplace documentation is updated
- [ ] Existing config files are migrated to `dataDir()/config/plugins.json`
- [ ] Documentation is reviewed and approved

---

## Dependencies

- Task 1 must complete before Tasks 2-3 (plugin registry is required)
- Task 2 must complete before Task 4 (compaction plugin is needed for Settings)
- Task 3 must complete before Task 4 (memory plugin is needed for Settings)
- Task 4 must complete before Task 5 (Settings UI is needed for marketplace activation)
- Task 5 must complete before Task 6 (marketplace integration is needed for security)
- Task 6 must complete before Task 7 (security is needed for documentation)

## Risks

- **Risk 1**: Plugin API design may need iteration based on developer feedback
- **Risk 2**: Migration of existing config files may break user data
- **Risk 3**: Plugin security model may need refinement based on testing

## Open Questions

- Should plugins be able to register new settings tabs (not just rows)?
- Should plugins be able to register new apps (not just config apps)?
- How should plugin updates be handled (automatic vs manual)?
