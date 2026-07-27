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

### Session 2026-07-27 (UI redesign, after initial implementation)

- Q: The first implementation stacked every field (mode, model, provider, credentials) in one long vertical list, with **model appearing before the auth method that determines whether a model selection even makes sense** — user feedback: "very unintuitive." How should it actually be organized? → A: **Pick the harness first, then one row per CLI, ordered top-to-bottom by dependency.** A single top-level control chooses **which coding agent** runs development tasks — **Claude Code** or **OpenCode** — replacing the old 5-way `transport` enum (which conflated "which CLI" with "which transport" with "which MCP remote"). Below it, two clearly-delineated panels ("rows"), one per CLI, each internally ordered **auth method → auth method's own fields → model** (you cannot sensibly pick a model before you've picked how the CLI authenticates). Only the row matching the top-level selection is enabled/editable; the other is visible (so the full shape of the settings is always apparent) but disabled/dimmed — this also visually reinforces that the two CLIs' settings are independent and don't interfere with each other.
- Q: Where does the credential-file paste box (existing write-only textarea) fit into "auth method"? → A: **It's one of the auth method options, not a separate always-shown section.** Each CLI's auth method list now reads, top to bottom: **credential file** (paste `~/.claude` material / OpenCode `auth.json` — the original mechanism, FR-001/002/003) | **API key** (Claude only) | **AWS Bedrock** (Claude only) | **Google Vertex AI** (Claude only) | **named provider** (OpenCode only). Selecting "credential file" reveals the paste box; selecting any other option reveals that method's own fields instead. This is a rename, not new behavior: the old `claudeProviderMode`/`opencodeProviderMode` value `"default"` (which meant "use the credential file, unchanged") becomes the explicit `"credential-file"` / `"auth-file"` option in the same list, so the credential paste box is no longer visually or conceptually separated from the other auth choices.
- Q: Where do the MCP-mode options (stdio/http/sse — connecting to an already-running remote Claude Code MCP harness instead of spawning `claude` locally) fit? → A: **Nested inside the Claude Code row as a "run mode" sub-choice, above the auth-method list** (since auth/provider/model settings are meaningless once you're pointing at an already-authenticated remote harness instead of spawning `claude` locally). OpenCode has no such sub-choice — it is always a local headless spawn (unchanged, matches the existing constraint that OpenCode isn't exposed over the MCP `Agent` path). Claude row order becomes: **Run mode** (Local CLI, default | MCP stdio | MCP HTTP | MCP SSE) → *(Local CLI only, from here down)* **Auth method** → its fields → **Model**.
- Q: Does each CLI keep its own independent model override now that both rows are always present? → A: **Yes.** `model` (a single shared field) is split into `claudeModel`/`opencodeModel` so switching the top-level harness selection never discards the other CLI's remembered model choice.
- Q: How is this reconciled with existing stored config (the old `transport`/`model`/`claudeProviderMode`/`opencodeProviderMode` fields from the first implementation, which shipped only hours earlier)? → A: **One-time, read-time derivation, no migration script.** `load()` derives `harness`/`claudeRunMode`/`claudeAuthMethod`/`opencodeAuthMethod`/`claudeModel`/`opencodeModel` from the legacy fields when the new ones are absent (`transport: "opencode"` → `harness: "opencode"`; `transport` ∈ {stdio,http,sse} → `harness: "claude"`, `claudeRunMode: <that value>`; otherwise `harness: "claude"`, `claudeRunMode: "cli"`; old `"default"` provider mode → new `"credential-file"`/`"auth-file"`). Since this feature has no real installs yet, this is a courtesy for anyone who saved settings during initial development, not a hard requirement — but costs little to include and is good practice regardless.

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

### User Story 4 — A clear, dependency-ordered settings layout (Priority: P1)

A user opens Settings → Dev Harness and can immediately tell which coding agent is active, see both CLIs' full configuration shape without hunting through a single long vertical list, and configure each one in the order its own fields actually depend on (you can't sensibly pick a model before you've picked how the CLI authenticates).

