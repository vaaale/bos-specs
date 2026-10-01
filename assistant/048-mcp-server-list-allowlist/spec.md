# Feature Specification: `mcp_server_list` per-agent allowlist enforcement

**Feature Branch**: `bos/self-heal-0056`

**Created**: 2026-10-01

**Status**: Draft

**App Target**: bos-core

**Input**: Self-heal case EHS-0056. The agent-facing `mcp_server_list` tool returns the *entire* global MCP registry (via `listMcpServers()`) and describes its result as the servers "available to you," while every other agent-facing MCP entry point (`mcp_tool_search`, `mcp_server_tools`, `mcp_tool_schema`, `mcp_tool_call`) threads `ctx.agentId` into the allowlist-enforcing gateway. For any agent carrying a non-empty `mcp` allowlist (`011-per-agent-capabilities`), `mcp_server_list` therefore advertises servers the gateway will then refuse. The gateway's own header states the invariant being violated: *"All entry points enforce the active agent's MCP server allowlist (011)"* (`src/lib/mcp/gateway.ts`). This feature closes the single entry point that breaks it.

## Context & Root Cause

`mcp_server_list` is defined at `src/lib/assistant/tools/server/mcp.ts:57-61` as `async () => JSON.stringify(await listMcpServers())` — the handler takes no `ctx`, so it cannot reach the caller's agent, and the result is the raw global registry from `src/lib/mcp/store.ts`.

The allowlist already exists and is the correct single source of truth: `allowedServers(agentId?)` at `src/lib/mcp/gateway.ts:43-58` reads `listMcpServers()` and filters by the agent's `mcp` allowlist via `isAllowed(allow, s.name, s.endpoint)`, treating an unset/empty allowlist as "all." Every other gateway entry point (`listServerTools`, `findTools`, `getToolSchema`, `callServerToolRaw`, `readServerResource`) resolves its target through this function.

The gateway comment warns explicitly against copying this gate ("duplicating a GATE is the thing that eventually lets one copy drift into a softer door"), so the fix reuses `allowedServers` rather than re-implementing the `isAllowed` filter in `mcp.ts`.

## Clarifications

### Session 2026-10-01 (self-heal pre-authorized)

- Q: Reuse the gateway's `allowedServers(agentId)`, or re-implement `isAllowed` inline in `mcp.ts`? → A: **Reuse.** Export `allowedServers` from `gateway.ts` and call it with `ctx.agentId`. This keeps one gate, matching the gateway's own design comment, and guarantees `mcp_server_list`'s output is exactly the set the rest of the gateway will accept.
- Q: Scope — fix `mcp_server_list` only, or also `mcp_server_add` / `mcp_server_remove` (the registry mutations)? → A: **`mcp_server_list` only.** The `add`/`remove` tools are global registry *mutations* (they change which servers exist), not per-agent *reads*; their results legitimately describe the deployment's registry, not the caller's view. The defect is specifically that a *read* presented itself as the caller's personal list. This matches `014` FR-006, which governs the *discovery/read* gateway tools. (No description/behavior of `add`/`remove` changes.)
- Q: What does the tool return for a scoped agent that is allowed *zero* servers? → A: An empty list (JSON `[]`) — the correct, truthful answer, consistent with an allowlist filter that matches nothing.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A scoped agent sees only the servers it may actually use (Priority: P1)

An agent restricted by an `mcp` allowlist (per `011`) calls `mcp_server_list` to orient itself before searching tools. Today the tool returns every server registered on the deployment, including ones the gateway will refuse; the agent reasonably attempts one and hits `No MCP server "X" available to this agent.` — the exact failure observed in EHS-0056. After the fix, the list contains only servers the gateway will accept, so the advertised set and the callable set agree.

**Why this priority**: This is the whole defect. Without it, any scoped agent receives a false signal that misdirects it and produces a spurious gateway refusal.

