# Feature Specification: HTML Viewer (System Component)

**Feature ID**: `html-viewer`  
**Status**: Draft  
**Priority**: P1 (Critical for Agent Prototyping & Visualization)  
**Dependencies**: Window Manager, VFS (Virtual File System), Agent Tool Registry  

## 1. Overview
The **HTML Viewer** is a system-level component designed to render arbitrary HTML content in a sandboxed window. Unlike standard applications, it is not installed via the app store and does not appear in the dock. Instead, it is invoked dynamically by agents via the `openPreview` tool to display mockups, generated diagrams, reports, or any transient HTML content.

### User Scenarios
1.  **Agent Prototyping**: An agent generates a UI mockup as an HTML file, writes it to `/tmp/mockup.html`, and calls `openPreview` to show the user.
2.  **Data Visualization**: An agent generates an interactive chart (HTML/JS) and opens it for inspection.
3.  **Report Generation**: An agent creates a formatted HTML report and displays it in a clean, distraction-free window.

## 2. Requirements

### Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| **FR-001** | The system MUST provide a `openPreview` tool accessible to agents. | P1 |
| **FR-002** | The `openPreview` tool MUST accept a file path (VFS) or URL as input. | P1 |
| **FR-003** | The tool MUST spawn a new window rendering the content in a sandboxed `<iframe>`. | P1 |
| **FR-004** | The viewer window MUST include a header with a title (configurable) and a Close button. | P1 |
| **FR-005** | The viewer MUST support "Fullscreen" mode to maximize the viewing area. | P2 |
| **FR-006** | The content inside the iframe MUST be strictly sandboxed (no access to parent window, no external network unless allowed). | P1 |
| **FR-007** | The viewer window MUST NOT register as an "App" in the system (no dock icon, no app menu entry). | P1 |
| **FR-008** | If the file path is invalid or the URL fails to load, the viewer MUST display a clear error message within the window. | P2 |

### Non-Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| **NFR-001** | **Security**: The iframe MUST use `sandbox` attributes preventing script injection into the parent context. | P1 |
| **NFR-002** | **Performance**: The viewer must load content within 200ms for files < 5MB. | P2 |
| **NFR-003** | **Responsiveness**: The layout must adapt to window resizing (iframe fills available space). | P1 |

## 3. Architecture & Design

### 3.1 Component Structure
The viewer consists of a single React component (`HtmlViewer`) that acts as a shell for the content.

*   **Props**:
    *   `source`: `string` (VFS path like `/tmp/file.html` or URL).
    *   `title`: `string` (Optional, defaults to "Preview").
    *   `onClose`: `() => void` (Callback to destroy the window).
*   **Internal State**:
    *   `isLoading`: `boolean`
    *   `error`: `string | null`
    *   `contentUrl`: `string | null` (Blob URL for local files, or direct URL).

### 3.2 Security Model (Sandboxing)
The iframe MUST be configured with the following attributes to prevent security risks:
```html
<iframe 
  src="..." 
  sandbox="allow-scripts allow-same-origin" 
  title="Preview Content"
/>
```
*   **`allow-scripts`**: Required for interactive mockups (JS/CSS).
*   **`allow-same-origin`**: Required to load local blobs/paths correctly.
*   **Restrictions**: No `allow-top-navigation`, no `allow-forms` (unless explicitly needed), no access to `window.parent`.

### 3.3 Tool Definition (`openPreview`)
The tool is registered in the Agent Tool Registry.

**Schema**:
```json
{
  "name": "openPreview",
  "description": "Opens a system preview window for HTML content (mockups, charts, reports). Content must be provided as a VFS path or URL.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "filePath": {
        "type": "string",
        "description": "Path to the HTML file in the Virtual File System (e.g., '/tmp/mockup.html'). Required if 'url' is not provided."
      },
      "url": {
        "type": "string",
        "description": "External URL to preview. Required if 'filePath' is not provided."
      },
      "title": {
        "type": "string",
        "description": "Optional title for the window header. Defaults to 'Preview'."
      }
    },
    "required": [] 
  },
  "handler": (args) => {
    // 1. Validate input (filePath OR url must exist)
    // 2. If filePath: Read from VFS, convert to Blob URL.
    // 3. Spawn Window Manager System View with 'HtmlViewer' component.
    // 4. Return success message with window ID.
  }
}
```

### 3.4 Window Manager Integration
The Window Manager must support a new "System View" type:
*   **Type**: `system-view`
*   **Component**: `HtmlViewer`
*   **Behavior**: 
    *   Does not appear in the App Switcher (Alt+Tab) unless configured otherwise.
    *   Can be closed via the title bar or by calling a system API (`closeWindow`).
    *   Does not persist state on window close (transient).

## 4. User Interface Design

### 4.1 Window Layout
The window consists of two main parts:
1.  **Header Bar** (Fixed height, ~40px):
    *   **Left**: Title text (e.g., "Settings Mockup").
    *   **Right**:
        *   [Fullscreen] Icon (Toggle).
        *   [Close] Icon (X) - Closes the window.
2.  **Content Area** (Flex-grow):
    *   Contains the `<iframe>` which fills 100% width/height.
    *   If loading: Shows a spinner.
    *   If error: Shows an error card with "Reload" and "Close" buttons.

### 4.2 Visual Style
*   Matches the current BrowserOS dark/light theme (inherits CSS variables).
*   Clean, minimal borders to maximize content visibility.
*   No scrollbars on the outer window (iframe handles internal scrolling).

## 5. Edge Cases & Error Handling

| Scenario | Behavior |
|----------|----------|
| **File Not Found** | Display error message: "File not found: [path]". Provide "Close" button. |
| **Invalid HTML** | Render the raw HTML (browser will attempt to parse). If it crashes, show a generic "Content Error". |
| **External URL Blocked** | If CORS or security policies block the URL, display an error: "Failed to load external resource." |
| **Large File (>10MB)** | Show a warning before rendering: "File is large. Rendering may be slow." |
| **Script Injection Attempt** | The sandbox prevents scripts from accessing the parent context. Logs a security warning in console if detected. |

## 6. Success Criteria

1.  **Agent Workflow**: An agent can successfully create an HTML file, call `openPreview`, and see the content in a new window.
2.  **Security**: The rendered content cannot access BrowserOS internals (e.g., `window.parent` is null or restricted).
3.  **Usability**: The window closes cleanly, and the title bar is intuitive.
4.  **No App Pollution**: The viewer does not appear in the "Installed Apps" list or dock.

## 7. Implementation Plan (High Level)

1.  **Phase 1: Core Component**
    *   Create `HtmlViewer` React component with iframe and header.
    *   Implement VFS file reading and Blob URL generation.
2.  **Phase 2: Tool Integration**
    *   Define `openPreview` tool schema.
    *   Register tool with Agent Runtime.
3.  **Phase 3: Window Manager Hook**
    *   Add "System View" spawn logic to Window Manager.
    *   Connect tool handler to window spawn.
4.  **Phase 4: Testing & Security**
    *   Test with malicious scripts (verify sandbox).
    *   Test with large files and external URLs.
