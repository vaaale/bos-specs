# Feature Specification: Agent Settings Redesign

**Feature Branch**: `agent-settings-redesign`
**Created**: 2026-07-02
**Status**: Draft
**Input**: "The current 'Assistant' page in Settings is disorganized and lacks clear hierarchy. Users need a Master-Detail layout to manage agent personalities, scope their capabilities (Skills, MCP, Tools), and edit instructions with auto-save."

> **Context**: The current Settings → Assistant tab lists agents but lacks a dedicated configuration view for capabilities. Agents cannot be scoped to specific skills or tools via the UI. This feature introduces a comprehensive Master-Detail interface that aligns with the BOS UI Style Guide (opacity colors, dense typography, dark theme) and enables fine-grained capability scoping per agent.

## User Scenarios & Testing (Mandatory)

### User Story 1 - Manage Agent Personalities (Priority: P1)
A user wants to create a new agent persona, edit an existing one, or delete one they no longer need.
**Acceptance Scenarios**:
1.  **Given** the Settings → Assistant page, **When** the user clicks "+ New Agent", **Then** a dialog opens to input Name, Description, and initial System Prompt.
2.  **Given** an agent list, **When** the user selects an agent, **Then** the right panel displays its full configuration (Name, Description, Instructions, Capabilities).
3.  **Given** an agent is selected, **When** the user edits any field and clicks away (blur), **Then** changes are saved automatically without a manual "Save" button.
4.  **Given** an agent is selected, **When** the user clicks "Delete Agent" in the Danger Zone, **Then** a confirmation dialog appears before permanent deletion.

### User Story 2 - Scope Agent Capabilities (Priority: P1)
A user wants to restrict an agent to only use specific skills, MCP servers, or tools.
**Acceptance Scenarios**:
1.  **Given** an agent's capabilities section, **When** the user unchecks a Skill, **Then** that skill is removed from the agent's allowed list and is no longer available to it.
2.  **Given** an agent's capabilities section, **When** the user unchecks an MCP Server, **Then** all tools from that server are hidden from the agent.
3.  **Given** an agent's capabilities section, **When** the user unchecks a specific Tool (e.g., `delete_path`), **Then** only that tool is restricted while others remain accessible.
4.  **Given** a capability list is empty (no allowlist set), **When** the agent runs, **Then** it has full access to all available items (back-compatible default).

### User Story 3 - Visual Consistency & UX (Priority: P1)
The new interface must feel like a native BrowserOS app.
**Acceptance Scenarios**:
1.  **Given** the Settings page, **When** viewed, **Then** it uses the BOS dark theme with opacity-based colors (`rgba(255,255,255,0.1)`), `text-xs` default typography, and tight spacing (`gap-1`).
2.  **Given** a capability item, **When** hovered, **Then** it shows a subtle background change (`hover:bg-white/10`) and border highlight.
3.  **Given** a tool with a dangerous action (e.g., `delete_path`), **When** viewed, **Then** its description is highlighted in red with a warning icon (⚠️).
4.  **Given** a save action occurs, **When** completed, **Then** a "Saved" indicator appears briefly in the header and fades out.

## Requirements (Mandatory)

### Functional Requirements

- **FR-001**: The Settings → Assistant tab MUST render a **Master-Detail layout**:
  - **Left Panel**: Scrollable list of agents with Name, Description, and Active status indicator.
  - **Right Panel**: Detailed configuration form for the selected agent.
- **FR-002**: The Agent data model MUST be extended to include a `capabilities` object:
  ```typescript
  interface AgentCapabilityScope {
    skills?: string[];      // Allowlist of skill IDs (empty = all)
    mcpServers?: string[];  // Allowlist of server names (empty = all)
    tools?: string[];       // Allowlist of tool names (empty = all)
  }
  ```
- **FR-003**: The UI MUST support **Auto-Save**:
  - Changes to text fields MUST save on `blur`.
  - Changes to checkboxes MUST save on `change`.
  - A "Saving..." / "Saved" status indicator MUST be visible in the header.
- **FR-004**: The Capabilities section MUST be organized into three distinct groups:
  1.  **Skills Access**: Grid of cards with checkboxes and descriptions.
  2.  **MCP Servers**: Grid of cards with checkboxes and descriptions.
  3.  **Tool Access**: Categorized accordions (OS, FILES, WEB) with "Toggle All" buttons per category.
