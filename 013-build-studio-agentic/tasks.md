# Tasks: Build Studio Agentic V2 — Iterative App Design

**Input**: `specs/bos-system-specs/013-build-studio-agentic/spec.md`, `plan.md`

**Prerequisites**: spec.md and plan.md reviewed and approved.

**Organization**: Tasks grouped by implementation phase / user story. Phases are dependency-ordered: P1 (registry) must complete before P2/P3/P4.

## Phase 1: Setup

**Purpose**: Verify the active feature branch and ensure the Developer has the required context.

- [ ] T001 Confirm the active feature branch is `bos/bs-design-process`; if not, ask the user to switch/create it before implementation.
- [ ] T002 Ensure `specs/bos-system-specs/013-build-studio-agentic/spec.md` and `plan.md` are readable from the feature worktree.

---

## Phase 2: Foundational — Surface-tools registry (User Story 10)

**Purpose**: Implement the client-side registry that lets mounted app windows register runtime (Tier 2) assistant tools. This blocks every later phase.

**Independent Test**: Open two apps with different surface tools; start an assistant run and verify that only the currently-open windows' tools are included in `surfaceTools`.

### Implementation

- [ ] T003 Create `src/lib/assistant/client/surface-tools.ts` with:
  - `registerAppSurfaceTools(windowId, declarations, handlers)`
  - `unregisterAppSurfaceTools(windowId)`
  - `getActiveSurfaceToolDeclarations()`
  - `dispatchSurfaceToolCall(windowId, name, args)`
- [ ] T004 Update `src/lib/assistant/client/run-client.ts` to call `getActiveSurfaceToolDeclarations()` and merge the result into the run's `surfaceTools` automatically.
- [ ] T005 Update `src/components/agent/v2/AssistantChatV2.tsx` so it no longer requires `surfaceTools` to be passed explicitly; read them from the registry.
- [ ] T006 Update `src/components/agent/v2/ChatInputV2.tsx` to read surface tools from the registry instead of receiving them as props.
- [ ] T007 Refactor Build Studio's existing surface tools (`buildstudio_artifact_open`, `buildstudio_tree_refresh`) to register through the registry in `src/apps/build-studio/index.tsx` on mount. Keep their existing handlers intact.
- [ ] T008 Run `npx tsc --noEmit` and `npm run lint`; fix every error before finishing this phase.

**Checkpoint**: All existing Build Studio surface tools still work; registry can be demonstrated with a minimal test app.

---

## Phase 3: UI Preview app and Tier 1/Tier 2 tools (User Stories 9 & 10)

**Purpose**: Create the UI Preview built-in app that renders A2UI surfaces and exposes the tools the agent uses during UI design.

**Independent Test**: From the assistant, call `bos_app_launch("ui-preview")` and `ui_preview_render` with a sample A2UI operations envelope; verify the surface renders inside the app window.

### Tests

- [ ] T009 [P] Add Playwright test `e2e/013-ui-preview.spec.ts` that opens UI Preview and asserts the surface container mounts.

### Implementation

- [ ] T010 Create `src/apps/ui-preview/manifest.ts` with `id: "ui-preview"`, singleton, lucide icon, and default window size.
- [ ] T011 Create `src/apps/ui-preview/index.tsx`:
  - Host the `@copilotkit/a2ui-renderer` component.
  - Maintain a local state for the current surface id and operations history.
  - Render a design context panel (active requirement, iteration history, notes) per FR-021.
  - Register Tier 2 surface tools on mount via the registry from Phase 2.
- [ ] T012 Create `src/apps/ui-preview/agent-tools-v2.ts` with Tier 2 tool declarations and handlers:
  - `ui_preview_render(surfaceId, operations)` — apply A2UI operations to the renderer.
  - `ui_preview_show_requirement(specPath, requirementId)` — call `buildstudio_artifact_open` with the spec path and requirement anchor.
- [ ] T013 Register `ui_preview_open` as a Tier 1 installed-app tool in `src/lib/agent/capabilities-registry.ts` (or via the UI Preview manifest/static `agent-tools.ts` per FR-019).
- [ ] T014 Implement the `ui_preview_open` handler to open or focus the UI Preview window.
- [ ] T015 Run `npm run gen:apps` so the new built-in app is discovered and manifests/components are regenerated.
- [ ] T016 Run `npx tsc --noEmit` and `npm run lint`; fix every error before finishing this phase.

**Checkpoint**: Agent can open UI Preview and push A2UI operations to it; the surface updates in place.

---

## Phase 4: `a2ui_render` server tool (User Story 9)

**Purpose**: Provide a server tool that generates validated A2UI operations using `@ag-ui/a2ui-toolkit` and the configured BOS provider/model.

**Independent Test**: Call `a2ui_render` with a simple UI description and verify it returns a valid operations envelope that the UI Preview app can render.

### Implementation

