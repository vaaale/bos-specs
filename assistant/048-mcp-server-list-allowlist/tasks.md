# Tasks: `mcp_server_list` per-agent allowlist enforcement

**Branch**: `bos/self-heal-0056` | **Plan**: `plan.md` | **Spec**: `spec.md`

> Test-driven: the failing test (T2) is written and observed failing **before** the
> implementation (T3/T4). A test written after the fix does not prove the fix.

## Phase 1 — Test first (TDD)

- [ ] **T1 — Author the regression test**
  - File: `tests/assistant/mcp-server-list-allowlist.test.ts` (NEW).
  - `import "../services/_stub-server-only";` and `import { test, expect } from "@playwright/test";` (Playwright unit runner, like `discovery-modes.test.ts`).
  - `vi.mock("@/lib/mcp/store")` so `listMcpServers` resolves to a **fixed** registry of three configs: `playwright`, `gitlab-mcp`, `comfy-mcp` (distinct names + endpoints).
  - `vi.mock("@/lib/agent/subagents/store")` so `getAgent(id)` returns a **fixture agent** whose `mcp` allowlist is configurable per test (the `setInRunAgent` pattern does NOT work — `getAgent` resolves filesystem agents only).
  - Import the **real** `mcpTools()` from `src/lib/assistant/tools/server/mcp.ts`; drive `.mcp_server_list.execute!({}, { agentId: "scoped-agent" } as unknown as ToolContext)`; parse the JSON result; compare the set of names.
  - Cases (map to spec acceptance scenarios + SC-001…SC-004):
    1. **Scoped (name allowlist)** — agent `mcp: ["playwright","gitlab-mcp"]` → result contains exactly `playwright`+`gitlab-mcp`, and **NOT** `comfy-mcp`. *(This is the one that must FAIL on the base.)*
    2. **Unscoped** — agent with no `mcp` field → all three.
    3. **Empty allowlist** — agent `mcp: []` → all three (empty = "all").
    4. **Zero permitted** — agent `mcp: ["other"]` → `[]`.
    5. **Endpoint-based allowlist** — agent `mcp: [comfy-mcp's endpoint]` → `comfy-mcp` present (matches by endpoint, per `isAllowed`/`allowedServers`).
    6. **Set-equality** — for the scoped agent, result === `await allowedServers("scoped-agent")` (import the real gateway fn) — proves single-source-of-truth (US3 / SC-003).
    7. **Missing agentId** — `execute!({}, { } as ToolContext)` → in-band `Error: mcp_server_list: MCP gateway requires an agentId (no active-agent fallback).` string (FR-006); not a throw, not the full registry.
  - Reference the command in a header comment: `npx playwright test -c playwright.unit.config.ts tests/assistant/mcp-server-list-allowlist.test.ts`.

- [ ] **T2 — Observe it fail on the UNMODIFIED base**
  - Run T1's suite against the current (unfiltered) `mcp_server_list`.
  - **Gate**: case 1 MUST fail because `comfy-mcp` is present in the (unfiltered) result; cases 2/3/5/7 should already pass. Confirm the failure is for the *right reason* (the scoped agent sees a server it may not call), not a harness error. Do not proceed until this is the observed state.

## Phase 2 — Implement

- [ ] **T3 — Export the gateway gate**
  - File: `src/lib/mcp/gateway.ts` (MODIFY).
  - Add `export` to `allowedServers(agentId?)` (currently a private `async function` at ~line 43). No behavior change.

- [ ] **T4 — Point `mcp_server_list` at the gate**
  - File: `src/lib/assistant/tools/server/mcp.ts` (MODIFY).
  - Import `allowedServers` from `@/lib/mcp/gateway` (extend the existing gateway import line: `findTools, listServerTools, getToolSchema, callServerTool` → add `allowedServers`).
  - Change the handler from `async () => JSON.stringify(await listMcpServers())` to `async (_input, ctx) => JSON.stringify(await allowedServers(ctx.agentId))` (mirror the `ctx` pattern of `mcp_tool_search`/`mcp_server_tools`).
  - Remove `listMcpServers` from the `@/lib/mcp/store` import **if and only if** it becomes unused (verify: `mcp_server_list` was its only user in this file; `addMcpServer`/`removeMcpServer` stay). Do not leave a dead import.
  - Leave the tool's `name`, `description` ("available to you" is now *true*), and `schema()` (no args) unchanged. Do NOT touch `mcp_server_add`/`mcp_server_remove`.

- [ ] **T5 — Re-run; the suite passes**
  - Run T1's suite against the fix. All 7 cases green. Case 1 now passes (scoped set excludes `comfy-mcp`).

## Phase 3 — Coverage & docs

- [ ] **T6 — Coverage ≥ 60% line + branch on every modified file**
  - Measure coverage on `src/lib/assistant/tools/server/mcp.ts` and `src/lib/mcp/gateway.ts`.
  - If `mcp.ts` is below 60% line coverage because the *sibling* tools (`mcp_server_add`/`mcp_server_remove`) are untested, add minimal tests (happy path + the documented error path) to `mcp-server-list-allowlist.test.ts` (or a sibling test) to lift the file — the gate is per-modified-file, so this is in scope. Do not change sibling behavior.
  - Ensure the new `allowedServers` branch coverage (name match, endpoint match, empty= all, missing-agentId throw) is exercised — cases 1/2/3/5/7 already cover it; add a branch test if any remain uncovered.

- [ ] **T7 — Docs sync (Constitution VI)**
  - `docs/dev/mcp/mcp.md` (UPDATE): the Gateway section currently says the gateway "provides three operations, all enforcing the agent's allowlist (`011`)". Extend it to note the **agent-facing** `mcp_server_list` tool now also enforces the allowlist (returns only the calling agent's permitted servers), closing the gap where it returned the raw global registry. Keep it accurate to the new behavior.
  - `docs/dev/assistant/actions-and-tools.md` (UPDATE, one line, optional): where it describes the agent-facing MCP gateway tool set, note the tool set (including `mcp_server_list`) respects the per-agent allowlist. Skip if it does not currently make a claim this change contradicts (avoid a no-op edit).

- [ ] **T8 — Full verification**
  - Typecheck clean, and the full unit test suite passes (`npx playwright test -c playwright.unit.config.ts`) — no regressions in existing MCP/assistant tests.

## Out of scope (do not touch)

- `mcp_server_add` / `mcp_server_remove` behavior (global registry mutations stay unfiltered).
- The user-facing **Settings → MCP Servers** tab / right-hand MCP status view (the user's view of all configured servers — intentionally unaffected).
- `agent_definition_get`'s field projection.
- `src/tools/supervisor/**` or BOS build config.