- **FR-005**: Tool descriptions MUST be visible under each tool name, with special styling for dangerous tools (red text + warning icon).
- **FR-006**: The UI MUST adhere to the **BOS UI Style Guide** (`docs/dev/guides/style-guide.md`):
  - Dark theme only (no light mode variants).
  - Colors using opacity scale (e.g., `bg-white/5`, `text-white/60`).
  - Typography: `text-xs` default, `font-mono` for tool names.
  - Layout: `flex h-full flex-col` with `min-h-0 flex-1` for scroll regions.
- **FR-007**: The "Delete Agent" action MUST be isolated in a "Danger Zone" section at the bottom, requiring explicit confirmation.
- **FR-008**: Existing agents without a `capabilities` field MUST default to full access (no breaking changes).

### Key Entities

- **Agent Personality**: A named configuration with instructions and scoped capabilities.
- **Capability Scope**: The allowlist of Skills, MCP Servers, and Tools accessible to an agent.
- **Master-Detail Layout**: The two-pane UI pattern used for management.
- **Danger Zone**: The isolated section for destructive actions (Delete).

## Success Criteria (Mandatory)

### Measurable Outcomes

- **SC-001**: Users can create, edit, and delete agents within 3 clicks of opening Settings.
- **SC-002**: Capability scoping is visually distinct and easy to configure (no more than 2 seconds to toggle a skill).
- **SC-003**: Auto-save works reliably with no visible lag or data loss.
- **SC-004**: The UI passes visual regression tests against the included mockup: `user-specs/agent-settings-redesign/mockup.html`.
- **SC-005**: Scoped agents correctly hide restricted tools/skills in the chat interface and instruction composition.

### Visual Reference
The implementation must match the design in `mockup.html` exactly, including:
-   Master-Detail layout with a 260px sidebar.
-   Auto-save indicator in the header.
-   Capability grids with descriptions.
-   Categorized tool accordions with "Toggle All".
-   Danger Zone styling for delete actions.

## Implementation Details

### Data Structure
Agents are stored as markdown files in `data/agents/<id>/AGENT.md`. The frontmatter must include:
```yaml
name: "Assistant"
description: "The default agent..."
type: "local"
systemPrompt: |
  You are the default agent...
capabilities:
  skills: ["memory", "file-organizer"]
  mcpServers: ["github"]
  tools: [] # Empty means all
```

### Component Architecture
1.  **`AgentList`**: Fetches `listSubAgents()`, renders list with selection state.
2.  **`AgentDetails`**:
    -   **Header**: Name/Description inputs (debounced save).
    -   **Instructions**: Large textarea for `systemPrompt`.
    -   **CapabilitiesSection**:
        -   Fetches `listSkills()`, `listMcpServers()`, `listTools()` (grouped by category).
        -   Renders grids and accordions.
        -   Handles checkbox state changes and persists to agent file.
3.  **`AutoSaveStatus`**: Small status indicator in the header.

### Tool Grouping Logic
Tools must be grouped by a `category` metadata field (e.g., "OS", "FILES", "WEB"). If missing, group under "General". The UI MUST expose this grouping.

## Edge Cases & Constraints

- **Large Capability Lists**: If >50 items exist in a category, the grid should handle scrolling gracefully (no pagination needed for MVP).
- **Missing Descriptions**: If a skill/tool lacks a description, display a generic placeholder ("No description available").
- **Concurrent Edits**: If two users edit the same agent simultaneously, the last write wins (standard VFS behavior).
- **Hydration Mismatches**: Ensure data fetching happens in `useEffect` to avoid SSR/client mismatches.

## Dependencies

-   **Data Layer**: `listSubAgents`, `saveAgent` (or equivalent file write logic).
-   **Tool Registry**: Ability to list all available tools grouped by category.
-   **Skill Registry**: Ability to list all available skills with descriptions.
-   **MCP Registry**: Ability to list all connected MCP servers with descriptions.
-   **UI System**: BOS Style Guide compliance (Tailwind opacity utilities, Geist fonts).

## Future Enhancements (Out of Scope for MVP)

-   **Search/Filter**: Add a search bar in the Agent List to filter by name or description.
-   **Bulk Actions**: Select multiple agents to apply capability changes at once.
-   **Import/Export**: Ability to export agent configs as JSON/YAML and import them.
-   **Version History**: Track changes to agent instructions over time.