- [ ] T017 Create `src/lib/assistant/tools/server/a2ui-render.ts`:
  - Declare the `a2ui_render` server tool.
  - Use `@ag-ui/a2ui-toolkit` to run a constrained sub-agent.
  - Sub-agent prompt includes BOS style guide and A2UI catalog rules from `data/skills/bos-app/references/a2ui-catalog.md`.
  - Validate the returned operations envelope and surface id.
  - Use the active BOS provider/model configuration.
- [ ] T018 Register `a2ui_render` in `src/lib/agent/capabilities-registry.ts` with the correct group/context and schema.
- [ ] T019 Add error handling and recovery: if the sub-agent returns invalid operations, retry once with a stricter prompt; if still invalid, return a clear error.
- [ ] T020 Run `npx tsc --noEmit` and `npm run lint`; fix every error before finishing this phase.

**Checkpoint**: Agent can call `a2ui_render` and receive a surface operations envelope.

---

## Phase 5: Spec viewer anchors and transient highlights (User Story 8)

**Purpose**: Let `buildstudio_artifact_open` scroll to a specific section and transiently highlight it so the user sees live spec updates.

**Independent Test**: Open a spec with `buildstudio_artifact_open(path, anchor="some-heading")`; verify the viewer scrolls to the heading and briefly highlights it.

### Tests

- [ ] T021 [P] Add Playwright test `e2e/013-spec-anchor.spec.ts` that opens a spec at a heading anchor and asserts the element is highlighted.

### Implementation

- [ ] T022 Extend the parameter schema for `buildstudio_artifact_open` in `src/apps/build-studio/agent-tools-v2.ts` to accept an optional `anchor` string (heading slug or section id).
- [ ] T023 Update `src/apps/build-studio/index.tsx`:
  - When opening a spec with an `anchor`, find the matching heading/section element after the Markdown renders.
  - Scroll the element into view.
  - Apply a transient highlight CSS class; remove it after ~3 seconds.
- [ ] T024 Generate stable heading anchors from Markdown headings in the Build Studio Markdown renderer (if not already available via `id` attributes).
- [ ] T025 Run `npx tsc --noEmit` and `npm run lint`; fix every error before finishing this phase.

**Checkpoint**: Agent can open a spec at a specific section and the user sees a transient highlight.

---

## Phase 6: Integration & Polish

**Purpose**: Wire everything together, update docs, and validate the end-to-end flow.

- [ ] T026 Update `src/apps/build-studio/agent-tools-v2.ts` and `src/apps/build-studio/index.tsx` so Build Studio's surface tools use the new registry and anchor support cleanly.
- [ ] T027 Update `seed/agents/build-studio/AGENT.md` and `data/agents/build-studio/AGENT.md` to include `a2ui_render`, `ui_preview_render`, and `ui_preview_show_requirement` in the tools allowlist once they exist.
- [ ] T028 Update `docs/dev/guides/apps.md` and `docs/dev/guides/features-and-components.md` with the new surface-tools and UI Preview patterns.
- [ ] T029 [P] Run `npx tsc --noEmit` and `npm run lint` across the whole branch; fix every error.
- [ ] T030 [P] Run the Playwright tests from T009 and T021.
- [ ] T031 End-to-end validation: from the assistant in Build Studio, run a minimal `bos-app` design session and verify interview → spec update + highlight → UI Preview open/render → delegate flows without errors.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies.
- **Phase 2 (Surface-tools registry)**: Blocks Phase 3, 4, 5.
- **Phase 3 (UI Preview app)**: Depends on Phase 2.
- **Phase 4 (`a2ui_render`)**: Depends on Phase 3 (uses UI Preview for validation).
- **Phase 5 (Spec viewer anchors)**: Independent of Phase 3/4 but should be validated with the spec updates from a `bos-app` session.
- **Phase 6 (Polish)**: Depends on all implementation phases.

### Within Each Phase

- Tests first (if included).
- Models/types before services/endpoints.
- `tsc`/`lint` before declaring the phase done.

### Parallel Opportunities

- Phase 3 UI Preview component and Phase 5 spec anchor work can be developed in parallel once Phase 2 is done.
- T009/T021 tests can be drafted in parallel with their story implementations.
- Phase 4 can proceed once the UI Preview surface is rendering sample operations.

## Implementation Strategy

### MVP First

1. Complete Phase 2 (registry).
2. Complete Phase 3 (UI Preview app + tools).
3. **STOP and validate**: agent can open UI Preview and render a static A2UI surface.
4. Complete Phase 4 (`a2ui_render`).
5. Complete Phase 5 (anchors/highlight).
6. Complete Phase 6 (polish + docs + e2e validation).

### Suggested Developer Brief

"Implement P1–P4 of `specs/bos-system-specs/013-build-studio-agentic/plan.md` in order. Start with the surface-tools registry, then the UI Preview app, then `a2ui_render`, then spec anchors. Keep all changes on `bos/bs-design-process`. Run `npx tsc --noEmit` and `npm run lint` after every phase and fix all errors. Add/update docs under `docs/dev/guides/` as needed. Do not merge or promote branches."
