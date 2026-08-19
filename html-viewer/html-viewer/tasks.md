---

description: "Task list for HTML Viewer feature implementation"
---

# Tasks: HTML Viewer

**Input**: Design documents from `/user-specs/html-viewer/`

**Prerequisites**: plan.md (required), spec.md (required)

**Tests**: Security and integration tests are REQUIRED per specification requirements.

**Organization**: Tasks are grouped by implementation phase with clear dependencies.

## Format: `[ID] [P?] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- Include exact file paths in descriptions

---

## Phase 1: Component Development

**Goal**: Create the `HtmlViewer` React component with full UI and VFS integration

### T001 [P] Create HtmlViewer component structure

Create `src/components/HtmlViewer/HtmlViewer.tsx` with:
- Component interface defining props: `source` (string), `title` (string, optional), `onClose` (callback)
- Internal state: `isLoading` (boolean), `error` (string | null), `contentUrl` (string | null)
- Basic JSX structure with placeholder for header and content area
- Export from `src/components/HtmlViewer/index.ts`

**Acceptance Criteria**:
- Component compiles without TypeScript errors
- Props interface matches specification (FR-004, FR-008)
- State variables initialized correctly
- File exists at correct path

---

### T002 [P] Implement HtmlViewer header bar UI

In `src/components/HtmlViewer/HtmlViewer.tsx`:
- Create fixed-height header bar (~40px) with title text on left
- Add fullscreen toggle icon button on right
- Add close (X) icon button on right
- Apply BrowserOS theme CSS variables for styling
- Create `src/components/HtmlViewer/HtmlViewer.styles.ts` for styles

**Acceptance Criteria**:
- Header displays configurable title or defaults to "Preview"
- Fullscreen button visible and clickable
- Close button visible and clickable
- Styles match BrowserOS dark/light theme
- Layout is responsive (header maintains height on resize)

---

### T003 [P] Implement content area with iframe

In `src/components/HtmlViewer/HtmlViewer.tsx`:
- Create flex-grow content area below header
- Add `<iframe>` element that fills 100% width/height
- Configure iframe with sandbox attributes: `sandbox="allow-scripts allow-same-origin"`
- Add loading spinner that shows when `isLoading` is true
- Add error display component that shows when `error` is not null

**Acceptance Criteria**:
- Iframe uses correct sandbox attributes (NFR-001, FR-006)
- Loading spinner visible during content load
- Error message displays with "Close" button when load fails (FR-008)
- No scrollbars on outer window (iframe handles internal scrolling)

---

### T004 [P] Implement fullscreen toggle functionality

In `src/components/HtmlViewer/HtmlViewer.tsx`:
- Add state for `isFullscreen` (boolean)
- Wire fullscreen button to toggle `isFullscreen` state
- Apply fullscreen CSS class when active (iframe fills entire window)
- Update button icon to show exit fullscreen when in fullscreen mode

**Acceptance Criteria**:
- Clicking fullscreen button expands iframe to fill available space (FR-005)
- Clicking again returns to normal size
- Button icon updates to reflect current state
- No layout shifts or flickering during toggle

---

### T005 [P] Implement VFS file reading utility

Create `src/services/vfsReader.ts` with:
- Function `readVfsFile(filePath: string): Promise<string>` that reads file content from VFS
- Function `generateBlobUrl(content: string, mimeType: string = 'text/html'): string` that creates Blob URL
- Error handling for invalid paths or read failures

**Acceptance Criteria**:
- Function successfully reads HTML files from VFS paths like `/tmp/mockup.html`
- Returns content as string on success
- Throws descriptive error on file not found or permission issues
- Blob URL generation works correctly with HTML content

---

### T006 [P] Integrate VFS reading into HtmlViewer component

In `src/components/HtmlViewer/HtmlViewer.tsx`:
- Add `useEffect` hook that watches `source` prop changes
- When source starts with `/` (VFS path): call `readVfsFile`, then `generateBlobUrl`, set `contentUrl`
- When source is URL: set `contentUrl` directly to source
- Set `isLoading` to true before load, false after success or error
- Catch and set errors appropriately

**Acceptance Criteria**:
- VFS paths like `/tmp/file.html` are read and converted to Blob URLs (FR-002)
- External URLs are passed through directly
- Loading state toggles correctly during async operations
- Errors are captured and displayed in UI (FR-008)

---

### T007 [P] Implement error handling UI

