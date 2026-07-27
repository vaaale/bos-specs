# Tasks: Settings — Dev Harness (credentials, provider & MCP servers)

Feature branch: `029-settings-dev-harness`. Phase A (credentials) was already implemented under `026-multiuser-usability` and is recorded as done here for traceability now that the topic has a dedicated home. Phases B and C (provider, MCP inclusion) are open.

## Phase A — Credentials (US1) — done, relocated from `026-multiuser-usability` Phase E

- [x] A1. Credential material for Claude/OpenCode is managed via a dedicated write-only route (`GET/POST /api/dev-harness/credentials`) + helpers in `harness-config.ts` (`hasClaudeCreds`/`hasOpenCodeAuth`/`writeClaudeCreds`/`writeOpenCodeAuth`/`clear*`). Kept out of the generic config namespace so raw secrets are never stored there or returned to the client — only a set/unset indicator.
- [x] A2. Credentials are written into a dedicated harness `HOME` (`{dataDir}/dev-harness/home/.claude/.credentials.json`, `.../.local/share/opencode/auth.json`) with dir `0o700` and files `0o600`; `harnessCredentialEnv()` sets `HOME`/`XDG_*` and is merged into `envForCwd()` in `claude-runner.ts` — only when credentials exist, so local dev with a real `~/.claude` is unaffected.
- [x] A3. `DevHarnessTab` UI: write-only credential textareas with SET/NOT SET indicators, Save + Clear per CLI, shown for CLI/OpenCode modes with container guidance.
- [x] A4. Typecheck + lint green.

## Phase B — Provider selection (US2) — done

**Goal**: A user can select a per-CLI auth provider instead of (or alongside) pasting credential material.

**Independent Test**: Select `api-key` for Claude CLI, enter a key, save, inspect the generated `.claude/settings.json`'s `env` block, confirm a `claude -p` run authenticates via it. — Verified via the live API (`PATCH /api/config`): the generated `.claude/settings.json` contained the correct `env` block; secrets round-tripped through the config store's existing masking (never echoed in the response).