**Why this priority**: The first shipped layout stacked mode, model, provider, and credentials in one undifferentiated column with model appearing *before* the auth method that determines whether it's even applicable — reported back as "very unintuitive." Getting the structure right is foundational to every other user story in this spec being usable at all.

**Independent Test**: Open Settings → Dev Harness with no prior configuration; confirm a single top-level "Claude Code / OpenCode" choice is the first thing shown, exactly one of the two rows below it is enabled at a time, and within the enabled row every field appears in dependency order (run mode, if applicable → auth method → that method's fields → model).

**Acceptance Scenarios**:

1. **Given** the Dev Harness settings, **When** the tab loads, **Then** a single top-level control lets the user choose **Claude Code** or **OpenCode** as the active harness.
2. **Given** Claude Code is the active harness, **When** the tab renders, **Then** the Claude Code row is enabled (editable) and the OpenCode row is visibly present but disabled; selecting OpenCode instead flips which row is enabled without discarding either row's saved settings.
3. **Given** the Claude Code row, **When** it renders, **Then** fields appear in this order: **Run mode** (Local CLI, default | MCP stdio | MCP HTTP | MCP SSE); then, only for Local CLI, **Auth method** (Credential file | API key | AWS Bedrock | Google Vertex AI); then that method's own fields (e.g. the credential-file paste box, or the API key + base URL); then **Model**.
4. **Given** the OpenCode row, **When** it renders, **Then** fields appear in this order: **Auth method** (Credential file | Named provider); then that method's own fields; then **Model**.
5. **Given** settings saved under the pre-redesign layout (`transport`/`model`/`claudeProviderMode`/`opencodeProviderMode`), **When** the tab loads, **Then** they are read into the new fields without data loss (no re-configuration required), per the read-time derivation in Clarifications.

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
- **FR-008**: The Dev Harness settings UI MUST expose one top-level control — **`harness`**: `claude` | `opencode` — choosing which coding agent runs development tasks. This supersedes the old 5-way `transport` field as the primary axis; it MUST be the first control shown.
- **FR-009**: Below the top-level control, the UI MUST render exactly two panels, one per CLI, at all times (not conditionally hidden) — **only** the panel matching the current `harness` selection MUST be enabled/editable; the other MUST be visibly present but disabled, so the full configuration surface is always apparent and the two CLIs' settings are visibly independent.
- **FR-010**: The Claude Code panel MUST expose a **`claudeRunMode`**: `cli` (default) | `stdio` | `http` | `sse`, ABOVE its auth-method controls. `stdio`/`http`/`sse` connect to an already-running remote Claude Code MCP harness instead of spawning `claude` locally, so the auth-method/model controls below (FR-011/FR-004/model) MUST only render when `claudeRunMode` is `cli`; `stdio` instead shows the existing MCP stdio command field, `http`/`sse` the existing MCP harness URL field. OpenCode has no equivalent run-mode choice (always a local headless spawn).
- **FR-011**: Each CLI's per-CLI auth mechanism (credential-file paste from FR-001–003, and the provider modes from FR-004) MUST be presented as **one unified, ordered "auth method" choice** per row, not as a separately-positioned credential section plus a separately-positioned provider selector: Claude Code's auth-method options are, in order, **Credential file** (the FR-001–003 paste box) | **API key** | **AWS Bedrock** | **Google Vertex AI**; OpenCode's are **Credential file** (`auth.json` paste box) | **Named provider**. Selecting an auth method reveals only that method's own fields (the paste box, or the relevant provider fields) — never more than one method's fields at once. The internally-stored value for "credential file" (`"credential-file"` for Claude, `"auth-file"` for OpenCode) is what FR-005's "provider is the default/credential-file case → omit provider-specific fields" condition keys off.
- **FR-012**: Each CLI's row MUST end with its own **Model** field (`claudeModel` for the Claude row when `claudeRunMode` is `cli`, `opencodeModel` for the OpenCode row), positioned after the auth-method fields — a model choice is meaningless before authentication is configured, so it MUST NOT be positioned earlier. Each CLI retains its own model independently of which `harness` is currently active (switching the top-level selection MUST NOT discard the inactive CLI's remembered model).
- **FR-013**: On load, if the new fields (`harness`, `claudeRunMode`, `claudeAuthMethod`, `opencodeAuthMethod`, `claudeModel`, `opencodeModel`) are absent but the pre-redesign fields (`transport`, `model`, `claudeProviderMode`, `opencodeProviderMode`) are present, BOS MUST derive the new fields from the old ones (per the read-time derivation in Clarifications) rather than discarding a user's existing configuration. This is a read-time derivation only — no migration script, no rewrite of stored data until the next save.

