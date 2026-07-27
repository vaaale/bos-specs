# Implementation Plan: Settings — Dev Harness (credentials, provider & MCP servers)

**Branch**: `029-settings-dev-harness` | **Date**: 2026-07-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/bos-system-specs/029-settings-dev-harness/spec.md`

## Summary

US1 (credential material → dedicated harness `HOME`) was already implemented under `026-multiuser-usability` and is relocated here for topic clarity — no behavior change. US2 (per-CLI provider selection) and US3 (MCP-server inclusion via `030-settings-mcp-servers`'s `includeInDevHarness` flag) are new: both are realized by **generating** (never hand-editing) the harness's own native config files inside the dedicated harness `HOME` — Claude's `~/.claude/settings.json` (provider `env` block) + `~/.claude.json` (`mcpServers`), and OpenCode's `~/.config/opencode/opencode.json` (`provider`/`model` fields + `mcp` block). `harnessCredentialEnv()`'s HOME/XDG redirection is broadened to trigger on any of credentials, provider, or included MCP servers — not credentials alone.

**2026-07-27 addendum (US4)**: the first shipped UI stacked every field in one undifferentiated column, with model appearing *before* auth — reported back as unintuitive. Reworked to: a single top-level `harness: "claude"|"opencode"` choice, then two always-rendered panels (one per CLI, only the selected one enabled), each internally ordered by real dependency (Claude: run mode → auth method → that method's fields → model; OpenCode: auth method → its fields → model). The credential-file paste box (US1) is folded in as one of the auth-method options rather than a separately-positioned section. This renames several `dev-harness` namespace fields (`transport`→`harness`+`claudeRunMode`, `model`→`claudeModel`/`opencodeModel`, `claudeProviderMode`/`opencodeProviderMode`→`claudeAuthMethod`/`opencodeAuthMethod` with `"default"` renamed to `"credential-file"`/`"auth-file"`); `load()` derives the new fields from the old at read time when the new ones are absent, so nothing is lost for anyone who saved settings under the very first implementation.

## Technical Context

**Language/Version**: TypeScript (Node ≥ 20), Next.js App Router; server-only code under `src/lib/devharness/`.

**Primary Dependencies**: none new. Reuses the existing config store (`readNamespace`/`writeNamespace`/`patchNamespace`) and `writeFileAtomic`.

**Storage**: credential material and provider config persist in the `dev-harness` config namespace (`data/config/dev-harness.json`), under the (post-redesign) field names `harness`, `claudeRunMode`, `command`, `url`, `claudeModel`, `opencodeModel`, `claudeAuthMethod`, `claudeApiKey`, `claudeApiBaseUrl`, `claudeBedrockRegion`, `claudeBedrockProfile`, `claudeVertexProject`, `claudeVertexRegion`, `opencodeAuthMethod`, `opencodeProviderId`, `opencodeApiKey`, `opencodeBaseUrl` — the pre-redesign names (`transport`, `model`, `claudeProviderMode`, `opencodeProviderMode`) remain readable indefinitely via a read-time derivation in `load()`. Provider API keys get the same write-only/secret treatment as existing credential fields. NEW generated files inside the harness `HOME` (regenerated in full on every relevant save, never hand-merged): `.claude/settings.json` (Claude provider `env` block), `.claude.json` (Claude `mcpServers`), `.config/opencode/opencode.json` (OpenCode `provider`/`model`/`mcp`). Reads `data/mcp-servers.json` (`030-settings-mcp-servers`'s store) filtered on `includeInDevHarness`.

**Testing**: `npx tsc --noEmit` + `npm run lint`. Manual verification: select a non-default provider, save, confirm the generated file's shape; flag an MCP server, save, confirm it appears in both generated files; unflag, confirm removal; confirm the four-way-false case (no credentials, default providers, no flagged servers) produces no redirection and no generated files (unchanged from pre-existing behavior).

**Target Platform**: BOS server (Next.js) — standalone or behind the Bastion; no Docker/Bastion-specific code path.

**Project Type**: Single project (BOS `src/`).

**Performance Goals**: N/A — config generation is a small, infrequent (on-save) file write, not a hot path.

**Constraints**: Generated files are full-replace and MUST NOT be hand-merged with pre-existing content at those paths. Provider secrets and credential material MUST NOT be logged or returned to the client after save.

**Scale/Scope**: Bounded by two CLIs (Claude, OpenCode) and however many MCP servers are flagged for inclusion.

## Constitution Check

*GATE: must pass before design; re-check after.*

- **I. Spec-Driven — SAAP**: US1 is documented as already-implemented (relocated from `026`); US2/US3 follow spec → plan → tasks order. PASS.
- **II. Server Authority & SSR Boundary**: all credential/provider/generation logic is server-only (`src/lib/devharness/`); secrets never returned to the client. PASS.
- **III. Always Delegate; Claude Codes**: implementation runs through the Developer harness on a feature branch. PASS.
- **IV. Minimize Blast Radius**: additive fields + one new generation module; the `default`/`false`/unset case for every new field reproduces exactly today's behavior (verified explicitly in Edge Cases and SC). PASS.
- **V. The VFS Is Not the Source**: N/A.
- **VI. Specs & Docs Stay in Sync**: this extraction from `026` (which now cross-references this spec instead of owning the content) **is** the sync fix for this topic; `docs/dev/**` updates for the new provider/MCP-inclusion mechanics are this spec's closeout, not `026`'s.
- **VII. Respect Boundaries**: no new dependencies, no `package.json`/lockfile changes.

No violations → Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/bos-system-specs/029-settings-dev-harness/
├── spec.md        # done
├── plan.md        # this file
└── tasks.md       # done
```

### Source Code (repository root, BOS `src/`)

```text
src/lib/devharness/
├── harness-config.ts   # EDIT — broaden harnessCredentialEnv()'s gate (US1 pre-existing; US2/US3 new triggers)
├── provider.ts          # NEW — HarnessProviderConfig discriminated union per CLI + validation/defaults (US2)
└── generate-config.ts   # NEW — regenerateHarnessConfigFiles(): writes .claude/settings.json, .claude.json,
                            # opencode.json from provider config + listMcpServers().filter(includeInDevHarness) (US2/US3)

src/lib/config/registry.ts             # EDIT — dev-harness namespace: add provider fields (US2)
src/lib/mcp/store.ts                    # EDIT — addMcpServer/removeMcpServer call regenerateHarnessConfigFiles() (US3)
                                           # (McpServerConfig.includeInDevHarness itself is 030's field/task, not this spec's)
src/lib/agent/subagents/claude-runner.ts # unchanged — envForCwd() already calls harnessCredentialEnv()
src/components/apps/settings/DevHarnessTab.tsx  # EDIT — full layout rework: top-level harness choice + two
                                                   # enable/disable per-CLI panels, dependency-ordered fields (US4)
```

**Structure Decision**: single BOS project; all new logic lives under `src/lib/devharness/`, touching `registry.ts` and `mcp/store.ts` only at their existing extension points (config schema, post-save hook).

## Design Notes

### Pre-existing credentials (US1, relocated from `026`)

`harness-config.ts` already writes Claude (`~/.claude/.credentials.json`) and OpenCode (`~/.local/share/opencode/auth.json`) credential material into a dedicated harness `HOME` (`{dataDir}/dev-harness/home/`) with `0o700`/`0o600` permissions, and `harnessCredentialEnv()` redirects `HOME`/`XDG_*` when either is set. `DevHarnessTab.tsx`'s `CredentialField` is write-only (paste to set, Save/Clear, SET/NOT SET indicator only). Unchanged by this plan except for the broadened gate below.

### Provider config (US2)

`lib/devharness/provider.ts`: a small discriminated union per CLI —
`{ tool: "claude"; mode: "default" } | { tool: "claude"; mode: "api-key"; apiKey: string; baseUrl?: string } | { tool: "claude"; mode: "bedrock"; region?: string; profile?: string } | { tool: "claude"; mode: "vertex"; project?: string; region?: string }`
and analogously for OpenCode: `{ tool: "opencode"; mode: "default" } | { tool: "opencode"; mode: "provider"; providerId: string; apiKey?: string; baseUrl?: string }`. Persisted in the `dev-harness` config namespace next to `transport`/`model`; `apiKey` fields get the same secret handling as existing credential fields (write-only in the API — the loader in `registry.ts` returns a set/unset indicator, not the value, mirroring the existing `creds` pattern in `DevHarnessTab.tsx`).

### Config generation (US2 + US3)

`lib/devharness/generate-config.ts`: one function, `regenerateHarnessConfigFiles()`, reads the current `dev-harness` config + provider config + `listMcpServers().filter(s => s.includeInDevHarness)` (from `030-settings-mcp-servers`'s store), and rewrites (full replace, `writeFileAtomic`) three files under the harness `HOME`:
- `.claude/settings.json` — `{ env: { ANTHROPIC_API_KEY?, ANTHROPIC_BASE_URL?, CLAUDE_CODE_USE_BEDROCK?, CLAUDE_CODE_USE_VERTEX?, ... } }` derived from the Claude provider mode; empty/omitted when `mode: "default"`.
- `.claude.json` — `{ mcpServers: { <name>: { command, args, env } | { type: "http"|"sse", url, headers } } }` from every `includeInDevHarness` server, translating BOS's `McpServerConfig` shape into Claude's native `mcpServers` entry shape per transport.
- `.config/opencode/opencode.json` — `{ provider: { <id>: { options: { apiKey, baseURL } } }, model: "<id>/<model>", mcp: { <name>: { type: "local"|"remote", command: [...], environment, url, headers, enabled: true } } }` from the OpenCode provider mode + the same MCP server list, translated to OpenCode's `mcp` shape.

Called from two places so either surface staying in sync with the other doesn't require re-saving both: the `dev-harness` config's `save()` in `registry.ts` (provider/transport/model changed), and `addMcpServer`/`removeMcpServer` in `lib/mcp/store.ts` (a server's `includeInDevHarness` flag changed, owned by `030`). Regeneration is a full, idempotent rewrite — BOS never merges into these files, so they MUST NOT be hand-edited (documented in `docs/dev/`).

### Env-redirection gate (US1 pre-existing + US2/US3 new triggers)

`harnessCredentialEnv()`'s existing `if (!hasClaudeCreds() && !hasOpenCodeAuth()) return {}` guard is broadened to `if (!hasClaudeCreds() && !hasOpenCodeAuth() && !hasNonDefaultProvider() && !hasIncludedMcpServers()) return {}`, so `HOME`/`XDG_*` redirect whenever there's anything in the harness `HOME` worth the CLI reading — not only pasted credential material. Local dev with a real `~/.claude` and no BOS-side customization at all is unaffected (all four checks false → no redirection, same as today).

### UI reorganization & field rename (US4)

**Top-level + two panels**: `DevHarnessTab.tsx` renders one `harness` select (Claude Code / OpenCode) first, then always renders both a Claude panel and an OpenCode panel — the one NOT matching `harness` gets its inputs `disabled` and the panel dimmed (`opacity`), rather than being conditionally unmounted, so the user can see the full shape of both configurations at a glance.

**Per-panel field order**: Claude panel — `claudeRunMode` select (`cli`/`stdio`/`http`/`sse`) first; when `cli`, then `claudeAuthMethod` select (`credential-file`/`api-key`/`bedrock`/`vertex`) and that method's own fields (the existing `CredentialField` paste box for `credential-file`; the existing generic secret/text inputs for the other three); then `claudeModel`. When `claudeRunMode` is `stdio`/`http`/`sse`, the panel instead shows only the existing MCP `command`/`url` field (auth/model don't apply to an already-running remote harness). OpenCode panel — `opencodeAuthMethod` select (`credential-file`/`provider`) and its fields, then `opencodeModel`; no run-mode choice (OpenCode is always a local headless spawn).

**Field rename + back-compat derivation** (`registry.ts`'s `load()`): the `dev-harness` schema's fields become `harness`, `claudeRunMode`, `command`, `url`, `claudeAuthMethod`, `claudeApiKey`, `claudeApiBaseUrl`, `claudeBedrockRegion`, `claudeBedrockProfile`, `claudeVertexProject`, `claudeVertexRegion`, `claudeModel`, `opencodeAuthMethod`, `opencodeProviderId`, `opencodeApiKey`, `opencodeBaseUrl`, `opencodeModel`. `provider.ts`'s mode enums are renamed to match (`ClaudeAuthMethod = "credential-file" | "api-key" | "bedrock" | "vertex"`; `OpenCodeAuthMethod = "credential-file" | "provider"`) — the "is this the credential-file/unconfigured case" check that `generate-config.ts` and the env-redirection gate rely on now compares against `"credential-file"`/`"credential-file"` instead of `"default"`. `load()` derives the new fields from the legacy `transport`/`model`/`claudeProviderMode`/`opencodeProviderMode` when the new fields are absent (see spec Clarifications for the exact mapping); `save()` only ever writes the new field names going forward.

## Out of scope (v1)

- The MCP server entity, its editor, and the `includeInDevHarness` checkbox itself — `030-settings-mcp-servers`.
- Docker/Bastion-specific concerns (image contents, container detection) — `026-multiuser-usability`.
- Per-agent capability scoping of anything — `011-per-agent-capabilities`.

## Complexity Tracking

*(No constitution violations — table intentionally empty.)*
