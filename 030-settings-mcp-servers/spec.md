# Feature Specification: Settings — MCP Servers (configuration)

**Feature Branch**: `030-settings-mcp-servers`

**Created**: 2026-07-27 (extracted from `000-browseros-core` FR-017 for topic clarity)

**Status**: Implemented (configuration); the `includeInDevHarness` checkbox is new — see Requirements

**Input**: "Extract any and all information pertaining to configuring MCP servers into a dedicated spec. Add a checkbox so a configured MCP server can be included in the Dev Harness, with the mechanics described in `029-settings-dev-harness` and only a reference here."

> This spec owns the **MCP server entity and its configuration surface** (Settings → MCP Servers): what a server is, how it's added/edited/tested/removed, and where it's persisted. It does **not** own how the agent discovers/calls a server's tools at chat time — that's `014-mcp-tool-gateway`, which consumes the `description` field this spec defines. It also does **not** own what happens when a server is flagged for the Dev Harness — that's `029-settings-dev-harness`, which this spec only points to.

## Clarifications

### Session 2026-07-27

- Q: Where did this content live before? → A: `000-browseros-core` FR-017 specified the MCP config UI directly; that FR is now replaced there with a pointer to this spec. `014-mcp-tool-gateway`'s FR-002 separately added the `description` field and its agent-facing consequences — the field's existence/persistence is this spec's concern, while what the gateway does with it stays in `014`.
- Q: Should the "Include in Dev Harness" checkbox's mechanics (what gets generated, which files, when) live here or in the Dev Harness spec? → A: **In the Dev Harness spec (`029-settings-dev-harness`).** This spec only defines the flag on the entity and the checkbox in the editor UI; `029` owns everything about what reading that flag produces.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Connect an MCP server (Priority: P1)

A user connects a Model Context Protocol server — remote (Streamable HTTP/SSE) or local (stdio) — so the assistant can use its tools.

**Why this priority**: Without this, the assistant has no access to any external MCP tool.

**Independent Test**: In Settings → MCP Servers, add a server of each transport, Test each, and confirm each reports its tool list.

**Acceptance Scenarios**:

1. **Given** the MCP Servers settings tab, **When** the user adds a server with transport **streamable HTTP** or **SSE**, providing a name, server URL, and optionally a bearer token and/or custom headers (e.g. `Private-Token`), **Then** it is saved keyed by its unique name.
2. **Given** the MCP Servers settings tab, **When** the user adds a server with transport **stdio**, providing a command, arguments, environment variables, and optionally a working directory (e.g. a local `docker run … ghcr.io/github/github-mcp-server`), **Then** it is saved keyed by its unique name.
3. **Given** a server being added or edited, **When** the user clicks **Test**, **Then** BOS connects and lists the server's tools (or reports the error) **without saving** the server.
4. **Given** a configured server, **When** the user clicks **Test** from the server list, **Then** BOS probes the already-saved config and reports connectivity + tool count.
5. **Given** a configured server, **When** the user clicks **Remove**, **Then** it is deleted and no longer available to the assistant.
6. **Given** JSON in the common `{ "mcpServers": { "<name>": { command/url/... } } }` or bare `{ "<name>": {...} }` shape (e.g. copied from another tool's config), **When** the user pastes it into "Import from JSON", **Then** each entry is normalized into a server config and saved.

### User Story 2 - Steer the agent with a description (Priority: P2)

A user gives a server a short description so the assistant's gateway (`014-mcp-tool-gateway`) knows what it's for.

**Acceptance Scenarios**:

1. **Given** the server editor, **When** the user sets a **Description**, **Then** it is persisted with the server config and available to any consumer that reads server configs (notably `014`'s agent-context index).
2. **Given** no description is set, **When** the server is read by a consumer, **Then** a sensible default is derivable from name/transport (the default derivation itself is `014`'s concern, not this spec's).

### User Story 3 - Manage servers as an agent, not only from Settings (Priority: P2)

The assistant itself can add, list, and remove MCP servers on the user's behalf (e.g. "connect the GitHub MCP server"), mirroring what Settings offers.

**Acceptance Scenarios**:

1. **Given** a chat request naming a server to connect (endpoint/command + args), **When** the agent calls its add-server tool, **Then** the server is saved exactly as if added via Settings (same validation, same store).
2. **Given** an existing server, **When** the agent calls its remove-server tool by name, **Then** it is deleted.
3. **Given** any configured servers, **When** the agent calls its list-servers tool, **Then** it gets the same data Settings shows (name, transport, description, etc. — not full credentials).

### User Story 4 - Include a server in the Dev Harness (Priority: P2)

A user who configured a server here wants it also available to the headless Dev Harness CLIs, without configuring it twice.

**Acceptance Scenarios**:

1. **Given** the server editor, **When** the user checks **"Include in Dev Harness"** and saves, **Then** the flag is persisted with the server config. What happens next (generation of harness-native config files) is specified in `029-settings-dev-harness`; this spec's contract ends at "the flag is saved and readable."
2. **Given** the flag is left unchecked (default), **When** the server is saved, **Then** its behavior is unchanged — visible to the gateway only, not to the harness.

### Edge Cases

- Renaming a server (a new `name` on an existing entry): treated as delete-old + add-new, since servers are keyed by name.
- An http/sse server missing an endpoint URL, or a stdio server missing a command (and no endpoint to derive one from), MUST be rejected with a clear validation error before saving.
- Import of a JSON blob with zero recognizable server entries MUST fail with a clear error, not silently save nothing.
- Deleting a server that the active agent's MCP allowlist (`011-per-agent-capabilities`) references by name simply removes it from that allowlist's effective set (no dangling-reference error) — enforcement lives in `011`/`014`, not here.
- Bearer token / stdio environment variables are user-supplied secrets shown in the editor while being typed (unlike Dev Harness credentials, which are write-only) — this spec's UI does not mask them after save; treat this as a lower-sensitivity secret class than harness credentials, consistent with existing behavior.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST let users configure MCP servers from Settings (a dedicated "MCP Servers" tab), each keyed by a unique **name**.
- **FR-002**: BOS MUST support three transports: **streamable HTTP** (default), **SSE**, and **stdio**.
- **FR-003**: For **http/sse**, BOS MUST accept a server **URL**, an optional bearer **token** (sent as `Authorization: Bearer …`), and arbitrary custom **headers** (merged with, and overriding, the bearer header).
- **FR-004**: For **stdio**, BOS MUST accept a **command**, **arguments**, **environment variables**, and an optional **working directory**.
- **FR-005**: Each server MUST have an optional, user-editable **description**, persisted with the server config (consumed by `014-mcp-tool-gateway` for its agent-context index; this spec is responsible only for the field's existence, storage, and editability).
- **FR-006**: Each server MUST be **testable from the UI** two ways: (a) from the editor, before saving (`{ test: true, ... }`, no persistence); (b) from the saved list, probing the stored config. Both report success + tool list, or a clear error.
- **FR-007**: The same operations (add, remove, list, and connectivity probe) MUST also be available to the agent as tools, mirroring the Settings UI's validation and persistence exactly (same store, same normalization) — no separate code path or separate rules for agent-driven vs. UI-driven configuration.
- **FR-008**: BOS MUST support importing servers from a pasted JSON blob shaped `{ "mcpServers": { "<name>": {...} } }` or a bare `{ "<name>": {...} }` map, normalizing each entry (`command` present ⇒ stdio; `type: "sse"` ⇒ sse; otherwise http) into a server config and saving it.
- **FR-009**: Configured servers MUST persist across restarts (a JSON file under `data/`), with an environment-variable fallback (a comma-separated list of endpoints) usable to seed an initial list before any file exists.
- **FR-010**: Each server MAY be flagged **`includeInDevHarness`** (boolean, default/absent `false`), settable via a checkbbox in the server editor next to Description. This spec owns only the flag's existence, persistence, and UI control — **`029-settings-dev-harness`** owns everything about what reading the flag produces.

### Key Entities

- **MCP server** (`McpServerConfig`) — `name` (unique key), `description?`, `transport?` (`http`/`sse`/`stdio`, default http), `endpoint?`, `apiKey?`, `headers?`, `command?`, `args?`, `cwd?`, `env?`, `includeInDevHarness?` (boolean, default false — see `029`).
- **Server store** — the persisted list of configured servers, keyed by name; upsert-by-name semantics (re-adding a name replaces it).
- **Probe result** — `{ ok, tools?, error? }`, the outcome of testing a server (saved or not-yet-saved).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can add an MCP server of each transport (streamable HTTP, SSE, stdio — including one with custom headers and one local stdio command with env) in Settings, Test it to see its tool list, and the agent can then use its tools.
- **SC-002**: Testing an unsaved, in-progress edit reports connectivity without altering the saved server list.
- **SC-003**: Importing a valid JSON blob with N server entries results in N servers appearing in the list, each individually editable afterward.
- **SC-004**: The agent can add/list/remove a server via its own tools with identical results to doing so from Settings.
- **SC-005**: Checking "Include in Dev Harness" and saving persists the flag; unchecking and saving clears it. (What consuming it produces is verified under `029`'s success criteria, not here.)

## Assumptions & Dependencies

- Extracted from `000-browseros-core` FR-017, which is now a pointer to this spec (constitution VI: specs stay in sync, no duplicated canonical content).
- Feeds `014-mcp-tool-gateway`: the `description` field and the server list itself are inputs to the gateway's per-agent allowlist and context index; this spec does not specify gateway behavior.
- Feeds `029-settings-dev-harness`: the `includeInDevHarness` flag is this spec's field, but `029` is solely responsible for reading it and generating anything from it.
- Depends on `011-per-agent-capabilities` for per-agent server allowlisting (a server configured here may or may not be visible to a given agent; that scoping logic lives in `011`).
- Secrets handling: bearer tokens / stdio env vars are treated as ordinary configuration values in this spec (visible while editing), a lower sensitivity class than Dev Harness credentials (which are write-only) — this is existing, accepted behavior, not a gap to close here.

## Notes

- A **larger refactoring pass** across the spec store (renumbering, resolving `026`/`027` duplicate numbers, filling in `overview.md`'s missing 019–028 rows, etc.) is acknowledged as needed but explicitly out of scope for this extraction.
