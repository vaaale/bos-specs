# Implementation Plan: `mcp_server_list` per-agent allowlist enforcement

**Branch**: `bos/self-heal-0056` | **Date**: 2026-10-01 | **Spec**: `spec.md`

**Input**: Feature specification from `/Specs/user-specs/assistant/048-mcp-server-list-allowlist/spec.md`

## Summary

Make the agent-facing `mcp_server_list` tool return **only the servers the calling agent is allowed to use**, so it stops advertising servers the MCP gateway will then refuse (the EHS-0056 defect). The fix reuses the gateway's existing allowlist resolver `allowedServers(agentId?)` as the single source of truth — the same gate every other entry point already threads through — rather than re-implementing the `isAllowed` filter. Concretely: export `allowedServers` from `src/lib/mcp/gateway.ts`, point `mcp_server_list`'s handler at it with `ctx.agentId`, and add a regression test that fails on the unfiltered base and passes on the fix.

**Design decision (the skippable `design` step, folded in):** This is a small, single-behavior change to an existing subsystem, so a standalone `design.md` is not warranted. The one architectural choice — *reuse the existing gate vs. copy it* — resolves unambiguously to **reuse**: `gateway.ts`'s own comment warns that "duplicating a GATE is the thing that eventually lets one copy drift into a softer door," and `014` FR-006 names the invariant. Exporting `allowedServers` (a one-line `export` addition) keeps exactly one gate and guarantees `mcp_server_list`'s output is set-equal to what `listServerTools`/`findTools`/`callServerTool` will accept. No new entity, no new external interface, no data-model or contracts artifact.

## Technical Context

**Language/Version**: TypeScript (BOS source, Node/Next.js server)
**Primary Dependencies**: none new — reuses `@/lib/mcp/gateway`, `@/lib/mcp/store`, `@/lib/agent/capabilities` (`isAllowed`), `@/lib/agent/subagents/store` (`getAgent`)
**Storage**: none — `listMcpServers()`/`getAgent()` are the existing read paths; no new persistence
**Testing**: Playwright unit runner (`npx playwright test -c playwright.unit.config.ts`), mirroring `tests/assistant/discovery-modes.test.ts` and `tests/assistant/file-tools-server.test.ts`
**Target Platform**: BOS server process (`"server-only"` modules)
**Project Type**: web service (Next.js) — server-side tool change
**Performance Goals**: no change — `mcp_server_list` still does zero MCP connections; `allowedServers` adds only the already-present `getAgent` read + in-memory filter
**Constraints**: must not change the other four read tools' behavior; must keep `add`/`remove` (registry mutations) unfiltered as-is; `serverTool` guarantees `execute()` never throws (failures become in-band `Error: <tool>: …` strings)
**Scale/Scope**: one agent tool behavior + one exported function + one test file + doc sync

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**No gates evaluable** — the governing constitution is not present in this feature worktree (`.specify/memory/constitution.md` does not resolve at repo root on this branch; the system store is read-only and not mounted here). Per the plan-step honesty rule this is recorded rather than asserted as a pass. The change is a narrow, backward-compatible server-tool fix reusing an existing platform gate, which is consistent with the standing design principle (close the structural gap in the platform; no workaround/hack).

## Project Structure

### Documentation (this feature)

```text
specs/user-specs/assistant/048-mcp-server-list-allowlist/
├── spec.md        # Feature spec (written)
├── plan.md        # This file
└── tasks.md       # Task breakdown (written)
```

No `research.md`, `data-model.md`, or `contracts/` — all NEEDS CLARIFICATION resolved directly from source (see Summary), no new entities, no new external interfaces.

### Source Code (files modified or created)