**Independent Test**: Configure an agent with a non-empty `mcp` allowlist that excludes at least one registered server. Invoke `mcp_server_list` with that agent's `ctx` and assert the returned set equals the agent's allowlist (and is a strict subset of the global registry). This delivers value on its own.

**Acceptance Scenarios**:

1. **Given** a deployment has servers `playwright`, `gitlab-mcp`, and `comfy-mcp` registered, and an agent `assistant` whose `mcp` allowlist is `["playwright", "gitlab-mcp"]`, **When** `mcp_server_list` runs with `ctx.agentId = "assistant"`, **Then** the result contains `playwright` and `gitlab-mcp` and does NOT contain `comfy-mcp`.
2. **Given** the same agent as above, **When** it calls `mcp_tool_search(server: "comfy-mcp")`, **Then** it still receives the gateway's `No MCP server "comfy-mcp" available to this agent.` — and `mcp_server_list` no longer pointed it there, so the mismatch is gone.

---

### User Story 2 - An unscoped agent sees all configured servers (Priority: P1)

An agent with no `mcp` allowlist set (or an empty one) should behave exactly as before: `mcp_server_list` returns every registered server. The allowlist is opt-in scoping, so the default must be lossless.

**Why this priority**: Regressing the common (unscoped) case would be worse than the bug being fixed. This is the "default = all" guarantee already implemented by `allowedServers`.

**Independent Test**: Invoke `mcp_server_list` with an agent that has no `mcp` field set; assert the result equals the full `listMcpServers()` output.

**Acceptance Scenarios**:

1. **Given** servers `playwright`, `gitlab-mcp`, `comfy-mcp` registered and an agent with no `mcp` allowlist, **When** `mcp_server_list` runs with that agent's `ctx`, **Then** all three appear in the result.
2. **Given** an agent whose `mcp` allowlist is explicitly `[]`, **When** `mcp_server_list` runs, **Then** all registered servers appear (empty allowlist = "all," per `isAllowed`/`011`).

---

### User Story 3 - The gateway's allowlist gate stays single-sourced (Priority: P2)

`mcp_server_list` derives its filtered set from the *same* `allowedServers(agentId)` the rest of the gateway uses, rather than a second copy of the filter. If the allowlist semantics change (e.g. endpoint matching), every entry point — including `mcp_server_list` — changes together, so the "all entry points enforce the allowlist" invariant cannot be silently broken again by adding a new read path.

**Why this priority**: This is the structural guarantee that prevents the *class* of bug (an entry point drifting out of sync with the gate), not just this one instance. It is architecturally load-bearing; without it the fix is a patch that can re-drift.

**Independent Test**: Assert `mcp_server_list`'s result for a given `agentId` is set-equal to `allowedServers(agentId)`. Assert `mcp.ts` imports/reuses `allowedServers` rather than calling `listMcpServers()` + an inline `isAllowed` filter.

**Acceptance Scenarios**:

1. **Given** `allowedServers("assistant")` returns `{playwright, gitlab-mcp}`, **When** `mcp_server_list` runs with `ctx.agentId = "assistant"`, **Then** the two are set-equal.
2. **Given** a future change to how the allowlist matches servers, **When** `mcp_server_list` runs, **Then** it reflects the change automatically because it shares `allowedServers`.

---

### Edge Cases

