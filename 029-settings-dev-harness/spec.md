# Feature Specification: Settings — Dev Harness (credentials, provider & MCP servers)

**Feature Branch**: `029-settings-dev-harness`

**Created**: 2026-07-27 (extracted from `026-multiuser-usability` User Story 10 / FR-020–022 for topic clarity)

**Status**: Partially Implemented (credentials shipped under `026`; provider selection and MCP-server inclusion are new — see Requirements)

**Input**: "Add the possibility of configuring both the provider and MCP servers for the Dev Harness (Claude Code / OpenCode) in the Settings app. For MCP servers, a checkbox 'Include in Dev Harness' folds a configured server into the harness. Extract any and all information about this topic — including what was previously in `026-multiuser-usability` — into this dedicated spec."

> This spec owns **how the headless Dev Harness (Claude Code / OpenCode) authenticates and is configured**: credential material, per-CLI auth provider, and which MCP servers it gets — independent of whether BOS runs standalone or behind the Bastion (`026-multiuser-usability`, which owns only the Docker/multi-user paradigm itself; container deployment is this spec's original *motivating* use case, not a hard dependency of the mechanism). The MCP server **entity and its editor** (transport, endpoint, description, the `includeInDevHarness` flag itself) belong to `030-settings-mcp-servers`; this spec owns only what happens when that flag is set.

## Clarifications

### Session 2026-07-14 (originally recorded under `026-multiuser-usability`)

- Q: How should Claude Code / OpenCode credentials be provided in a container? → A: **Mount credential files.** The user provides the CLIs' own auth material (Claude `~/.claude`, OpenCode `auth.json`); BOS writes them into a dedicated harness `HOME` before spawning. This supports OAuth/subscription logins, not just API keys.

### Session 2026-07-27

- Q: How should the Dev Harness let a user pick an auth **provider**, not just paste credential material? → A: **Per-CLI provider modes.** Claude CLI: `default` (existing credential-file login, unchanged) | `api-key` (`ANTHROPIC_API_KEY`, optional `ANTHROPIC_BASE_URL`) | `bedrock` (`CLAUDE_CODE_USE_BEDROCK=1` + AWS region/profile) | `vertex` (`CLAUDE_CODE_USE_VERTEX=1` + GCP project/region) — materialized into the harness's own `~/.claude/settings.json` `env` block. OpenCode CLI: `default` (existing `auth.json` login, unchanged) | a named provider id (e.g. `anthropic`, `openai`, `openrouter`, or a custom OpenAI-compatible id) with `apiKey`/`baseURL` — materialized into the harness's own `~/.config/opencode/opencode.json` `provider` + `model` fields.
- Q: Where do MCP servers opted into the Dev Harness get written? → A: Verified against upstream docs that Claude Code does **not** read `mcpServers` from `settings.json` — user-scope MCP servers live in `~/.claude.json` under `mcpServers`. So Claude's generated file for this purpose is `~/.claude.json` (not `settings.json`, which stays limited to `env`/provider fields). OpenCode's own `mcp` block in `opencode.json` is used for OpenCode. Both are **generated, not hand-edited** — BOS rewrites them from this spec's config plus every MCP server flagged `includeInDevHarness` (`030-settings-mcp-servers`) whenever either changes.
- Q: When should the harness `HOME`/`XDG_*` redirection (`harnessCredentialEnv()`) apply? → A: Whenever there is **any** dev-harness customization to honor — raw pasted credentials (existing), a non-default provider, or at least one `includeInDevHarness` MCP server — not only when credential material has been pasted. Otherwise a provider- or MCP-only configuration would be generated but never read by the spawned CLI.
- Q: Should the "Include in Dev Harness" checkbox and its generation mechanics be described here or in the MCP-servers spec? → A: **Here.** `030-settings-mcp-servers` owns only the flag's existence/persistence/checkbox UI on the server entity; this spec owns everything about what reading that flag produces (which files, what shape, when regenerated).
- Q: Is this spec dependent on `026-multiuser-usability` (Docker/Bastion)? → A: **No.** The mechanism (credentials, provider, generated config, MCP inclusion) works identically standalone or behind the Bastion; only the *motivating scenario* for the original credential-paste feature was container use (no interactive TTY). `026` is scoped to the multi-user/Docker paradigm only and cross-references this spec, not the reverse.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Configure Claude Code / OpenCode credentials (Priority: P2)

The dev-harness settings let the user provide the Claude Code and OpenCode credential material; BOS writes it into a dedicated harness `HOME` so the headless CLIs authenticate without an interactive login (essential in a container, where there is no TTY to run an interactive `claude`/`opencode auth login`, but equally usable standalone).

**Why this priority**: Without this, the headless CLIs are dead on arrival wherever an interactive login isn't possible.

**Independent Test**: Paste Claude credential material, save, run a `claude -p` task, confirm it authenticates without prompting.

**Acceptance Scenarios**:

1. **Given** the Dev Harness settings, **When** the user provides Claude credential material and saves, **Then** a subsequent `claude -p` run authenticates using it (no interactive login).
2. **Given** OpenCode credential material provided, **When** an `opencode run` executes, **Then** it authenticates using the written `auth.json`.
3. **Given** the "Test" action, **When** run, **Then** it reports whether the configured CLI is reachable/authenticated.

### User Story 2 — Choose an auth provider per CLI (Priority: P2)

Instead of (or in addition to) pasting credential-file material, a user selects how each CLI should authenticate: a raw API key, a cloud-managed credential path (Bedrock/Vertex for Claude), or a named provider (OpenCode).

**Why this priority**: Credential-file login requires an interactive OAuth/subscription flow to obtain material from; some deployments (API-key-only accounts, enterprise Bedrock/Vertex billing, self-hosted OpenAI-compatible endpoints) have no such file to paste and need a first-class alternative.

**Independent Test**: Select `api-key` for Claude CLI, enter a key, save, confirm the generated `~/.claude/settings.json` has the right `env` block and a subsequent `claude -p` run authenticates via it.

**Acceptance Scenarios**:

1. **Given** a non-default provider selected for Claude CLI (`api-key`/`bedrock`/`vertex`), **When** the user saves, **Then** BOS regenerates `~/.claude/settings.json` inside the harness home with the corresponding `env` values, and a subsequent `claude -p` run picks them up without needing the credential-file login.
2. **Given** a named provider configured for OpenCode CLI, **When** the user saves, **Then** BOS regenerates `~/.config/opencode/opencode.json`'s `provider` and `model` fields inside the harness home, and a subsequent `opencode run` uses it instead of `auth.json`.
3. **Given** the provider is left as `default` for both CLIs, **When** the user saves, **Then** the generated files omit provider-specific fields and existing credential-file behavior is unchanged (no regression for users who never touch provider config).

### User Story 3 — Give the Dev Harness the MCP servers it needs (Priority: P2)

A user has already configured an MCP server for the assistant (`030-settings-mcp-servers`) and wants the headless Claude/OpenCode CLI to have it too, without re-entering it — via the "Include in Dev Harness" checkbox on that server.

**Why this priority**: Re-entering the same server (command/args/env or endpoint/headers) twice is error-prone and a maintenance burden; a headless coding agent frequently needs the same tools (e.g. an issue tracker, a private package registry) the chat assistant already has configured.

**Independent Test**: Flag an existing MCP server "Include in Dev Harness", save, inspect the harness home's `~/.claude.json` and `opencode.json`, confirm the server appears in each in that tool's native shape.

**Acceptance Scenarios**:

1. **Given** an MCP server configured in `030-settings-mcp-servers` and flagged `includeInDevHarness`, **When** the Dev Harness config is (re)generated, **Then** BOS writes the harness's `~/.claude.json` (`mcpServers`) and `~/.config/opencode/opencode.json` (`mcp`) to include it, mapped to each format's native shape (stdio → command/args/env; http/sse → url/headers).
2. **Given** an MCP server previously included, **When** the flag is unchecked and saved, **Then** it is removed from both generated harness files on the next regeneration.
3. **Given** no MCP servers are flagged for inclusion, **When** the harness config is generated, **Then** neither generated file gains an `mcpServers`/`mcp` block beyond what the harness itself needs — no empty or dangling blocks.

### Edge Cases

- Credential material is secret: it MUST be stored with the same protection as other config secrets and MUST NOT be echoed back to the client after save (write-only field with a "set/!set" indicator).
- Provider API keys (Claude `api-key` mode, OpenCode named-provider `apiKey`) are secrets: same write-only, never-echoed, never-logged treatment as harness credentials, even though they land in a generated **config** file rather than the dedicated credential files.
- An MCP server flagged `includeInDevHarness` with a transport BOS cannot translate 1:1 for a given CLI (none expected today — stdio/http/sse all map cleanly) MUST fail the generation step for that file with an actionable error rather than silently omitting the server.
- Regeneration is idempotent and full-replace per file: BOS never merges into a hand-edited `~/.claude.json` / `opencode.json` — the generated content there is authoritative and MUST NOT be treated as user-editable.
- Local dev with a real `~/.claude`/OpenCode login and no BOS-side customization (no credentials pasted, provider `default`, no servers flagged) MUST be completely unaffected — no `HOME`/`XDG_*` redirection, no generated files.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The `dev-harness` config MUST accept credential material for **Claude Code** (its `~/.claude` credentials/config) and **OpenCode** (`auth.json`). BOS MUST write this material into a dedicated harness `HOME` directory and set `HOME` (and any required env) when spawning `claude`/`opencode`, so the headless CLIs authenticate without interactive login.
- **FR-002**: Credential fields MUST be treated as secrets: stored with the config store's protection, never returned to the client after save (write-only with a set/unset indicator), and never logged.
- **FR-003**: The Dev Harness settings UI MUST let the user enter/update the credential material and MUST provide guidance for a container deployment; the existing "Test" action MUST report reachability/auth status.
- **FR-004**: The `dev-harness` config MUST let the user select a **provider** independently per CLI, on top of (not replacing) the existing credential-file login: for Claude CLI, one of `default` | `api-key` (`ANTHROPIC_API_KEY`, optional `ANTHROPIC_BASE_URL`) | `bedrock` (`CLAUDE_CODE_USE_BEDROCK=1` + region/profile) | `vertex` (`CLAUDE_CODE_USE_VERTEX=1` + project/region); for OpenCode CLI, `default` or a named provider (id + `apiKey`/`baseURL`). Provider secrets (API keys) MUST be treated as secrets per FR-002 even though they are written into a generated config file rather than the dedicated credential files.
- **FR-005**: BOS MUST generate — never hand-edit — the harness's own `~/.claude/settings.json` (Claude CLI: an `env` block derived from the selected provider) and `~/.config/opencode/opencode.json` (OpenCode CLI: `provider` + `model` fields derived from the selected provider) inside the dedicated harness `HOME`, rewriting both in full whenever the Dev Harness config changes. When the provider is `default`, the generated files MUST omit provider-specific fields so existing credential-file behavior is unaffected.
- **FR-006**: Every MCP server flagged **`includeInDevHarness`** (the flag itself is `030-settings-mcp-servers`'s field) MUST be folded into the harness's `~/.claude.json` (`mcpServers` block — NOT `settings.json`, which does not support MCP servers) and the OpenCode `mcp` block in `opencode.json`, mapping each BOS `McpServerConfig` transport (`stdio`/`http`/`sse`) to that tool's native shape. Regeneration MUST run on every Dev Harness config save AND every MCP server save/delete (so either surface changing keeps both files current).
- **FR-007**: `harnessCredentialEnv()`'s `HOME`/`XDG_*` redirection MUST apply whenever there is dev-harness customization to honor — raw pasted credentials (FR-001), a non-default provider (FR-004), or at least one `includeInDevHarness` MCP server (FR-006) — not only when credential material has been pasted, so the generated config files are actually read by the spawned CLI. When none of the three apply, behavior MUST be unchanged from before this spec (no redirection).

### Key Entities

- **Harness credential set** — Claude (`~/.claude` material) + OpenCode (`auth.json`) written into a dedicated `HOME`; secret, write-only in the API.
- **Harness provider config** — per-CLI provider selection (Claude: `default`/`api-key`/`bedrock`/`vertex`; OpenCode: `default`/named-provider), materialized into the harness's own generated `~/.claude/settings.json` / `~/.config/opencode/opencode.json`.
- **Harness MCP inclusion** — the generation step that reads every `includeInDevHarness`-flagged MCP server (`030-settings-mcp-servers`'s entity) and folds it into `~/.claude.json` (`mcpServers`) and `opencode.json` (`mcp`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: With Claude/OpenCode credentials configured, a headless harness run authenticates and completes without any interactive login.
- **SC-002**: Selecting a non-default provider for Claude or OpenCode CLI and saving produces a correctly-shaped generated config file, and a subsequent CLI run picks it up (verified via the existing "Test" action).
- **SC-003**: Flagging an MCP server "Include in Dev Harness" (`030-settings-mcp-servers`) and saving makes it appear in both generated harness files in that tool's native MCP shape; unflagging and saving removes it from both.
- **SC-004**: `npx tsc --noEmit` / `npm run lint` pass for all changed files.

## Assumptions & Dependencies

- Extracted from `026-multiuser-usability` (User Story 10, FR-020–022, the "Harness credential set" key entity, and its Phase E tasks), which is now scoped to the Docker/multi-user paradigm and cross-references this spec instead of owning this content. The mechanism itself has no Docker/Bastion dependency.
- Depends on `005-self-modification` for how/when the Dev Harness is spawned (`claude-runner.ts`); this spec only concerns *how it authenticates and is configured*, not when it runs.
- Depends on `030-settings-mcp-servers` for the MCP server entity and its `includeInDevHarness` flag; this spec is solely responsible for what happens when that flag is read.
- Storing harness credentials/provider secrets at rest is acceptable for self-hosted/trusted-operator installs (same threat model as `024-docker-multiuser`); they are treated as secrets in transit and in the API regardless of deployment mode.

## Notes

- This spec, `030-settings-mcp-servers`, and the residual `026-multiuser-usability` reference each other rather than duplicating content; a **larger refactoring pass** across the spec store (renumbering, resolving `026`/`027` duplicate numbers, filling `overview.md`'s gaps) is acknowledged as needed but explicitly out of scope here.