- [x] B1. `lib/devharness/provider.ts` (NEW): `ClaudeProviderConfig`/`OpenCodeProviderConfig` per CLI (Claude: `default`/`api-key`/`bedrock`/`vertex`; OpenCode: `default`/named-`provider`), with `normalizeClaudeProvider`/`normalizeOpenCodeProvider` reading the raw `dev-harness` namespace.
- [x] B2. `lib/config/registry.ts`: provider fields added to the `dev-harness` schema (`claudeProviderMode`, `claudeApiKey` (secret), `claudeApiBaseUrl`, `claudeBedrockRegion`, `claudeBedrockProfile`, `claudeVertexProject`, `claudeVertexRegion`, `opencodeProviderMode`, `opencodeProviderId`, `opencodeApiKey` (secret), `opencodeBaseUrl`) — reuses the **generic** `secret: true` field mechanism already used by `ai-provider`'s `apiKey` (masked by `/api/config`'s existing `maskValues()`/`coerce()`), so no bespoke write-only endpoint was needed for these (unlike the OAuth-material credential fields, which are simple strings, not session material).
- [x] B3. `lib/devharness/generate-config.ts` (NEW, shared with Phase C): `regenerateHarnessConfigFiles()` — writes `.claude/settings.json`'s `env` block from the Claude provider mode; writes `.config/opencode/opencode.json`'s `provider`/`model` fields from the OpenCode provider mode. `mode: "default"` on either CLI produces an empty result, which `writeGeneratedFile()` turns into deleting the file (also what keeps `harness-config.ts`'s existence-check gate accurate).
- [x] B4. Wired into the `dev-harness` namespace's `save()` in `registry.ts`.
- [x] B5/B6/B7. `DevHarnessTab.tsx`: "Provider" section per CLI mode (mode selector + API key/base URL/region/profile/project fields as applicable), using a new `SecretInput` component (masked, "saved — type to replace" placeholder, mirroring `ConfigForm.tsx`'s existing secret-field UX) — plus a guidance line pointing at the MCP-inclusion checkbox. `npx tsc --noEmit` and `npx eslint` (scoped to changed files) both clean.

## Phase C — MCP server inclusion (US3) — done

**Goal**: An MCP server flagged `includeInDevHarness` (`030-settings-mcp-servers`) is folded into both generated harness files.

**Independent Test**: Flag an existing MCP server, save, confirm it appears in both `.claude.json` (`mcpServers`) and `opencode.json` (`mcp`) in that tool's native shape; unflag, confirm removal from both. — Verified live via `POST`/`DELETE /api/mcp`: a stdio server and an http server (with bearer token) both appeared correctly shaped in `.claude.json` and `opencode.json`; removing both servers deleted both generated files.

- [x] C1. `lib/devharness/generate-config.ts`: `regenerateHarnessConfigFiles()` reads `listMcpServers().filter(s => s.includeInDevHarness === true)` and writes `.claude.json`'s `mcpServers` block (stdio → `command`/`args`/`env`; http/sse → `{ type, url, headers }`, bearer token merged into `headers.Authorization` the same way `mcp/client.ts`'s `httpHeaders()` does) and `opencode.json`'s `mcp` block (`type: "local"|"remote"`, `command: [...]`/`environment` or `url`/`headers`, `enabled: true`).
- [x] C2. Wired into `addMcpServer`/`removeMcpServer` in `lib/mcp/store.ts` directly (rather than at the two API/tool call sites) — both functions call `regenerateHarnessConfigFiles()` after persisting. This creates a `mcp/store.ts` ⇄ `devharness/generate-config.ts` import (store imports the regenerate function; generate-config imports `listMcpServers` from store), but both references are only used inside async function bodies (never at module-evaluation time), so it resolves cleanly — confirmed by a clean `tsc`/build.
- [x] C3. `harness-config.ts`: implemented as a single `hasGeneratedHarnessConfig()` (checks existence of all three generated files) rather than two separate `hasNonDefaultProvider()`/`hasIncludedMcpServers()` functions — simpler because `generate-config.ts` already deletes a file the moment it would be empty, so file-existence *is* "is there anything to honor". `harnessCredentialEnv()`'s gate is `!hasClaudeCreds() && !hasOpenCodeAuth() && !hasGeneratedHarnessConfig()`. Confirmed the four-way-false case (no creds, no generated files) still returns `{}`.
- [x] C4. Typecheck + lint green; manually verified end-to-end via the live dev server and its API: provider save → correct generated `.claude/settings.json`; MCP flag → correct `.claude.json`/`opencode.json`; unflag/reset-to-default → both files deleted again. Could not visually verify the Settings UI in a real browser (no browser-automation tool available in this environment) — the API-level behavior these components call into was exercised directly instead.

## Closeout

- [x] Z1. Removed the relocated content from `026-multiuser-usability` (User Story 10, FR-020–022, "Harness credential set" entity, Phase E tasks), replaced with a short cross-reference to this spec.
- [x] Z2. Added a `029` row to `overview.md`'s feature map.
- [x] Z3. Updated `docs/dev/mcp/mcp.md` (`includeInDevHarness` field) and `docs/dev/assistant/sub-agents-and-delegation.md` (new "Provider selection & MCP-server inclusion" subsection, clarifying the generated files live in the harness `HOME` not the `cwd`/worktree) + `docs/usage/settings/dev-harness.md` (Provider, Giving the harness your MCP servers) and `docs/usage/mcp/mcp-servers.md` ("Include in Dev Harness" checkbox, cross-linked).
- [x] Z4. Final typecheck gate: `npx tsc --noEmit` clean (no output); `npx eslint` on all changed files clean.

## Phase D — UI reorganization & field rename (US4) — done

**Goal**: Replace the single stacked-vertical layout with a top-level Claude/OpenCode choice and two dependency-ordered, enable/disable panels.

**Independent Test**: Open Settings → Dev Harness; confirm the harness choice appears first, both panels are always rendered with only the selected one enabled, and each panel's fields read top-to-bottom in dependency order (run mode → auth method → its fields → model for Claude; auth method → its fields → model for OpenCode). Save with the pre-redesign field names still on disk and confirm they load correctly into the new UI. — Verified live: seeded a legacy-shaped `dev-harness.json` (`transport: "opencode"`, `model`, `claudeProviderMode: "api-key"`), confirmed `GET /api/config` derived `harness/claudeRunMode/claudeAuthMethod/claudeModel/opencodeModel` correctly including the secret's `secretsSet`; then `PATCH`ed with new-shape values and confirmed the legacy keys were scrubbed from disk while the preserved secret (empty-string = keep-existing) still produced a correct generated `.claude/settings.json`.

- [x] D1. `lib/devharness/provider.ts`: renamed `ClaudeProviderMode`/`OpenCodeProviderMode` → `ClaudeAuthMethod`/`OpenCodeAuthMethod`, `"default"` → `"credential-file"` for both; `normalizeClaudeProvider`/`normalizeOpenCodeProvider` read the new field first, falling back to the legacy `claudeProviderMode`/`opencodeProviderMode` field (mapping old `"default"` → new `"credential-file"`) when absent — this fallback lives in the normalize functions themselves (not just `registry.ts`'s `load()`), so `generate-config.ts`'s direct `readNamespace()` call also benefits from it. Added `resolveHarnessSelection()` (new) for the top-level `harness`/`claudeRunMode`/`claudeModel`/`opencodeModel` fields, with the same kind of legacy-`transport`/`model` fallback.
- [x] D2. `lib/config/registry.ts`: replaced the `dev-harness` schema's `transport`/`model`/`claudeProviderMode`/`opencodeProviderMode` fields with `harness`, `claudeRunMode`, `claudeModel`, `opencodeModel`, `claudeAuthMethod`, `opencodeAuthMethod` (kept `command`/`url` and the provider-specific fields as-is). `load()` calls `resolveHarnessSelection`/`normalizeClaudeProvider`/`normalizeOpenCodeProvider` so the UI always sees derived, current-shape values; `save()` explicitly `delete`s the four legacy keys before writing, so the on-disk shape self-migrates to the new field names on first save.
- [x] D3. `lib/devharness/harness-config.ts`'s `getHarnessConfig()`: now calls `resolveHarnessSelection(v)` instead of reading `transport`/`model` directly; output type (`HarnessConfig` union) unchanged, so `claude-runner.ts`/`/api/dev-harness` routes needed no changes.
- [x] D4. `lib/devharness/generate-config.ts`: `model` now comes from `resolveHarnessSelection(raw).opencodeModel` instead of the old shared `raw.model`; the credential-file/unconfigured checks in `claudeEnvBlock`/the OpenCode provider branch already compared against literal mode strings that carried over unchanged (`"api-key"`/`"bedrock"`/`"vertex"`/`"provider"`), so only the `model` source needed updating.
- [x] D5. `components/apps/settings/DevHarnessTab.tsx`: full layout rework — top `harness` select first; two always-rendered `HarnessPanel`s (Claude, OpenCode), the inactive one `disabled` on every control + dimmed (`opacity-60`) rather than unmounted; within each panel, fields follow FR-010/011/012's order (Claude: run mode →, only for `cli`, auth method → its fields → model; OpenCode: auth method → its fields → model). `CredentialField`/`SecretInput` both gained a `disabled` prop threaded from the panel's active state.
- [x] D6. Typecheck (`npx tsc --noEmit`) and lint (`npx eslint`, scoped + full) clean. Manually verified via the live dev server: fresh load derives sane defaults; a legacy-shaped `dev-harness.json` derives correctly (see Independent Test above) with no data loss; saving through the new shape scrubs the legacy keys and preserves secrets correctly; generation (Phases B/C) unaffected by the rename.

## Dependencies & Execution Order

- Phase A is historical, listed for traceability only.
- Phase B is independently completable once Phase A's `harness-config.ts`/`registry.ts` extension points are read.
- Phase C depends on `030-settings-mcp-servers` T040 (the flag existing on the entity) and on Phase B's `generate-config.ts` (extends the same function rather than creating a second one).
- Phase D depends on Phases B and C (it renames fields/enums they introduced) and is otherwise UI/field-naming only — no new generation logic.
- Closeout runs after B, C, and D all land.

## Notes

- Phase A tasks reflect what the code already does, cross-checked against `harness-config.ts`, `claude-runner.ts`, and `DevHarnessTab.tsx` while writing this file — not aspirational.
