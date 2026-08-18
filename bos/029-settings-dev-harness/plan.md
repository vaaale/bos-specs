# Implementation Plan: Settings — Dev Harness (credentials, provider & MCP servers)

**Branch**: `029-settings-dev-harness` | **Date**: 2026-07-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/bos-system-specs/029-settings-dev-harness/spec.md`

## Summary

US1 (credential material → dedicated harness `HOME`) was already implemented under `026-multiuser-usability` and is relocated here for topic clarity — no behavior change. US2 (per-CLI provider selection) and US3 (MCP-server inclusion via `030-settings-mcp-servers`'s `includeInDevHarness` flag) are new: both are realized by **generating** (never hand-editing) the harness's own native config files inside the dedicated harness `HOME` — Claude's `~/.claude/settings.json` (provider `env` block) + `~/.claude.json` (`mcpServers`), and OpenCode's `~/.config/opencode/opencode.json` (`provider`/`model` fields + `mcp` block). `harnessCredentialEnv()`'s HOME/XDG redirection is broadened to trigger on any of credentials, provider, or included MCP servers — not credentials alone.

**2026-07-27 addendum (US4)**: the first shipped UI stacked every field in one undifferentiated column, with model appearing *before* auth — reported back as unintuitive. Reworked to: a single top-level `harness: "claude"|"opencode"` choice, then two always-rendered panels (one per CLI, only the selected one enabled), each internally ordered by real dependency (Claude: run mode → auth method → that method's fields → model; OpenCode: auth method → its fields → model). The credential-file paste box (US1) is folded in as one of the auth-method options rather than a separately-positioned section. This renames several `dev-harness` namespace fields (`transport`→`harness`+`claudeRunMode`, `model`→`claudeModel`/`opencodeModel`, `claudeProviderMode`/`opencodeProviderMode`→`claudeAuthMethod`/`opencodeAuthMethod` with `"default"` renamed to `"credential-file"`/`"auth-file"`); `load()` derives the new fields from the old at read time when the new ones are absent, so nothing is lost for anyone who saved settings under the very first implementation.

**2026-07-27 addendum (US5)**: OpenCode's free-text "Provider id" field was verified against OpenCode's own docs to be unsafe — an id outside OpenCode's built-in catalog silently produces a non-functional config (no `npm`/`models` fields, which unrecognized providers require). Replaced with a dropdown of real provider ids (credential-file, 9 generic-fields-only providers, Bedrock, Vertex, Azure, Custom), each revealing only its own fields. This surfaced a real architectural split: Bedrock/Custom are file-based (`opencode.json` `provider.options`/`npm`/`models`, same mechanism as before); Vertex/Azure are **environment-variable-only per OpenCode's own docs** (`GOOGLE_APPLICATION_CREDENTIALS`/`GOOGLE_CLOUD_PROJECT`/`VERTEX_LOCATION`, `AZURE_RESOURCE_NAME`) and so must be injected into the spawned `opencode` process's environment (`claude-runner.ts`) rather than written to a file — unlike Claude Code, which has its own `env` block inside `settings.json` for exactly this purpose, so its Bedrock/Vertex modes stay fully file-based. Vertex additionally needs a service-account JSON, handled as a third write-only credential-file paste box (same pattern as Claude's/OpenCode's existing ones).

## Technical Context

**Language/Version**: TypeScript (Node ≥ 20), Next.js App Router; server-only code under `src/lib/devharness/`.

**Primary Dependencies**: none new. Reuses the existing config store (`readNamespace`/`writeNamespace`/`patchNamespace`) and `writeFileAtomic`.

**Storage**: credential material and provider config persist in the `dev-harness` config namespace (`data/config/dev-harness.json`), under the (post-redesign) field names `harness`, `claudeRunMode`, `command`, `url`, `claudeModel`, `opencodeModel`, `claudeAuthMethod`, `claudeApiKey`, `claudeApiBaseUrl`, `claudeBedrockRegion`, `claudeBedrockProfile`, `claudeVertexProject`, `claudeVertexRegion`, `opencodeAuthMethod` (now the full provider dropdown value, e.g. `"amazon-bedrock"`/`"google-vertex"`/`"azure"`/`"custom"`/one of the generic-fields-only ids), `opencodeApiKey`, `opencodeBaseUrl`, `opencodeBedrockRegion`, `opencodeBedrockProfile`, `opencodeBedrockEndpoint`, `opencodeVertexProject`, `opencodeVertexLocation`, `opencodeAzureResourceName`, `opencodeCustomProviderId`, `opencodeCustomNpmPackage`, `opencodeCustomModelId` — the pre-redesign names (`transport`, `model`, `claudeProviderMode`, `opencodeProviderMode`, and the old free-text `opencodeProviderId`) remain readable indefinitely via a read-time derivation in `load()`/`normalizeOpenCodeProvider()`. Provider API keys get the same write-only/secret treatment as existing credential fields. NEW third credential file: `.config/gcloud/opencode-vertex-sa.json` under the harness `HOME` (write-only, same 0700/0600 permissions as the existing two). NEW generated files inside the harness `HOME` (regenerated in full on every relevant save, never hand-merged): `.claude/settings.json` (Claude provider `env` block), `.claude.json` (Claude `mcpServers`), `.config/opencode/opencode.json` (OpenCode `provider`/`model`/`mcp` — omitting Vertex/Azure's env-var-only fields). Reads `data/mcp-servers.json` (`030-settings-mcp-servers`'s store) filtered on `includeInDevHarness`.

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
├── harness-config.ts   # EDIT — broaden harnessCredentialEnv()'s gate (US1/US4); add vertex service-account
│                          # credential file helpers (US5, same pattern as claude/opencode creds)
├── provider.ts          # EDIT — OpenCodeAuthMethod becomes the full provider dropdown; OpenCodeProviderConfig
│                          # gains per-provider proprietary fields; openCodeProviderEnv() for Vertex/Azure env vars (US5)
└── generate-config.ts   # EDIT — OpenCode branch rewritten per selected provider; file-based (Bedrock/Custom)
                            # vs env-var-only (Vertex/Azure) split (US5)

src/lib/config/registry.ts             # EDIT — dev-harness namespace: OpenCode proprietary fields (US5)
src/lib/mcp/store.ts                    # unchanged (US3, already wired)
src/lib/agent/subagents/claude-runner.ts # EDIT — runOpenCodeCli merges openCodeProviderEnv() into the spawn
                                           # environment (US5) — the one caller that needs env vars beyond HOME/XDG
src/app/api/dev-harness/credentials/route.ts # EDIT — vertexServiceAccount set/clear, alongside claude/openCode (US5)
src/components/apps/settings/DevHarnessTab.tsx  # EDIT — full layout rework (US4); OpenCode auth-method dropdown +
                                                   # per-provider conditional fields incl. a third CredentialField (US5)
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

### OpenCode provider dropdown & proprietary fields (US5)

**Dropdown scope**: a curated ~13-entry list, not OpenCode's full ~48-provider catalog (explicit scoping decision, spec Clarifications) — the 9 providers needing nothing beyond `apiKey`/`baseURL` (Anthropic, OpenAI, OpenRouter, Groq, DeepSeek, Together AI, Fireworks AI, xAI, Ollama), the 3 with real proprietary fields (Bedrock, Vertex, Azure), and Custom.

**File-based providers** (Bedrock, the 9 generic ones, Custom) go into `opencode.json`'s `provider.<id>` exactly as before, just keyed by the dropdown selection instead of free text:
- Generic: `{ options: { apiKey?, baseURL? } }`.
- Bedrock (`amazon-bedrock`): `{ options: { region?, profile?, endpoint? } }` — no `apiKey` (AWS-credential-based, not API-key-based).
- Custom: `{ npm: <package, default "@ai-sdk/openai-compatible">, options: { apiKey?, baseURL? }, models: { <model id>: {} } }`, keyed by the user's own custom id — this is what makes an out-of-catalog provider actually functional, per OpenCode's own docs.

**Env-var-only providers** (Vertex, Azure) — confirmed via OpenCode's docs that these have NO `opencode.json` config surface for their required parameters:
- `openCodeProviderEnv(cfg: OpenCodeProviderConfig): Record<string,string>` (new, in `generate-config.ts` or `provider.ts`) returns, for `mode: "google-vertex"`: `GOOGLE_APPLICATION_CREDENTIALS` (path to the written service-account file, only if set), `GOOGLE_CLOUD_PROJECT`, `VERTEX_LOCATION`; for `mode: "azure"`: `AZURE_RESOURCE_NAME`. Empty object otherwise.
- `claude-runner.ts`'s `runOpenCodeCli` becomes `async function` (a one-line change — it already returns `Promise<AgentRunResult>`) so it can `await` this before spawning: `env: { ...envForCwd(cwd), ...(await getOpenCodeProviderEnv()) }`. `runClaudeCli` is untouched (Claude's Bedrock/Vertex stay fully file-based via `settings.json`'s own `env` block).
- Azure's API key is the one field that IS file-based even in an otherwise env-var-only provider (`opencode.json`'s `provider.azure.options.apiKey`) — the two mechanisms aren't mutually exclusive per provider, just per parameter.

**Vertex service-account file**: a third credential file, `harness-config.ts` gains `vertexServiceAccountPath()`/`hasVertexServiceAccount()`/`writeVertexServiceAccount()`/`clearVertexServiceAccount()`, mirroring `hasClaudeCreds()`/`writeClaudeCreds()` exactly (dir `0700`, file `0600`). `api/dev-harness/credentials/route.ts` gains a third GET field (`vertexServiceAccountSet`) and POST branch (`vertexServiceAccount`/`clearVertexServiceAccount`). `DevHarnessTab.tsx` renders a third `CredentialField` instance when OpenCode's auth method is `google-vertex`.

**Back-compat**: the old free-text `opencodeProviderId` (paired with `opencodeProviderMode: "provider"`) has no reliable way to know which dropdown bucket it belongs to — `normalizeOpenCodeProvider()`'s legacy fallback maps it to `"custom"` (preserving the typed id as `opencodeCustomProviderId`) rather than guessing it matches one of the 9 generic-fields providers, since a wrong guess would silently change behavior more than falling through to Custom (which at least preserves the original id verbatim, even though it now additionally needs an npm package to actually work — a case already covered by the Edge Cases in spec.md).

## Out of scope (v1)

- The MCP server entity, its editor, and the `includeInDevHarness` checkbox itself — `030-settings-mcp-servers`.
- Docker/Bastion-specific concerns (image contents, container detection) — `026-multiuser-usability`.
- Per-agent capability scoping of anything — `011-per-agent-capabilities`.
- OpenCode's full ~48-provider catalog (only a curated ~13-entry subset is wired; documented as a scoping decision, not an oversight).

## Complexity Tracking

*(No constitution violations — table intentionally empty.)*