```text
src/lib/assistant/tools/server/mcp.ts   # MODIFY — mcp_server_list handler takes ctx, resolves via allowedServers(ctx.agentId); drop now-unused listMcpServers import
src/lib/mcp/gateway.ts                  # MODIFY — add `export` to allowedServers(agentId?)
tests/assistant/mcp-server-list-allowlist.test.ts  # NEW — regression test (fails on base, passes on fix)
docs/dev/mcp/mcp.md                     # UPDATE — note the agent-facing mcp_server_list now enforces the allowlist
docs/dev/assistant/actions-and-tools.md # UPDATE (optional, one line) — MCP gateway tool set respects the per-agent allowlist
```

**Structure Decision**: In-place edits to two existing server modules + one new test in `tests/assistant/` (the established home for server-tool tests) + doc sync. No new directories.

## Design (grounded in source)

**The one-line core change.** In `src/lib/assistant/tools/server/mcp.ts`, `mcp_server_list` is currently:

```ts
mcp_server_list: serverTool(
  "mcp_server_list",
  "List the MCP servers available to you, with their descriptions (what each is for).",
  schema(),
  async () => JSON.stringify(await listMcpServers()),
),
```

It becomes (handler signature mirrors `mcp_tool_search`/`mcp_server_tools`, which already take `ctx`):

```ts
mcp_server_list: serverTool(
  "mcp_server_list",
  "List the MCP servers available to you, with their descriptions (what each is for).",
  schema(),
  async (_input, ctx) => JSON.stringify(await allowedServers(ctx.agentId)),
),
```

where `allowedServers` is imported from `@/lib/mcp/gateway` and exported there (it is currently a private `async function` at gateway.ts:43). `allowedServers(agentId)`:
- returns `[]`-equivalent behavior via its `listMcpServers().filter(isAllowed(allow, s.name, s.endpoint))`,
- treats an unset/empty `mcp` allowlist as "all" (inherited from `isAllowed`),
- matches by name OR endpoint (legacy endpoint-based allowlists),
- throws `"MCP gateway requires an agentId (no active-agent fallback)."` when `agentId` is empty — which `serverTool` converts to the in-band `Error: mcp_server_list: MCP gateway requires an agentId (no active-agent fallback).` string (FR-006).

`mcp.ts` still imports `addMcpServer`/`removeMcpServer` (used by `mcp_server_add`/`mcp_server_remove`); the `listMcpServers` import is removed if it becomes unused (verify: `mcp_server_list` was its only user).

**Test strategy (TDD seam).** The unit under test is the *gate* — `allowedServers` + `isAllowed` — applied to the real tool handler. To keep it deterministic and independent of the data dir, **module-mock the two leaf read sources** and let the gate run for real:
- `vi.mock("@/lib/mcp/store")` → `listMcpServers: () => [playwright, gitlab-mcp, comfy-mcp]` (three distinct configs).
- `vi.mock("@/lib/agent/subagents/store")` → `getAgent: (id) => <fixture agent>` where the fixture's `mcp` allowlist varies per test.
- Import the real `mcpTools()` from `src/lib/assistant/tools/server/mcp.ts`; call `.mcp_server_list.execute!({}, { agentId: "scoped-agent" } as ToolContext)`; parse the JSON; assert the set.

This works because `allowedServers` is in `gateway.ts` and calls `listMcpServers` (mocked) + `getAgent` (mocked) + `isAllowed` (real, from `@/lib/agent/capabilities`). **The `discovery-modes.test.ts` `setInRunAgent` pattern does NOT apply here** — `getAgent` resolves filesystem agents via `listDelegatableAgents()` and does not see ephemeral in-run agents, so a filesystem fixture or (as chosen) a module mock is required.

**Regression-proof requirement (SC-004).** The test MUST assert the *filtered* result (e.g. scoped allowlist `["playwright","gitlab-mcp"]` → result excludes `comfy-mcp`). On the unfiltered base (handler calling `listMcpServers()` directly) that assertion fails (`comfy-mcp` present) — so the test proves the defect, not an implementation detail. A second assertion set confirms the unscoped case returns all three (zero regression). A third confirms set-equality with `allowedServers(agentId)`.

## Complexity Tracking

No constitution violations to justify (no gates were evaluable).
