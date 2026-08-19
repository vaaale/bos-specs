# Tasks: Plugin Pipeline Architecture

**Spec**: `user-specs/026-plugin-pipeline/spec.md`
**Plan**: `user-specs/026-plugin-pipeline/plan.md`

**Created**: 2026-07-20

**Status**: Draft

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create `src/lib/plugins/` directory structure
- [ ] T002 Define plugin types in `src/lib/plugins/types.ts`
- [ ] T003 [P] Configure plugin registry in `src/lib/plugins/registry.ts`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core plugin infrastructure that MUST be complete before any plugin can be implemented

**⚠️ CRITICAL**: No plugin implementation can begin until this phase is complete

- [ ] T004 Create plugin loader in `src/lib/plugins/loader.ts`
- [ ] T005 [P] Implement plugin validator in `src/lib/plugins/validator.ts`
- [ ] T006 [P] Implement plugin monitor in `src/lib/plugins/monitor.ts`
- [ ] T007 Configure plugin sandbox in `src/lib/plugins/sandbox.ts`
- [ ] T008 Setup plugin config migration from existing config files

**Checkpoint**: Foundation ready - plugin implementation can now begin in parallel

---

## Phase 3: Extract Compaction (Priority: P1) 🎯 MVP

**Goal**: Refactor existing compaction middleware to use plugin hooks

**Independent Test**: Compaction works as before but implemented as a plugin

### Implementation for Compaction Plugin

- [ ] T009 [P] [US1] Create compaction plugin in `src/plugins/compaction/index.ts`
- [ ] T010 [P] [US1] Create compaction plugin init in `src/plugins/compaction/init.ts`
- [ ] T011 [US1] Refactor compaction middleware in `src/lib/agent/compaction/middleware.ts`
- [ ] T012 [US1] Extract compaction view into plugin
- [ ] T013 [US1] Extract compaction summarize into plugin
- [ ] T014 [US1] Extract compaction sidecar into plugin
- [ ] T015 [US1] Migrate compaction config to `dataDir()/config/plugins.json`

**Checkpoint**: Compaction plugin is fully functional and can be activated/deactivated

---

## Phase 4: Extract Memory (Priority: P1)

**Goal**: Refactor existing memory system to use plugin hooks

**Independent Test**: Memory works as before but implemented as a plugin

### Implementation for Memory Plugin

- [ ] T016 [P] [US2] Create memory plugin in `src/plugins/memory/index.ts`
- [ ] T017 [P] [US2] Create memory plugin init in `src/plugins/memory/init.ts`
- [ ] T018 [US2] Refactor memory system in `src/lib/memory/memory.ts`
- [ ] T019 [US2] Extract memory loop into plugin
- [ ] T020 [US2] Migrate memory config to `dataDir()/config/plugins.json`

**Checkpoint**: Memory plugin is fully functional and can be activated/deactivated

---

## Phase 5: Settings Integration (Priority: P2)

**Goal**: Create Settings → Plugins tab for managing plugins

**Independent Test**: Users can view, configure, and manage plugins in Settings

### Implementation for Settings UI

- [ ] T021 [P] [US3] Create PluginsTab component in `src/components/apps/settings/PluginsTab.tsx`
- [ ] T022 [US3] Add "Plugins" tab to Settings navigation
- [ ] T023 [US3] Create settings registration loader in `src/lib/plugins/settings.ts`
- [ ] T024 [US3] Implement plugin config form rendering
- [ ] T025 [US3] Implement plugin config save functionality
- [ ] T026 [US3] Implement plugin reorder functionality

**Checkpoint**: Settings → Plugins tab is fully functional

---

## Phase 6: Marketplace Integration (Priority: P2)

**Goal**: Add server plugin support to the Marketplace

**Independent Test**: Users can install plugins from Marketplace and activate them

### Implementation for Marketplace

- [ ] T027 [P] [US4] Add `installServerPlugin()` to `src/lib/marketplace/client.ts`
- [ ] T028 [P] [US4] Update marketplace manifest validation
- [ ] T029 [US4] Add "Server Plugin" item type to Marketplace UI
- [ ] T030 [US4] Implement plugin activation/deactivation in Settings
- [ ] T031 [US4] Implement plugin uninstall in Settings

**Checkpoint**: Marketplace supports installing and managing plugins

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple plugins

- [ ] T032 [P] Documentation updates in `docs/developer/plugins.md`
- [ ] T033 [P] Code cleanup and refactoring
- [ ] T034 [P] Performance optimization across all plugins
- [ ] T035 Security hardening for plugin execution
- [ ] T036 Run quickstart.md validation

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all plugins
- **Compaction Plugin (Phase 3)**: Depends on Foundational phase completion
- **Memory Plugin (Phase 4)**: Depends on Foundational phase completion (can run in parallel with Phase 3)
- **Settings Integration (Phase 5)**: Depends on Compaction and Memory plugins
- **Marketplace Integration (Phase 6)**: Depends on Settings Integration
- **Polish (Phase 7)**: Depends on all plugins being complete

### Parallel Opportunities

- Phase 3 (Compaction) and Phase 4 (Memory) can run in parallel
- All Phase 1 tasks marked [P] can run in parallel
- All Phase 2 tasks marked [P] can run in parallel

---

## Implementation Strategy

### MVP First (Compaction Plugin Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all plugins)
3. Complete Phase 3: Compaction Plugin
4. **STOP and VALIDATE**: Test Compaction Plugin independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add Compaction Plugin → Test independently → Deploy/Demo (MVP!)
3. Add Memory Plugin → Test independently → Deploy/Demo
4. Add Settings Integration → Test independently → Deploy/Demo
5. Add Marketplace Integration → Test independently → Deploy/Demo
6. Polish → Final deployment

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: Compaction Plugin (Phase 3)
   - Developer B: Memory Plugin (Phase 4)
3. After both plugins complete:
   - Developer C: Settings Integration (Phase 5)
   - Developer D: Marketplace Integration (Phase 6)
4. All developers: Polish (Phase 7)

---

## Notes

- Each phase should be independently completable and testable
- Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate independently
- Avoid: vague tasks, same file conflicts, cross-phase dependencies that break independence