### Key Entities

- **Harness selection** — the top-level `harness: "claude" | "opencode"` choice that determines which of the two per-CLI rows is enabled.
- **Harness credential set** — Claude (`~/.claude` material) + OpenCode (`auth.json`) written into a dedicated `HOME`; secret, write-only in the API. Selected via each row's auth method being `"credential-file"`/`"auth-file"`.
- **Harness run mode** (Claude only) — `claudeRunMode: "cli" | "stdio" | "http" | "sse"`, gating whether the auth-method/model controls apply at all (only for `"cli"`) or the row instead shows an MCP command/URL field.
- **Harness auth method** — per-CLI, ordered choice unifying the credential-file paste option with the provider options (Claude: credential-file/api-key/bedrock/vertex; OpenCode: credential-file/named-provider), materialized into the harness's own generated `~/.claude/settings.json` / `~/.config/opencode/opencode.json` when non-credential-file.
- **Harness MCP inclusion** — the generation step that reads every `includeInDevHarness`-flagged MCP server (`030-settings-mcp-servers`'s entity) and folds it into `~/.claude.json` (`mcpServers`) and `opencode.json` (`mcp`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: With Claude/OpenCode credentials configured, a headless harness run authenticates and completes without any interactive login.
- **SC-002**: Selecting a non-default provider for Claude or OpenCode CLI and saving produces a correctly-shaped generated config file, and a subsequent CLI run picks it up (verified via the existing "Test" action).
- **SC-003**: Flagging an MCP server "Include in Dev Harness" (`030-settings-mcp-servers`) and saving makes it appear in both generated harness files in that tool's native MCP shape; unflagging and saving removes it from both.
- **SC-004**: `npx tsc --noEmit` / `npm run lint` pass for all changed files.
- **SC-005**: Loading the Dev Harness tab shows the top-level Claude/OpenCode choice first, exactly one of the two rows enabled at a time, and every field within the enabled row in dependency order (run mode → auth method → that method's fields → model).
- **SC-006**: Settings saved under the pre-redesign field names load correctly into the new UI with no data loss.

## Assumptions & Dependencies

- Extracted from `026-multiuser-usability` (User Story 10, FR-020–022, the "Harness credential set" key entity, and its Phase E tasks), which is now scoped to the Docker/multi-user paradigm and cross-references this spec instead of owning this content. The mechanism itself has no Docker/Bastion dependency.
- Depends on `005-self-modification` for how/when the Dev Harness is spawned (`claude-runner.ts`); this spec only concerns *how it authenticates and is configured*, not when it runs.
- Depends on `030-settings-mcp-servers` for the MCP server entity and its `includeInDevHarness` flag; this spec is solely responsible for what happens when that flag is read.
- Storing harness credentials/provider secrets at rest is acceptable for self-hosted/trusted-operator installs (same threat model as `024-docker-multiuser`); they are treated as secrets in transit and in the API regardless of deployment mode.
- The pre-redesign field names (`transport`, `model`, `claudeProviderMode`, `opencodeProviderMode`) remain valid on-disk shapes indefinitely via the read-time derivation (FR-013) — there is no deadline by which they must be migrated, since every write from the redesigned UI naturally uses the new field names going forward.

## Notes

- This spec, `030-settings-mcp-servers`, and the residual `026-multiuser-usability` reference each other rather than duplicating content; a **larger refactoring pass** across the spec store (renumbering, resolving `026`/`027` duplicate numbers, filling `overview.md`'s gaps) is acknowledged as needed but explicitly out of scope here.