In `src/components/HtmlViewer/HtmlViewer.tsx`:
- Create error display component with:
  - Error message text (e.g., "File not found: /tmp/mockup.html")
  - "Close" button that calls `onClose`
  - Optional "Reload" button for recoverable errors
- Handle specific error types: file not found, invalid path, CORS errors

**Acceptance Criteria**:
- Clear error message displayed for each failure scenario (FR-008)
- Close button closes the window when clicked
- Error UI matches BrowserOS design patterns
- Specific error messages help diagnose issues

---

### T008 [P] Write unit tests for HtmlViewer component

Create `tests/unit/components/HtmlViewer.test.tsx` with:
- Test: Component renders with default props (title="Preview")
- Test: Component renders with custom title
- Test: Loading spinner shows during load
- Test: Error message shows on failure
- Test: Iframe has correct sandbox attributes
- Test: Fullscreen toggle changes state

**Acceptance Criteria**:
- All tests pass
- Tests cover all component states (loading, success, error)
- Sandbox attributes verified in rendered output
- Component exports work correctly

---

## Phase 2: Tool Integration

**Goal**: Define and register the `openPreview` tool with Agent Runtime

### T009 [P] Create openPreview tool definition

Create `src/tools/openPreview.ts` with:
- Tool schema object matching specification:
  - `name`: "openPreview"
  - `description`: "Opens a system preview window for HTML content..."
  - `inputSchema` with properties: `filePath`, `url`, `title`
  - Validation logic: at least one of `filePath` or `url` required
- Handler function signature matching Agent Tool Registry interface

**Acceptance Criteria**:
- Schema matches specification exactly (FR-001, FR-002)
- TypeScript types are correct
- Input validation rejects invalid inputs (neither filePath nor url provided)
- File compiles without errors

---

### T010 [P] Implement openPreview tool handler logic

In `src/tools/openPreview.ts` handler function:
- Validate input: ensure `filePath` OR `url` is provided
- If `filePath`: resolve VFS path, read file content, generate Blob URL
- If `url`: use directly as source
- Call Window Manager API to spawn system view with `HtmlViewer` component
- Pass `source` (Blob URL or URL), `title`, and `onClose` handler to component
- Return success message with window ID

**Acceptance Criteria**:
- Handler correctly processes VFS file paths (FR-002)
- Handler correctly processes external URLs
- Window spawns with correct component and props (FR-003)
- Returns structured response with window ID on success
- Returns error response on failure (invalid path, read error)

---

### T011 [P] Register openPreview tool with Agent Runtime

In appropriate registration file (e.g., `src/tools/index.ts` or tool registry config):
- Import `openPreview` tool definition
- Register tool with Agent Tool Registry
- Ensure tool is discoverable by agents via runtime API

**Acceptance Criteria**:
- Tool appears in available tools list for agents
- Agents can invoke tool via standard tool calling interface
- Tool schema is visible to agents for parameter completion
- Registration does not break existing tools

---

### T012 [P] Write unit tests for openPreview tool

Create `tests/unit/tools/openPreview.test.ts` with:
- Test: Tool schema validation (required fields, types)
- Test: Handler succeeds with valid filePath
- Test: Handler succeeds with valid url
- Test: Handler fails when neither filePath nor url provided
- Test: Handler handles VFS read errors gracefully
- Test: Response includes window ID on success

**Acceptance Criteria**:
- All tests pass
- Tool schema validation works correctly
- Error handling tested for all failure modes
- Integration with Window Manager mocked appropriately

---

## Phase 3: Window Manager Integration

**Goal**: Add system view support to Window Manager and connect to HtmlViewer

### T013 [P] Define SystemView window type

In `src/windowManager/types.ts`:
- Add new window type: `SystemView` or extend existing type union
- Define interface for system view windows:
  - `type: 'system-view'`
  - `component`: Component reference (HtmlViewer)
  - `props`: Props object passed to component
  - `excludeFromDock`: boolean (should be true for system views)
  - `excludeFromSwitcher`: boolean (should be true for system views)

**Acceptance Criteria**:
- New type integrates with existing Window Manager type system
- Interface includes all required properties for HtmlViewer
- Type is exported and available to other modules
- No breaking changes to existing window types

---

### T014 [P] Implement system view spawn logic

In `src/windowManager/hooks.ts` or spawn API:
- Add function `spawnSystemView(component: string, props: object): WindowId`
- Create window with `type: 'system-view'`
- Set `excludeFromDock: true` and `excludeFromSwitcher: true`
- Register component reference and props
- Return new window ID

