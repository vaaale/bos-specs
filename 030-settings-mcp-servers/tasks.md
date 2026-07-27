# Tasks: Settings — MCP Servers (configuration)

Feature branch: `030-settings-mcp-servers`. US1–US3 were already implemented under `000-browseros-core` FR-017 before this spec existed; recorded here as done for traceability now that the topic has a dedicated home. US4 (`includeInDevHarness`) is the only open work.

## Phase 1: Foundational

- [x] T001. `McpServerConfig` type (`src/lib/mcp/types.ts`): `name`, `description?`, `transport?` (`http`/`sse`/`stdio`), `endpoint?`, `apiKey?`, `headers?`, `command?`, `args?`, `cwd?`, `env?`.
- [x] T002. `data/mcp-servers.json` store (`src/lib/mcp/store.ts`): `listMcpServers`/`addMcpServer` (upsert-by-name)/`removeMcpServer`, with `BOS_MCP_SERVERS` env-var fallback before the file exists.
- [x] T003. `connectMcpClient`/`probeMcpServer`/`extractText` (`src/lib/mcp/client.ts`): stdio/http/sse transports, bearer + custom header merging, 8s (http/sse) / 30s (stdio) timeouts.

## Phase 2: User Story 1 - Connect an MCP server (P1)

- [x] T010. [US1] `src/app/api/mcp/route.ts`: `GET` (list, or probe by `?probe=name`), `POST` (`{ test: true, ... }` probes without saving; otherwise upserts), `DELETE` (`?name=` or legacy `?endpoint=`).
- [x] T011. [US1] `McpServersTab.tsx`: transport picker (http/sse/stdio), http/sse fields (endpoint, bearer token, custom headers via `KeyValueRows`), stdio fields (command, newline-separated args, env via `KeyValueRows`, optional cwd), per-server Test + list view with connection status.
- [x] T012. [US1] `McpServersTab.tsx`: "Import from JSON" — accepts `{ mcpServers: {...} }` or a bare name-keyed map, normalizes each entry, saves each.

**Checkpoint**: US1 functional — a server of each transport can be added, tested, and used.

## Phase 3: User Story 2 - Steer the agent with a description (P2)

- [x] T020. [US2] `description?: string` field in `McpServerConfig`, editable in `McpServersTab.tsx`'s editor, persisted with the server config. (Consumption by `014-mcp-tool-gateway` is that spec's task, not repeated here.)

## Phase 4: User Story 3 - Manage servers as an agent (P2)

- [x] T030. [US3] `src/lib/assistant/tools/server/mcp.ts`: `mcp_server_add`/`mcp_server_remove`/`mcp_server_list` tools, sharing the same `normalizeConfig` validation and `store.ts` persistence as the Settings UI/API route.

**Checkpoint**: US1–US3 functional — this is the state the topic was in before extraction from `000-browseros-core`.

## Phase 5: User Story 4 - Include a server in the Dev Harness (P2) — OPEN

**Goal**: An MCP server can be flagged so `029-settings-dev-harness`'s config generator includes it.

**Independent Test**: Check "Include in Dev Harness" on a server, save, confirm `listMcpServers()` returns `includeInDevHarness: true`; uncheck, save, confirm it's gone. (Generation into harness files is verified under `029`'s tasks, not here.)

- [x] T040. [US4] `src/lib/mcp/types.ts`: add `includeInDevHarness?: boolean` to `McpServerConfig`.
- [x] T041. [US4] `src/components/apps/settings/McpServersTab.tsx`: "Include in Dev Harness" checkbox in the editor (next to Description) + a "harness" badge on included servers in the list view, wired through the existing `buildConfig()`/save path.
- [x] T042. [US4] `src/app/api/mcp/route.ts`'s `normalizeConfig` AND `src/lib/assistant/tools/server/mcp.ts`'s `normalizeConfig`: both accept/persist `includeInDevHarness`.
- [x] T043. [US4] Typecheck + lint green (verified `npx tsc --noEmit` and `npx eslint` clean); manually verified end-to-end via the live API (add/flag/unflag a server, confirmed both generated harness files update accordingly — see `029`'s T-verification).

**Checkpoint**: US4 functional — the flag round-trips through Settings, the API, and the agent tool.

## Phase 6: Closeout — `000-browseros-core` sync

- [x] Z1. Replaced `000-browseros-core/spec.md` FR-017 with a one-line pointer to this spec.
- [x] Z2. Added a `030` row to `overview.md`'s feature map.
- [ ] Z3. `docs/dev/**` / `docs/usage/**`: confirm existing MCP Servers docs don't need updating beyond the new checkbox (covered by `029`'s closeout, since the checkbox's *effect* is documented there).

## Dependencies & Execution Order

- Phases 1–4 are historical (already implemented under the old FR-017) and listed for traceability only.
- Phase 5 (US4) depends only on Phase 1 (the existing type/store) — independently completable.
- `029-settings-dev-harness`'s harness-generation tasks depend on T040 landing here before its `generate-config.ts` can filter on the flag.

## Notes

- Historical tasks (T001–T030) reflect what the code already does, cross-checked against `types.ts`, `store.ts`, `client.ts`, `route.ts`, `tools/server/mcp.ts`, and `McpServersTab.tsx` while writing this file — not aspirational.
