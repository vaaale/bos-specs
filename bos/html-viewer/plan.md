# Implementation Plan: HTML Viewer

**Branch**: `html-viewer` | **Date**: 2024-01-15 | **Spec**: `user-specs/html-viewer/spec.md`

**Input**: Feature specification from `/user-specs/html-viewer/spec.md`

## Summary

The HTML Viewer is a system-level component that renders arbitrary HTML content in a sandboxed window. It provides the `openPreview` tool for agents to dynamically display mockups, diagrams, reports, and other transient HTML content without registering as a full application. The implementation uses a React-based `HtmlViewer` component with iframe sandboxing, VFS file reading with Blob URL generation, and Window Manager integration for system views.

## Technical Context

**Language/Version**: TypeScript 5.x, React 18.x

**Primary Dependencies**: 
- BrowserOS Window Manager API
- VFS (Virtual File System) client
- Agent Tool Registry
- React DOM

**Storage**: VFS (Virtual File System) - reads HTML files from paths like `/tmp/mockup.html`

**Testing**: Jest for unit tests, custom sandbox verification tests

**Target Platform**: BrowserOS web application (WASM/browser environment)

**Performance Goals**: Load content within 200ms for files < 5MB

**Constraints**: 
- Iframe must use strict sandbox attributes (`allow-scripts allow-same-origin`)
- No access to parent window context from rendered content
- Window must not appear in dock or app switcher
- Support files up to 10MB with warning for large files

**Scale/Scope**: Single React component, one tool definition, Window Manager integration hook

## Constitution Check

*GATE: Must pass before implementation. Re-check after design.*

Based on the constitution file review:
- ✅ This feature does not require constitutional changes
- ✅ Security sandboxing requirements align with BrowserOS security model
- ✅ System view type is an acceptable extension to Window Manager
- ✅ No persistence or app registration concerns

## Project Structure

### Documentation (this feature)

```text
user-specs/html-viewer/
├── spec.md           # Original feature specification
├── plan.md           # This file (implementation plan)
└── tasks.md          # Task list for implementation
```

### Source Code (BrowserOS repository root)

```text
# BrowserOS single-project structure
src/
├── components/
│   └── HtmlViewer/
│       ├── HtmlViewer.tsx      # Main React component
│       ├── HtmlViewer.styles.ts # Styled-components or CSS modules
│       └── index.ts            # Exports
├── tools/
│   └── openPreview.ts          # Tool definition and handler
├── windowManager/
│   ├── types.ts                # Add SystemView type
│   ├── hooks.ts                # System view spawn logic
│   └── registry.ts             # Register HtmlViewer component
└── services/
    └── vfsReader.ts            # VFS file reading utilities

tests/
├── unit/
│   ├── components/
│   │   └── HtmlViewer.test.tsx
│   ├── tools/
│   │   └── openPreview.test.ts
│   └── services/
│       └── vfsReader.test.ts
├── integration/
│   └── html-viewer-integration.test.ts
└── security/
    └── sandbox-verification.test.ts
```

**Structure Decision**: Single project structure following BrowserOS conventions. Components in `src/components/`, tools in `src/tools/`, Window Manager extensions in `src/windowManager/`, and utilities in `src/services/`. Tests organized by type (unit, integration, security).

## Complexity Tracking

No violations detected. The implementation follows standard patterns:
- Single React component with clear props/state interface
- Tool registration follows existing Agent Tool Registry patterns
- Window Manager integration adds a new view type without modifying core logic
- Security sandboxing uses standard iframe attributes

## Phase Breakdown

### Phase 1: Component Development
Create the `HtmlViewer` React component with:
- Header bar (title, fullscreen toggle, close button)
- Content area with responsive iframe
- Loading state indicator
- Error handling UI
- VFS file reading and Blob URL generation logic

### Phase 2: Tool Integration
Define and register the `openPreview` tool:
- Input schema validation (filePath OR url required)
- VFS path resolution and file reading
- Blob URL generation for local files
- Direct URL passthrough for external URLs
- Tool handler registration with Agent Runtime

### Phase 3: Window Manager Integration
Add system view support to Window Manager:
- Define `system-view` window type
- Register `HtmlViewer` as a system component
- Implement spawn logic that excludes windows from dock/app switcher
- Connect tool handler to window spawning API

### Phase 4: Testing & Security Verification
Comprehensive testing including:
- Unit tests for component rendering and state management
- Integration tests for end-to-end agent workflow
- Security sandbox verification (script injection, parent access)
- Performance testing for large files
- Edge case handling (invalid paths, CORS errors, malformed HTML)

## Success Criteria Validation

1. **Agent Workflow Test**: Agent creates HTML file → writes to VFS → calls `openPreview` → window appears with content
2. **Security Test**: Rendered content cannot access `window.parent`, no script injection into parent context
3. **Usability Test**: Window closes cleanly, title bar is intuitive, fullscreen toggle works
4. **No App Pollution Test**: Viewer does not appear in "Installed Apps" list or dock

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Sandbox bypass vulnerability | High | Strict iframe attributes, security tests, audit by security team |
| Large file memory issues | Medium | File size validation, streaming for large files, warning UI |
| CORS issues with external URLs | Medium | Clear error messages, fallback to download option |
| Window Manager compatibility | Low | Follow existing patterns, incremental testing |

## Dependencies

- **Window Manager**: Must support custom view types and system view exclusion from dock
- **VFS**: Must provide file reading API with path resolution
- **Agent Tool Registry**: Must allow dynamic tool registration with schema validation

## Next Steps

1. Review and approve this plan
2. Generate tasks.md from this plan
3. Delegate implementation to Developer sub-agent
4. Run tests after implementation
5. Verify security sandboxing before merge