**Acceptance Criteria**:
- System view windows are created successfully (FR-003)
- Windows do not appear in dock (FR-007)
- Windows do not appear in app switcher (Alt+Tab) unless configured otherwise
- Window ID is returned for tracking/closing

---

### T015 [P] Register HtmlViewer as system component

In `src/windowManager/registry.ts` or component registry:
- Import `HtmlViewer` component
- Register component with name "HtmlViewer" or similar identifier
- Map component name to actual React component for window manager
- Ensure component is available for system view spawning

**Acceptance Criteria**:
- HtmlViewer is registered and discoverable by Window Manager
- Component name matches what tool handler uses
- Registration does not affect existing app components
- Component can be spawned via system view API

---

### T016 [P] Connect openPreview tool to Window Manager

In `src/tools/openPreview.ts` or integration point:
- Import Window Manager spawn system view function
- Update handler to call `spawnSystemView('HtmlViewer', { source, title, onClose })`
- Implement `onClose` callback that calls Window Manager close API
- Handle spawn failures gracefully

**Acceptance Criteria**:
- Tool successfully spawns HtmlViewer windows (FR-003)
- Close button in window calls proper cleanup
- Spawn failures return error to agent
- Integration is clean with no tight coupling

---

### T017 [P] Write integration tests for Window Manager integration

Create `tests/integration/html-viewer-integration.test.ts` with:
- Test: System view spawns correctly with HtmlViewer component
- Test: Window does not appear in dock list
- Test: Window does not appear in app switcher
- Test: Close button destroys window properly
- Test: Fullscreen toggle works on spawned window

**Acceptance Criteria**:
- All tests pass
- Integration between tool, Window Manager, and component verified
- System view behavior matches specification (FR-007)
- Cleanup works correctly on close

---

## Phase 4: Security Testing & Verification

**Goal**: Verify sandbox security and edge case handling

### T018 [P] Create sandbox security test suite

Create `tests/security/sandbox-verification.test.ts` with tests for:
- Test: Rendered content cannot access `window.parent`
- Test: Scripts in iframe cannot inject into parent context
- Test: iframe sandbox attributes prevent top navigation
- Test: External network requests from iframe are restricted (unless allowed)
- Test: Form submissions are blocked (no `allow-forms`)

**Acceptance Criteria**:
- All security tests pass
- Sandbox attributes verified to block dangerous operations (NFR-001, FR-006)
- Malicious script attempts are contained within iframe
- No cross-context communication possible

---

### T019 [P] Test with malicious HTML content

Create test files in `tests/security/fixtures/`:
- `malicious-parent-access.html`: Attempts to access `window.parent`
- `malicious-injection.html`: Attempts to inject into parent DOM
- `malicious-navigation.html`: Attempts top navigation
- Run HtmlViewer with these files and verify they fail safely

**Acceptance Criteria**:
- Malicious scripts execute within iframe but cannot affect parent
- No console errors in parent context
- Security warnings logged if injection attempts detected
- Window remains functional after malicious content load

---

### T020 [P] Test edge cases: invalid paths and CORS errors

Create integration tests for:
- Test: Invalid VFS path shows "File not found" error (FR-008)
- Test: Non-existent URL shows "Failed to load external resource" error
- Test: CORS-blocked URL shows appropriate error message
- Test: Malformed HTML renders (browser parsing) or shows generic error

**Acceptance Criteria**:
- All error scenarios display user-friendly messages (FR-008)
- No unhandled exceptions in component
- Error UI provides clear recovery options (Close, Reload)
- Edge cases covered in test suite

---

### T021 [P] Test large file handling

Create tests for:
- Test: File < 5MB loads within 200ms (NFR-002)
- Test: File > 10MB shows warning before rendering
- Test: Memory usage stays reasonable during load
- Test: Very large files don't crash the browser

**Acceptance Criteria**:
- Performance meets specification for small files (NFR-002)
- Large file warning displayed appropriately
- No memory leaks or crashes with large content
- Loading indicator shows during slow operations

---

### T022 [P] Test responsive layout behavior

Create tests for:
- Test: Window resize updates iframe dimensions correctly
- Test: Header maintains fixed height during resize
- Test: Fullscreen mode adapts to different screen sizes
- Test: No horizontal scrollbars appear on resize

**Acceptance Criteria**:
- Layout responds correctly to window resizing (NFR-003)
- Iframe fills available space without overflow
- Responsive behavior works across viewport sizes
- CSS is robust against edge case dimensions

---

## Phase 5: Final Integration & Validation

**Goal**: End-to-end validation and cleanup

