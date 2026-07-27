# Tasks: Settings — Dev Harness (credentials, provider & MCP servers)

Feature branch: `029-settings-dev-harness`. Phase A (credentials) was already implemented under `026-multiuser-usability` and is recorded as done here for traceability now that the topic has a dedicated home. Phases B and C (provider, MCP inclusion) are open.

## Phase A — Credentials (US1) — done, relocated from `026-multiuser-usability` Phase E

- [x] A1. Credential material for Claude/OpenCode is managed via a dedicated write-only route (`GET/POST /api/dev-harness/credentials`) + helpers in `harness-config.ts` (`hasClaudeCreds`/`hasOpenCodeAuth`/`writeClaudeCreds`/`writeOpenCodeAuth`/`clear*`). Kept out of the generic config namespace so raw secrets are never stored there or returned to the client — only a set/unset indicator.
- [x] A2. Credentials are written into a dedicated harness `HOME` (`{dataDir}/dev-harness/home/.claude/.credentials.json`, `.../.local/share/opencode/auth.json`) with dir `0o700` and files `0o600`; `harnessCredentialEnv()` sets `HOME`/`XDG_*` and is merged into `envForCwd()` in `claude-runner.ts` — only when credentials exist, so local dev with a real `~/.claude` is unaffected.
- [x] A3. `DevHarnessTab` UI: write-only credential textareas with SET/NOT SET indicators, Save + Clear per CLI, shown for CLI/OpenCode modes with container guidance.
- [x] A4. Typecheck + lint green.

## Phase B — Provider selection (US2) — OPEN

**Goal**: A user can select a per-CLI auth provider instead of (or alongside) pasting credential material.

**Independent Test**: Select `api-key` for Claude CLI, enter a key, save, inspect the generated `.claude/settings.json`'s `env` block, confirm a `claude -p` run authenticates via it.

- [ ] B1. `lib/devharness/provider.ts` (NEW): `HarnessProviderConfig` discriminated union per CLI (Claude: `default`/`api-key`/`bedrock`/`vertex`; OpenCode: `default`/named-`provider`), with validation/defaults.
- [ ] B2. `lib/config/registry.ts`: persist provider config alongside `transport`/`model` in the `dev-harness` namespace; provider API keys get the same write-only/secret treatment as existing credential fields (return a set/unset indicator, never the value).
- [ ] B3. `lib/devharness/generate-config.ts` (NEW, shared with Phase C): `regenerateHarnessConfigFiles()` — Claude branch writes `.claude/settings.json`'s `env` block from the Claude provider mode; OpenCode branch writes `.config/opencode/opencode.json`'s `provider`/`model` fields from the OpenCode provider mode. `mode: "default"` on either CLI omits provider-specific fields entirely.
- [ ] B4. Wire `regenerateHarnessConfigFiles()` into the `dev-harness` namespace's `save()` in `registry.ts`.
- [ ] B5. `harness-config.ts`: broaden `harnessCredentialEnv()`'s gate to also cover a non-default provider (see Phase D for the full gate). 
- [ ] B6. `DevHarnessTab.tsx`: add a "Provider" section per CLI mode (shown alongside the existing Model field for `cli`/`opencode` transports) — mode selector + the relevant fields (API key, base URL, region/profile/project as applicable).
- [ ] B7. Typecheck + lint green.

## Phase C — MCP server inclusion (US3) — OPEN

**Goal**: An MCP server flagged `includeInDevHarness` (`030-settings-mcp-servers`) is folded into both generated harness files.

**Independent Test**: Flag an existing MCP server, save, confirm it appears in both `.claude.json` (`mcpServers`) and `opencode.json` (`mcp`) in that tool's native shape; unflag, confirm removal from both.

**Depends on**: `030-settings-mcp-servers` Phase 5 (T040: `includeInDevHarness` field must exist on `McpServerConfig` before this phase can filter on it).

- [ ] C1. `lib/devharness/generate-config.ts`: extend `regenerateHarnessConfigFiles()` (from Phase B) to also read `listMcpServers().filter(s => s.includeInDevHarness)` and write `.claude.json`'s `mcpServers` block (stdio → `command`/`args`/`env`; http/sse → `{ type, url, headers }`) and `opencode.json`'s `mcp` block (`type: "local"|"remote"`, `command: [...]`/`environment` or `url`/`headers`, `enabled: true`).
- [ ] C2. Wire `regenerateHarnessConfigFiles()` into `addMcpServer`/`removeMcpServer` in `lib/mcp/store.ts` (`030`'s file — a small, additive hook call, not a restructuring of that module).
- [ ] C3. `harness-config.ts`: extend `harnessCredentialEnv()`'s gate (started in B5) to its final form — `!hasClaudeCreds() && !hasOpenCodeAuth() && !hasNonDefaultProvider() && !hasIncludedMcpServers()` — confirm the four-way-false case still returns `{}` unchanged (no behavior change for installs that touch none of this).
- [ ] C4. Typecheck + lint green; manual verification: flag a server, save, confirm both generated files; unflag, confirm removal from both; confirm the untouched-install case produces no generated files and no redirection.

## Closeout

- [ ] Z1. Remove the relocated content from `026-multiuser-usability` (User Story 10, FR-020–022, "Harness credential set" entity, Phase E tasks), replacing it with a short cross-reference to this spec.
- [ ] Z2. Add a `029` row to `overview.md`'s feature map.
- [ ] Z3. Update `docs/dev/**` (Dev Harness provider modes; the generated, never-hand-edit `.claude/settings.json` / `.claude.json` / `opencode.json` files) and `docs/usage/**` (the "Include in Dev Harness" checkbox, cross-linking to `030`'s MCP Servers docs).
- [ ] Z4. Final typecheck gate: `npx tsc --noEmit` + `npm run lint` clean.

## Dependencies & Execution Order

- Phase A is historical, listed for traceability only.
- Phase B is independently completable once Phase A's `harness-config.ts`/`registry.ts` extension points are read.
- Phase C depends on `030-settings-mcp-servers` T040 (the flag existing on the entity) and on Phase B's `generate-config.ts` (extends the same function rather than creating a second one).
- Closeout runs after B and C both land.

## Notes

- Phase A tasks reflect what the code already does, cross-checked against `harness-config.ts`, `claude-runner.ts`, and `DevHarnessTab.tsx` while writing this file — not aspirational.