- **Scoped agent allowed zero servers**: `mcp_server_list` returns an empty list (`[]`), not an error and not the full registry.
- **Missing `ctx.agentId`**: The gateway's `allowedServers` throws `"MCP gateway requires an agentId (no active-agent fallback)."` when no id is supplied. `mcp_server_list` MUST surface this consistently with its sibling gateway tools (an in-band `Error: …` string via `serverTool`), not crash or silently fall back to the global registry.
- **Servers matching by endpoint, not name**: `allowedServers` matches an allowlist entry by name OR endpoint (legacy endpoint-based allowlists). `mcp_server_list` MUST inherit this by reusing `allowedServers`, so endpoint-scoped agents also see the correct set.
- **`add`/`remove` unchanged**: Registry mutations still report the full deployment registry (their legitimate behavior). This feature does not alter them.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The `mcp_server_list` tool handler MUST receive the tool context (`ctx`) and resolve the server list through the gateway's `allowedServers(ctx.agentId)` rather than the unfiltered `listMcpServers()`.
- **FR-002**: The `mcp_server_list` result for an agent with a non-empty `mcp` allowlist MUST contain exactly the servers permitted by that allowlist (matched by name or endpoint per `isAllowed`), and MUST NOT contain any other registered server.
- **FR-003**: The `mcp_server_list` result for an agent with no `mcp` allowlist set (or an empty one) MUST contain every registered server — behaviorally identical to today's unscoped output.
- **FR-004**: The gateway's `allowedServers(agentId?)` MUST be exported from `src/lib/mcp/gateway.ts` so it is the single shared resolution path; `mcp_server_list` MUST reuse it and MUST NOT introduce a second inline `isAllowed` filter.
- **FR-005**: `mcp_server_list` MUST remain consistent with the other agent-facing MCP read tools: its advertised server set MUST equal the set those tools will accept for the same `agentId`.
- **FR-006**: When `ctx.agentId` is missing, `mcp_server_list` MUST surface the gateway's no-active-agent error as an in-band `Error: …` string (via `serverTool`), not a thrown exception or an unfiltered fallback.

### Non-Functional Requirements

- **NFR-001**: `mcp_server_list` MUST NOT introduce a network connection to any MCP server — it returns *configured* (and allowed) server descriptors, not their live tool lists, exactly as `listMcpServers()` does today. (No behavior change to cost/latency.)
- **NFR-002**: The fix MUST keep the agent-facing MCP tool set constant in shape; it MUST NOT add tools or change the other four read tools' signatures.

### Key Entities

- **MCP server descriptor** — the shape `listMcpServers()` returns (name, description, transport, …); `mcp_server_list` now returns the *allowlist-filtered* subset of these.
- **`allowedServers(agentId)`** — the gateway's single allowlist resolution function; becomes exported and reused.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For an agent whose allowlist excludes at least one registered server, `mcp_server_list` returns a set equal to the allowlist and a strict subset of the global registry (the EHS-0056 symptom — advertising an uncallable server — is eliminated).
- **SC-002**: For an unscoped agent, `mcp_server_list` output is byte-equivalent (as a set) to today's `listMcpServers()` output — zero regression for the default case.
- **SC-003**: `mcp_server_list`'s result is set-equal to `allowedServers(ctx.agentId)` for every agent configuration (scoped, unscoped, endpoint-matched, empty allowlist, zero-permitted).
- **SC-004**: A regression test fails on the pre-fix base (unfiltered `listMcpServers()`) and passes on the fix — proving the test targets the actual defect, not an implementation detail.

## Assumptions & Dependencies

- Builds on `011-per-agent-capabilities` (the `Agent.mcp` allowlist + `isAllowed`) and `014-mcp-tool-gateway` (the gateway and its allowlist invariant). This feature *closes the `mcp_server_list` gap* against `014` FR-006 ("All gateway tools MUST enforce the active agent's MCP server allowlist"); it does not re-spec that requirement.
- `serverTool` already threads `ctx` to handlers (as `mcp_tool_search` and siblings do), so receiving `ctx` in `mcp_server_list` is a handler-signature change only — no framework change.
- The user-facing **MCP tab** (Settings) and its live-status view are a *different* surface (the user's view of all configured servers) and are intentionally unaffected; this feature scopes only the *agent-facing* `mcp_server_list` tool.
- Out of scope: changing `mcp_server_add`/`mcp_server_remove` (global mutations); adding per-agent allowlist UI; altering the `agent_definition_get` projection.