### T023 [P] End-to-end agent workflow test

Create `tests/integration/agent-workflow.test.ts` with:
- Simulate agent creating HTML file in VFS (`/tmp/test.html`)
- Agent calls `openPreview` tool with filePath
- Verify window spawns with correct content
- Verify user can interact with rendered content
- Verify close works and cleanup occurs

**Acceptance Criteria**:
- Full agent workflow completes successfully (Success Criteria #1)
- No manual intervention required
- Content renders correctly in spawned window
- All components integrate seamlessly

---

### T024 [P] Documentation updates

Update or create:
- `src/components/HtmlViewer/README.md`: Component usage guide
- `src/tools/openPreview.md`: Tool documentation for agents
- Update BrowserOS developer docs with new system view type

**Acceptance Criteria**:
- Documentation is accurate and complete
- Usage examples provided for component and tool
- Developer onboarding documents updated
- Agent-facing documentation clear and actionable

---

### T025 [P] Code cleanup and final review

- Remove any console.log debug statements
- Ensure all TypeScript strict mode compliance
- Run linting and formatting across all new files
- Verify no unused imports or variables
- Check for consistent error handling patterns

**Acceptance Criteria**:
- Code passes all linting checks
- No TypeScript warnings
- Consistent code style throughout
- Ready for merge review

---

## Dependencies & Execution Order

### Phase Dependencies

| Phase | Depends On | Blocks |
|-------|------------|--------|
| Phase 1: Component Development | None | Phase 2, Phase 3 |
| Phase 2: Tool Integration | Phase 1 (component must exist) | Phase 4 (testing) |
| Phase 3: Window Manager Integration | Phase 1 (component must exist) | Phase 4 (testing) |
| Phase 4: Security Testing | Phase 2 + Phase 3 complete | Phase 5 |
| Phase 5: Final Integration | All previous phases complete | None |

### Within-Phase Parallel Opportunities

**Phase 1**: T001-T008 can largely run in parallel:
- T001 (structure) must complete before T002-T007
- T002, T003, T004, T005 can run in parallel after T001
- T006 depends on T002, T003, T005
- T007 depends on T003
- T008 (tests) can run after T001-T007 complete

**Phase 2**: T009-T012:
- T009 (schema) and T010 (handler) can run in parallel
- T011 (registration) depends on T009, T010
- T012 (tests) depends on T009-T011

**Phase 3**: T013-T017:
- T013 (types) must complete before T014, T015
- T014 (spawn) and T015 (register) can run in parallel after T013
- T016 (integration) depends on T014, T015
- T017 (tests) depends on T013-T016

**Phase 4**: T018-T022:
- All security tests can run in parallel once Phases 2+3 complete
- Edge case tests (T020) can run independently
- Performance tests (T021) can run independently
- Responsive tests (T022) can run independently

**Phase 5**: T023-T025:
- T023 (e2e test) requires all previous phases complete
- T024 (docs) can run in parallel with T023
- T025 (cleanup) runs last after all testing

### Critical Path

```
T001 → T002/T003/T005 → T006 → T009/T010 → T013 → T014/T015 → T016 → T018-T022 → T023
```

### Recommended Execution Order (Single Developer)

1. **Phase 1**: T001 → T002 → T003 → T005 → T006 → T004 → T007 → T008
2. **Phase 2**: T009 → T010 → T011 → T012
3. **Phase 3**: T013 → T014 → T015 → T016 → T017
4. **Phase 4**: T018 → T019 → T020 → T021 → T022
5. **Phase 5**: T023 → T024 → T025

---

## Acceptance Criteria Summary

Upon completion of all tasks:

1. ✅ **Agent Workflow**: Agent can create HTML file, call `openPreview`, see content in window
2. ✅ **Security**: Rendered content cannot access BrowserOS internals (sandbox verified)
3. ✅ **Usability**: Window closes cleanly, title bar intuitive, fullscreen works
4. ✅ **No App Pollution**: Viewer does not appear in dock or installed apps list
5. ✅ **Performance**: Files < 5MB load within 200ms
6. ✅ **Error Handling**: Clear messages for all failure scenarios
7. ✅ **All Tests Pass**: Unit, integration, and security tests passing

---

## Notes

- [P] tasks = different files, no dependencies, can run in parallel
- Security testing is REQUIRED per specification (NFR-001, FR-006)
- Each phase should be validated before proceeding to next phase
- Commit after each task or logical group of tasks
- Stop at Phase 4 checkpoint to validate security before final integration
