# Implementation Plan: Settings — MCP Servers (configuration)

**Branch**: `030-settings-mcp-servers` | **Date**: 2026-07-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/bos-system-specs/030-settings-mcp-servers/spec.md`

## Summary

Most of this spec (US1–US3: transports, editor, Test, import, agent-parity tools) was already implemented under `000-browseros-core` FR-017 and is being **relocated here** for topic clarity — no behavior change. The only new work (US4) is the `includeInDevHarness` boolean on the MCP server entity plus a checkbox in the editor; the consumer of that flag is entirely `029-settings-dev-harness`'s responsibility.

## Technical Context

**Language/Version**: TypeScript (Node ≥ 20), Next.js App Router, server-only code under `src/lib/mcp/`.

**Primary Dependencies**: `@modelcontextprotocol/sdk` (`src/lib/mcp/client.ts`, existing). No new dependencies for the `includeInDevHarness` addition.

**Storage**: `data/mcp-servers.json` — `McpServerConfig[]`, upsert-by-name. `includeInDevHarness?: boolean` is a new optional field on that same array's entries (default/absent `false`) — no migration needed (absent = false).

**Testing**: `npx tsc --noEmit` + `npm run lint`. Manual verification per Success Criteria (add each transport, Test before/after save, import JSON, agent-tool parity, flag round-trip).

**Target Platform**: BOS server (Next.js), same process as the rest of the config system.

**Project Type**: Single project (BOS `src/`).

**Performance Goals**: N/A beyond existing connection timeouts (8s http/sse, 30s stdio, in `client.ts`).

**Constraints**: Server identity is name-keyed; renaming is delete-old+add-new (no rename-in-place primitive). Agent-driven and UI-driven configuration MUST share the same validation/store — no parallel code path.

**Scale/Scope**: Bounded by however many servers an installation configures; no hard limit.

## Constitution Check

*GATE: must pass before design; re-check after.*

- **I. Spec-Driven — SAAP**: this plan documents already-implemented behavior (relocated from `000-browseros-core`) plus one small new addendum (US4), each traceable to a requirement. PASS.
- **II. Server Authority & SSR Boundary**: all persistence/validation/connection logic is server-only (`src/lib/mcp/`, `src/app/api/mcp/route.ts`); the client only renders forms and calls the API. PASS.
- **III. Always Delegate; Claude Codes**: the addendum (US4) is source work delegated to the Developer harness on a feature branch. PASS.
- **IV. Minimize Blast Radius**: the addendum is one optional field + one checkbox; default `false` means zero behavior change for existing servers. PASS.
- **V. The VFS Is Not the Source**: N/A.
- **VI. Specs & Docs Stay in Sync**: this extraction **is** the sync fix — `000-browseros-core` FR-017 becomes a pointer to this spec instead of duplicating its content. `014-mcp-tool-gateway` is left as-is (its FR-002 dual concern — description-field existence vs. gateway consumption — is noted as a follow-up for the larger refactor, not resolved here). PASS.
- **VII. Respect Boundaries**: no new dependencies, no `package.json`/lockfile changes. PASS.

No violations → Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/bos-system-specs/030-settings-mcp-servers/
├── spec.md        # done
├── plan.md        # this file
└── tasks.md       # done
```

### Source Code (repository root, BOS `src/`)

```text
src/lib/mcp/
├── types.ts        # McpServerConfig — EDIT: + includeInDevHarness?: boolean (US4). Everything else pre-existing.
├── store.ts         # listMcpServers/addMcpServer/removeMcpServer — pre-existing, unchanged
├── client.ts        # connectMcpClient/probeMcpServer/extractText — pre-existing, unchanged
└── ui.ts             # MCP-UI passthrough helpers — pre-existing, unchanged

src/app/api/mcp/route.ts                        # normalizeConfig — EDIT: pass through includeInDevHarness (US4)
src/lib/assistant/tools/server/mcp.ts            # mcp_server_add's normalizeConfig — EDIT: same (US4);
                                                    # mcp_server_list/mcp_server_add/mcp_server_remove pre-existing (US3)
src/components/apps/settings/McpServersTab.tsx   # EDIT: add "Include in Dev Harness" checkbox (US4);
                                                    # everything else (transport picker, KV rows, Test, Import) pre-existing (US1/US2)
```

**Structure Decision**: single BOS project. The addendum touches only the entity's type/persistence/UI in three existing files — no new files, no new routes.

## Design Notes

### Pre-existing configuration surface (US1–US3, relocated from `000-browseros-core`)

`McpServersTab.tsx` is the single editor for both http/sse (endpoint, bearer token, custom headers via `KeyValueRows`) and stdio (command, args as newline-separated text, env via `KeyValueRows`, optional cwd) servers, plus a JSON import path (`{ mcpServers: {...} }` or bare map) and per-server Test (`probe`). `POST /api/mcp` accepts `{ test: true, ... }` to probe an unsaved config, or persists via `addMcpServer` (upsert-by-name) otherwise. `src/lib/assistant/tools/server/mcp.ts` exposes the identical operations as agent tools (`mcp_server_add`, `mcp_server_remove`, `mcp_server_list`), sharing `normalizeConfig` logic with the API route so there is exactly one validation path regardless of caller.

### Addendum: `includeInDevHarness` (US4)

One optional boolean, default/absent `false`. Added to `McpServerConfig` (`types.ts`), threaded through both `normalizeConfig` implementations (API route + agent tool — kept in sync since they're structurally identical), and exposed as a checkbox in `McpServersTab.tsx`'s editor (next to Description). This spec's responsibility ends at "the flag round-trips through save/load correctly" — `029-settings-dev-harness`'s `generate-config.ts` is solely responsible for reading `listMcpServers().filter(s => s.includeInDevHarness)` and doing anything with the result.

## Out of scope (v1)

- Anything about what `includeInDevHarness` produces (harness config generation) — entirely `029`'s.
- Per-agent server allowlisting mechanics — `011-per-agent-capabilities`.
- The gateway's use of `description` (index injection, defaulting logic) — `014-mcp-tool-gateway`.
- Resolving `014`'s FR-002 dual concern (description-field ownership) — deferred to the larger spec-store refactor the user has flagged as a later pass.

## Complexity Tracking

*(No constitution violations — table intentionally empty.)*
