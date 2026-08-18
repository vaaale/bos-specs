# Tasks: External Repository Integration

**Spec**: user-specs/001-external-repo-integration/spec.md
**Plan**: user-specs/001-external-repo-integration/plan.md
**Branch**: bos/gitfs-external-repository-support

## Execution Order

```
Phase 0 (Foundation)
  ├── T001 Logging infrastructure
  ├── T002 git-lock serialization
  ├── T003 Core git operations
  └── T004 Auth resolution
       │
       ├─► Phase 1 (Remotes) ──► Phase 2 (OAuth) ◄── can run in parallel after Phase 0
       │         │                                      │
       │         └──────► Phase 3 (Mounts)               │
       │                                                   │
       └───────────────────► Phase 4 (Sync + Push) ◄──────┘
                                    │
                              Phase 5 (E2E + Polish)
```

---

## Phase 0: Foundation

**Goal**: Core infrastructure that ALL subsequent phases depend on. No user-facing functionality yet.

**Checkpoint**: All 4 unit test suites pass. `git-lock` correctly serializes concurrent operations.

---

### T001 [P] [F0] Structured logging infrastructure

Create the structured logging module for all git operations. This is a standalone utility — no business logic, just format + output.

**File**: `src/lib/gitops/logging.ts`

**What to implement**:
- `GitLogger` class with methods: `debug()`, `info()`, `warn()`, `error()`, `critical()`
- Each method accepts a structured object: `{ op: string, repoPath?: string, remote?: string, user?: string, durationMs?: number, success?: boolean, error?: { code: string, message: string, suggestion?: string } }`
- Log rotation: 7-day retention, max 10MB per file, compressed archives
- URL sanitization: strip embedded credentials (`user:pass@`) from all logged URLs
- Never log: SSH keys, tokens, passphrases (reject if any of these appear in structured data — throw or redact)
- File output: `data/logs/git-ops.log` (JSON lines format, one entry per line)
- Console output: only `warn` and above (configurable via env `GITOPS_LOG_LEVEL`)

**Existing patterns to follow**:
- Check `src/lib/integrations/webhooks/handler.ts` for BOS logging conventions
- Check if there's an existing logger in `src/lib/` — if so, extend it rather than creating a new one

**Acceptance criteria**:
- [ ] `GitLogger.info({ op: "test", message: "hello" })` writes a valid JSON line to `data/logs/git-ops.log`
- [ ] URL with embedded credentials is sanitized: `https://user:pass@github.com/x/y.git` → `https://****@github.com/x/y.git`
- [ ] Passing a token or SSH key to any log method triggers redaction (not just no-op)
- [ ] Log rotation works: after 7 days, old logs are compressed and removed
- [ ] All 5 log levels produce correctly formatted entries with timestamps
- [ ] Unit tests cover sanitization, redaction, rotation, and all log levels

---

### T002 [P] [F0] git-lock serialization mechanism

Create the `git-lock` system that serializes ALL git operations across all remotes and mounts. This prevents concurrent git corruption (the #1 risk in the risk register).

**File**: `src/lib/gitops/lock.ts`

**What to implement**:
- `GitLock` class with:
  - `acquire(repoPath: string, operation: string): Promise<ReleaseFn>` — acquires a lock for a specific repo path; operations on the same repo are serialized; operations on different repos can run in parallel
  - `withLock<T>(repoPath: string, operation: string, fn: (release: ReleaseFn) => Promise<T>): Promise<T>` — convenience method that acquires and releases
  - `ReleaseFn` — callable function that releases the lock
- Lock timeout: 30 seconds. If a lock is held for > 30s, force-release with a `warn` log entry (not `error` — the holding operation may still be in progress)
- Lock ordering: if two operations on different repos both need a lock on a shared resource (e.g., bare cache), acquire in alphabetical order to prevent deadlocks
- Lock state: in-memory map of `{ repoPath: { holder: string, acquiredAt: number } }`
- Use `fs.mkdirSync` + `fs.writeFileSync` to a `.git-lock` file in each repo's working directory as the actual lock mechanism (file-based, survives process restart)

**Critical details**:
- Lock files must be created in the repo's working directory (`.git-lock`) — NOT in `data/config/` or `data/.git-cache/`
- Lock files contain: `{ holder: "operation-id", acquiredAt: timestamp, timeout: 30000 }`
- On `acquire()`, check if lock file exists and is stale (> 30s old). If stale, log `warn` and overwrite
- On `release()`, delete the lock file

**Acceptance criteria**:
- [ ] Two concurrent `withLock(".", "push", ...)` calls execute sequentially (second waits for first)
- [ ] Two concurrent `withLock(".", "push", ...)` and `withLock("apps", "fetch", ...)` calls execute in parallel (different repos)
- [ ] Lock timeout fires after 30s with `warn` log entry, then force-releases
- [ ] Lock file is created and deleted correctly
- [ ] Unit tests: sequential execution, parallel execution for different repos, timeout, stale lock handling

---

### T003 [P] [F0] Core git operations module

Create `git-ops.ts` — thin wrappers around the git CLI for all operations used by this feature. This module does NOT handle auth or locking directly; callers are responsible for acquiring locks and resolving auth before calling these functions.

**File**: `src/lib/gitops/git-ops.ts`

**What to implement**:

```typescript
// Clone a repository (full history, not shallow)
function cloneRepo(
  url: string,
  targetPath: string,
  branch?: string,
  auth?: GitAuth
): Promise<void>

// Fetch updates from a remote
function fetchRepo(
  repoPath: string,
  remote?: string,
  branch?: string,
  auth?: GitAuth
): Promise<{ ahead: number; behind: number }>

// Push to a remote
function pushRepo(
  repoPath: string,
  remote: string,
  branch: string,
  auth?: GitAuth
): Promise<void>

// List branches on a remote
function listRemoteBranches(
  url: string,
  auth?: GitAuth
): Promise<string[]>

// Add a remote to a repository
function addRemote(
  repoPath: string,
  name: string,
  url: string
): Promise<void>

// Remove a remote from a repository
function removeRemote(
  repoPath: string,
  name: string
): Promise<void>

// List remotes in a repository
function listRemotes(
  repoPath: string
): Promise<{ name: string; url: string }[]>

// Get the default branch of a remote
function getDefaultBranch(
  url: string,
  auth?: GitAuth
): Promise<string>

// Get current branch in a repository
function getCurrentBranch(
  repoPath: string
): Promise<string>

// Test connection to a remote (ls-remote)
function testConnection(
  url: string,
  auth?: GitAuth
): Promise<{ ok: boolean; branches: string[]; error?: string }>

// Merge a remote branch
function mergeBranch(
  repoPath: string,
  remote: string,
  branch: string,
  strategy: "merge" | "merge-squash" | "commit"
): Promise<{ status: "success" | "conflict"; commitHash?: string }>

// Switch branch in a repository
function switchBranch(
  repoPath: string,
  branch: string
): Promise<void>

// Stash and pop changes
function stashChanges(repoPath: string): Promise<void>
function popStash(repoPath: string): Promise<void>

// Check for uncommitted changes
function hasUncommittedChanges(repoPath: string): Promise<boolean>

// Check if directory contains a .git directory
function hasGitDir(repoPath: string): Promise<boolean>

// Get bare cache path for a URL
function getBareCachePath(url: string): string

// Force-update a bare cache ref
function updateBareCache(
  url: string,
  branch: string,
  auth?: GitAuth
): Promise<void>

// Scan directory for symlinks that escape
function scanSymlinkEscapes(dirPath: string): Promise<string[]>
```

**Implementation details**:
- Use `child_process.spawn` (not `exec`) for all git commands — `exec` can't stream output and is a security risk
- All git commands run with `env: { ...process.env, ...authEnv(auth) }` where `authEnv` sets `GIT_TERMINAL_PROMPT=0` and configures SSH if needed
- For SSH auth: set `GIT_SSH_COMMAND` to point to a temp script that uses the stored key
- For token auth: embed credentials in URL (`https://oauth2:TOKEN@github.com/...`)
- For OAuth: use stored access token in URL (same as token auth)
- ALL functions must log at `debug` level (operation name, args) and `info` level (success) or `error` level (failure)
- Return structured errors with `code` and `message` (e.g., `{ code: "GIT_AUTH_FAILURE", message: "SSH key rejected", suggestion: "Check SSH key in Settings" }`)

**Acceptance criteria**:
- [ ] `cloneRepo("https://github.com/x/y.git", "/tmp/test", "main")` clones the repo with full history
- [ ] `fetchRepo(".", "origin")` returns correct ahead/behind counts
- [ ] `pushRepo(".", "origin", "main")` pushes successfully
- [ ] `listRemoteBranches("https://github.com/x/y.git")` returns branch names
- [ ] `mergeBranch(".", "origin", "feature", "merge-squash")` squashes and commits
- [ ] `mergeBranch(".", "origin", "feature", "commit")` stashes local, commits remote, pops stash
- [ ] `scanSymlinkEscapes("/path/to/repo")` returns list of symlinks pointing outside repo
- [ ] All functions log at correct levels
- [ ] Unit tests for all functions (mock `child_process.spawn`)

---

### T004 [P] [F0] Auth resolution module

Create `auth.ts` — resolves authentication credentials from SecretsStore for git operations. This module is responsible for reading tokens, keys, and passphrases from encrypted storage and returning them in a format usable by `git-ops.ts`.

**File**: `src/lib/gitops/auth.ts`

**What to implement**:

```typescript
type AuthType = "oauth" | "token" | "ssh"

interface GitAuth {
  type: AuthType
  // For OAuth:
  accessToken?: string
  // For token:
  pat?: string
  // For SSH:
  sshKeyPath?: string  // path to decrypted key file (temporary)
  sshKeyData?: string  // raw key data (if not stored as file)
}

// Resolve auth for a remote from SecretsStore
function resolveAuth(
  remoteName: string,
  authType: AuthType
): Promise<GitAuth | null>

// Apply auth to a git URL (returns URL with embedded credentials)
function applyAuthToUrl(
  url: string,
  auth: GitAuth
): string

// Configure SSH key for git operations (writes temp key file, returns GIT_SSH_COMMAND)
function configureSshAuth(
  sshKeyData: string,
  passphrase?: string
): { sshKeyPath: string; cleanup: () => void }

// Validate credentials by doing a test connection
function validateAuth(
  url: string,
  auth: GitAuth
): Promise<{ ok: boolean; error?: string }>

// Check if OAuth token is expiring soon (< 24h)
function isTokenExpiringSoon(
  tokenExpiry: number  // unix timestamp
): boolean

// Refresh OAuth token if needed (calls provider's refresh endpoint)
async function maybeRefreshOAuth(
  integrationId: string,
  remoteName: string
): Promise<{ refreshed: boolean; newToken?: string; error?: string }>
```

**Implementation details**:
- Read from SecretsStore using `getSecretsStore().get(id, key)` pattern (see `src/lib/integrations/secrets/store.ts`)
- Secret key format: `git_remote:<remoteName>:<authType>` (e.g., `git_remote:github-work:oauth`)
- For SSH: write key to `data/.ssh-keys/<remoteName>.key` with `0700` permissions, use `finally` block to ensure cleanup
- `applyAuthToUrl` must handle HTTPS URLs: `https://oauth2:TOKEN@github.com/user/repo.git`
- `applyAuthToUrl` must reject `git://` protocol (return null)
- `validateAuth` uses `git ls-remote <url>` with the auth to test connectivity
- `maybeRefreshOAuth` calls the existing OAuth manager (`src/lib/integrations/oauth/manager.ts`) to refresh tokens
- Log at `debug` level for all operations, `error` for failures

**Acceptance criteria**:
- [ ] `resolveAuth("github-work", "oauth")` returns `{ type: "oauth", accessToken: "..." }` from SecretsStore
- [ ] `resolveAuth("github-work", "ssh")` returns `{ type: "ssh", sshKeyData: "..." }` from SecretsStore
- [ ] `applyAuthToUrl("https://github.com/x/y.git", { type: "oauth", accessToken: "tok" })` returns URL with embedded credentials
- [ ] `applyAuthToUrl("git://github.com/x/y.git", ...)` returns `null`
- [ ] `configureSshAuth("-----BEGIN OPENSSH...")` creates temp file with 0700 permissions and returns cleanup function
- [ ] `validateAuth("https://github.com/x/y.git", auth)` returns `{ ok: true }` for valid credentials
- [ ] `maybeRefreshOAuth("github-git", "github-work")` refreshes token when < 24h remaining
- [ ] Unit tests for all functions (mock SecretsStore)

---

## Phase 1: Git Remotes

**Goal**: Users can register external remotes, authenticate, and push to them. This is the MVP increment — users get real value after this phase.

**Independent Test**: User registers a GitHub remote on BOS source, authenticates via token, and pushes to it.

**Checkpoint**: `git_add_remote`, `git_remove_remote`, `git_list_remotes`, `git_push`, `git_push_all_remotes`, `git_fetch` tools all work end-to-end.

---

### T005 [F1] Remote metadata config file

Create the `git-remotes.json` config file and its read/write utilities. This file stores remote metadata (not auth — auth stays in SecretsStore).

**File**: `src/lib/gitops/remote-config.ts` (new)
**Data file**: `data/config/git-remotes.json` (new)

**What to implement**:

```typescript
interface GitRemoteConfig {
  name: string
  url: string
  provider: "github" | "gitlab" | "generic"
  autoPush: boolean
  defaultBranch?: string  // discovered via git ls-remote --symref
  remoteBranch?: string   // optional: branch mapping for push target
  lastFetched?: string
  lastPushed?: string
  oauthTokenExpiresAt?: number
  createdAt: string
  updatedAt: string
}

// Read all remote configs
function readRemoteConfigs(): GitRemoteConfig[]

// Write all remote configs (atomic write)
function writeRemoteConfigs(configs: GitRemoteConfig[]): void

// Add a remote config
function addRemoteConfig(config: Omit<GitRemoteConfig, "createdAt" | "updatedAt">): GitRemoteConfig

// Update a remote config
function updateRemoteConfig(name: string, patch: Partial<GitRemoteConfig>): GitRemoteConfig

// Remove a remote config
function removeRemoteConfig(name: string): boolean

// Check if a remote name already exists (returns auto-renamed name if so)
function getUniqueRemoteName(name: string): string
```

**Implementation details**:
- File path: `data/config/git-remotes.json`
- Atomic write: write to temp file, then `fs.rename` (prevents corruption on crash)
- Auto-rename: if `name` exists, try `name-2`, `name-3`, etc.
- `getUniqueRemoteName` returns the name to use (original or auto-renamed)
- Log at `info` for add/update/remove, `debug` for read/write
- File permissions: 0600 (owner read/write only — contains URLs)

**Acceptance criteria**:
- [ ] `addRemoteConfig({ name: "github-work", url: "https://...", provider: "github", autoPush: true })` creates a config entry
- [ ] `getUniqueRemoteName("origin")` returns `"origin"` if it doesn't exist, `"origin-2"` if it does
- [ ] `writeRemoteConfigs` uses atomic write (temp file + rename)
- [ ] File is created at `data/config/git-remotes.json` with 0600 permissions
- [ ] Unit tests for CRUD operations, auto-rename, atomic write

---

### T006 [P] [F1] Agent tools: remote CRUD and list operations

Create the agent tools for registering, removing, and listing git remotes. These tools call `git-ops.ts` for the actual git operations and `remote-config.ts` for metadata persistence.

**File**: `src/lib/assistant/tools/server/git-remotes.ts` (new)

**What to implement**:

```typescript
// git_add_remote
tool: git_add_remote
description: "Register a new git remote with authentication"
params: {
  repoPath: string,        // e.g. "." for BOS source, "apps" for apps content
  name: string,
  url: string,
  provider: "github" | "gitlab" | "generic",
  authType: "oauth" | "token" | "ssh",
  // For token auth:
  token?: string,
  // For SSH auth:
  sshKey?: string,         // NEVER via agent tool — collected via secure UI
}
returns: {
  name: string,
  url: string,
  status: "success" | "error",
  message?: string,
  uniqueName?: string  // if auto-renamed
}

// git_remove_remote
tool: git_remove_remote
description: "Remove a git remote"
params: { repoPath: string, name: string }
returns: { status: "success" | "error", message?: string }

// git_list_remotes
tool: git_list_remotes
description: "List all git remotes for a repository"
params: { repoPath: string }
returns: { remotes: { name: string; url: string; provider?: string; autoPush?: boolean }[] }

// git_list_branches
tool: git_list_branches
description: "List available branches on a remote"
params: { url: string, authType: "oauth" | "token" | "ssh", token?: string }
returns: { branches: string[] }
```

**Implementation details**:
- Each tool function must:
  1. Validate inputs (reject `git://` URLs, validate VFS paths if applicable)
  2. Acquire `git-lock` for the repo path
  3. Resolve auth via `auth.ts` (if credentials are provided in the tool call, store them in SecretsStore first)
  4. Execute the git operation via `git-ops.ts`
  5. Update `remote-config.ts` metadata
  6. Release lock
  7. Log at appropriate level
- Tool manifest registration: register in `src/lib/assistant/tools/server/index.ts`
- Capability group: `git-remotes`
- Return structured errors with `code`, `message`, `suggestion`

**Acceptance criteria**:
- [ ] `git_add_remote` with a valid URL and token creates a remote and stores the token in SecretsStore
- [ ] `git_add_remote` with a duplicate name auto-renames (e.g., "origin" → "origin-2")
- [ ] `git_add_remote` with `git://` URL returns error
- [ ] `git_remove_remote` removes from both git config and remote-config.json
- [ ] `git_list_remotes` returns all remotes with their metadata
- [ ] `git_list_branches` returns branches from the remote
- [ ] Tools are registered and callable by the assistant
- [ ] Unit tests for all tool functions (mock git-ops, remote-config, SecretsStore)

---

### T007 [P] [F1] Agent tools: push and fetch operations

Create the agent tools for pushing to remotes and fetching updates. This is where multi-remote push and per-remote auto-push logic lives.

**File**: `src/lib/assistant/tools/server/git-push.ts` (new)
**File**: `src/lib/assistant/tools/server/git-fetch.ts` (new)

**What to implement**:

```typescript
// git_push
tool: git_push
description: "Push current branch to a specific remote"
params: { repoPath: string, remote: string, branch?: string }
returns: { status: "success" | "failed", pushed: boolean, error?: { code, message, suggestion? } }

// git_push_all_remotes
tool: git_push_all_remotes
description: "Push current branch to all registered remotes (or selected ones)"
params: {
  repoPath: string,
  remoteNames?: string[],  // if omitted, push to all
  branch?: string
}
returns: { results: { remoteName: string; status: "success" | "failed"; error?: { code, message } }[] }

// git_fetch
tool: git_fetch
description: "Fetch updates from a remote"
params: {
  repoPath: string,
  remote?: string,  // if omitted, fetch all remotes
  branch?: string
}
returns: {
  status: "success" | "failed",
  updates: { newBranches: string[]; updatedBranches: string[]; deletedBranches: string[] },
  error?: string
}
```

**Implementation details**:
- `git_push`:
  1. Acquire `git-lock` for repo path
  2. Resolve auth for remote
  3. Call `pushRepo()` from `git-ops.ts`
  4. Update `lastPushed` in `remote-config.json`
  5. Release lock
- `git_push_all_remotes`:
  1. Acquire `git-lock` for repo path
  2. For each remote (sequentially): resolve auth → push → log result
  3. Return summary of all results
  4. Release lock
  5. **Critical**: This is the atomic sequence that must not collide with Supervisor promote. The lock ensures serialization.
- `git_fetch`:
  1. Acquire `git-lock` for repo path
  2. Call `fetchRepo()` from `git-ops.ts`
  3. Update `lastFetched` in `remote-config.json`
  4. Release lock
- Auto-push integration: the Supervisor (Phase 4) will call `git_push` for each auto-push-enabled remote after a promote. This tool doesn't need to trigger auto-push itself.

**Acceptance criteria**:
- [ ] `git_push` pushes to a single remote successfully
- [ ] `git_push_all_remotes` pushes to all remotes and returns a summary
- [ ] `git_push` with an invalid remote returns a clear error
- [ ] `git_fetch` returns correct update counts
- [ ] Multi-remote push is serialized (two concurrent `git_push_all_remotes` calls execute sequentially)
- [ ] Unit tests for all tool functions

---

### T008 [F1] Settings UI: Git Remotes section

Create the Settings UI for managing git remotes. This goes in the Versions tab.

**File**: `src/components/apps/settings/versions/GitRemotesTab.tsx` (new component, integrated into existing VersionsTab)

**What to implement**:
- A "Git Remotes" section in the Versions settings tab (alongside the existing version control controls)
- List of registered remotes showing: name, URL, provider icon, auto-push toggle, last push/fetch time, status
- "Add Remote" button → opens a modal with: name, URL, provider dropdown, auth type selection
- For each remote: "Push" button, "Fetch" button, "Test Connection" button, "Edit" button, "Remove" button
- Auto-push toggle: visual toggle switch (on/off) for each remote
- Status indicators: ● Connected / ○ Not connected / ⚠ Error

**Integration with existing code**:
- The existing VersionsTab is at `src/components/apps/settings/versions/VersionsTab.tsx` (or similar path — check actual location)
- Add a new section/tab within the existing VersionsTab component
- Do NOT create a separate top-level settings page — this is a subsection of Versions

**Acceptance criteria**:
- [ ] Git Remotes section appears in Settings → Versions tab
- [ ] Registered remotes are listed with correct metadata
- [ ] "Add Remote" modal opens and submits correctly
- [ ] Auto-push toggle switches between on/off
- [ ] "Push" button triggers a push to the selected remote
- [ ] "Test Connection" button runs a connection test and shows result
- [ ] "Remove" button removes the remote (with confirmation dialog)
- [ ] Provider icons display correctly (GitHub, GitLab, Generic)
- [ ] Status indicators update after push/fetch operations

---

## Phase 2: OAuth Providers

**Goal**: OAuth authentication for GitHub and GitLab. This phase enables the most common authentication flow.

**Independent Test**: User connects their GitHub account via OAuth, registers a remote using that connection, and pushes successfully.

**Checkpoint**: OAuth flow works end-to-end for both GitHub and GitLab. Tokens are stored and refreshed correctly.

---

### T009 [P] [F2] GitHub git OAuth integration manifest

Create the OAuth integration manifest for GitHub git operations. This reuses the existing integrations framework.

**File**: `src/lib/integrations/oauth/github-git.ts` (new)

**What to implement**:
- Integration manifest following the pattern in `src/lib/integrations/services/telegram/manifest.ts`
- Integration ID: `github-git` (as per AD-003)
- OAuth configuration:
  - `client_id`: loaded from SecretsStore under `github-git:client_id`
  - `client_secret`: loaded from SecretsStore under `github-git:client_secret`
  - `scopes: ["repo"]`
  - `authorizeUrl: "https://github.com/login/oauth/authorize"`
  - `tokenUrl: "https://github.com/login/oauth/access_token"`
  - `callbackUrl: "/api/integrations/oauth/callback"`
- Token storage: `getSecretsStore().get/set("github-git", "tokens", ...)`
- Integration state: `data/integrations/github-git/state.json` (reuse existing state store pattern from `src/lib/integrations/state/store.ts`)
- Register the integration in `src/lib/integrations/index.ts` (add to the integrations registry)

**Implementation details**:
- Follow the exact pattern of existing integrations (e.g., Telegram bot manifest)
- The OAuth manager (`src/lib/integrations/oauth/manager.ts`) handles the heavy lifting — this file just provides the manifest/config
- Client ID/Secret are pre-configured by BOS (not user-provided) and stored in SecretsStore during BOS setup
- `startOAuth()` and `handleCallback()` are provided by the OAuth manager — this file just defines the config

**Acceptance criteria**:
- [ ] Integration manifest follows the same pattern as existing integrations
- [ ] Integration ID is `github-git`
- [ ] Scopes are `["repo"]`
- [ ] OAuth URLs are correct (GitHub authorize + token endpoints)
- [ ] Integration is registered in the integrations registry
- [ ] Unit tests for manifest structure

---

### T010 [P] [F2] GitLab git OAuth integration manifest

Create the OAuth integration manifest for GitLab git operations. Same pattern as T009.

**File**: `src/lib/integrations/oauth/gitlab-git.ts` (new)

**What to implement**:
- Integration ID: `gitlab-git`
- OAuth configuration:
  - `scopes: ["api"]`
  - `authorizeUrl: "https://gitlab.com/oauth/authorize"`
  - `tokenUrl: "https://gitlab.com/oauth/token"`
  - Callback URL: same as GitHub (`/api/integrations/oauth/callback`)
- Token storage: `getSecretsStore().get/set("gitlab-git", "tokens", ...)`
- State store: `data/integrations/gitlab-git/state.json`
- Register in `src/lib/integrations/index.ts`

**Acceptance criteria**:
- [ ] Integration manifest follows the same pattern as T009
- [ ] Integration ID is `gitlab-git`
- [ ] Scopes are `["api"]`
- [ ] OAuth URLs are correct (GitLab authorize + token endpoints)
- [ ] Integration is registered in the integrations registry
- [ ] Unit tests for manifest structure

---

### T011 [F2] OAuth callback handling and token refresh

Wire up the OAuth callback route and token refresh logic. The callback route already exists for other integrations — extend it to handle `github-git` and `gitlab-git`.

**File**: `src/app/api/integrations/oauth/callback/route.ts` (modify existing)
**File**: `src/lib/gitops/auth.ts` (extend with `maybeRefreshOAuth`)

**What to implement**:
- The existing OAuth callback route (`src/app/api/integrations/oauth/callback/route.ts`) should already handle any integration ID via the integrations registry — verify this works for `github-git` and `gitlab-git`
- If the callback route needs modification, add logic to:
  1. Look up integration manifest by ID
  2. Exchange authorization code for token
  3. Store token in SecretsStore
  4. Update integration state
  5. Redirect back to BOS with success
- Add `maybeRefreshOAuth()` to `auth.ts` (see T004) — checks if token expires within 24h, calls OAuth manager to refresh
- Log at `info` for successful refresh, `warn` for near-expiry, `error` for refresh failure

**Acceptance criteria**:
- [ ] GitHub OAuth flow: redirect to GitHub → authorize → callback → token stored → remote connected
- [ ] GitLab OAuth flow: redirect to GitLab → authorize → callback → token stored → remote connected
- [ ] Token refresh works when < 24h remaining
- [ ] Failed refresh returns clear error message
- [ ] Integration state file is updated after each OAuth operation
- [ ] E2E test for full OAuth flow (mock OAuth provider)

---

### T012 [F2] Settings UI: Git Providers section

Create the Settings UI for managing OAuth provider connections. This goes in the Integrations tab.

**File**: `src/components/apps/settings/integrations/GitProvidersTab.tsx` (new component, integrated into existing IntegrationsTab)

**What to implement**:
- A "Git Providers" section in the Settings → Integrations tab
- List of git providers (GitHub, GitLab) showing: name, connection status (● Connected / ○ Not connected), connected account (username/repo count)
- "Connect" button for each provider → redirects to OAuth authorize page
- "Disconnect" button → calls disconnect endpoint, clears tokens from SecretsStore
- "Reconnect" button (if already connected) → re-initiates OAuth flow
- Status indicators and connection metadata

**Integration with existing code**:
- Check existing IntegrationsTab for the pattern (e.g., Telegram, GSuite sections)
- Add "Git Providers" as a new section within the existing IntegrationsTab

**Acceptance criteria**:
- [ ] Git Providers section appears in Settings → Integrations tab
- [ ] GitHub and GitLab are listed with connection status
- [ ] "Connect" button redirects to OAuth authorize page
- [ ] After OAuth approval, status updates to "● Connected"
- [ ] "Disconnect" button clears tokens and updates status
- [ ] Connection metadata (username, scope) is displayed
- [ ] Matches the visual style of existing integration sections

---

## Phase 3: VFS Mounts

**Goal**: Users can clone external repos to any VFS path and work with them as regular directories.

**Independent Test**: User mounts a GitHub repo to `/Projects/my-webapp`, edits a file, pushes changes, and verifies they appear in the remote.

**Checkpoint**: Mount/unmount/fetch/push lifecycle works. Bare cache is functional. VFS integration is complete.

---

### T013 [P] [F3] VFS mount manager — core lifecycle

Create the mount manager that handles the full lifecycle of VFS-mounted repositories.

**File**: `src/lib/gitops/mounts/mount-manager.ts` (new)

**What to implement**:

```typescript
interface VfsMount {
  id: string
  vfsPath: string
  repoUrl: string
  branch: string
  authType: "oauth" | "token" | "ssh"
  status: "mounted" | "stale" | "error"
  lastFetched?: string
  lastSynced?: string
  isBareCache: boolean
  bare: boolean
  createdAt: string
  updatedAt: string
}

// Mount a repository to a VFS path
function mount(
  repoUrl: string,
  targetPath: string,
  branch: string,
  auth: GitAuth
): Promise<VfsMount>

// Unmount a repository (remove VFS path, preserve cache)
function unmount(vfsPath: string): Promise<void>

// Fetch updates for a mounted repository
function fetch(vfsPath: string, auth: GitAuth): Promise<SyncStatus>

// Push changes from a mounted repository
function push(vfsPath: string, branch: string, auth: GitAuth): Promise<void>

// Switch branch of a mounted repository
function switchBranch(vfsPath: string, branch: string, auth: GitAuth): Promise<void>

// List all mounts
function listMounts(): VfsMount[]

// Get mount status
function getMountStatus(vfsPath: string): VfsMount | null

// Validate mount path (must be within data/vfs/, not under apps/workflows)
function validateMountPath(targetPath: string): { valid: boolean; error?: string }

// Scan for symlink escapes after clone
function scanForEscapes(dirPath: string): Promise<string[]>
```

**Implementation details**:
- `validateMountPath`:
  - Must be within `data/vfs/`
  - Must NOT be under `data/vfs/apps/` or `data/vfs/workflows/`
  - Must not already contain a `.git` directory
  - Return structured error if invalid
- `mount`:
  1. Validate path
  2. Check for existing `.git` in target path
  3. `git clone --branch <branch> <url> <targetPath>` via `git-ops.ts`
  4. Scan for symlink escapes
  5. If escapes found: replace symlinks that point outside repo with empty directories (or warn and abort)
  6. Create mount entry in `data/config/vfs-mounts.json` (new config file)
  7. Register mount point in VFS (`src/os/vfs.ts`)
  8. Return mount status
- `unmount`:
  1. Remove mount entry from config
  2. Remove VFS registration
  3. Remove VFS directory (but preserve `data/.git-cache/`)
- `fetch`:
  1. Acquire `git-lock` for the mount path
  2. `git fetch origin`
  3. Compare local vs remote
  4. Update `lastFetched` and status
  5. Release lock
- `push`:
  1. Acquire `git-lock` for the mount path
  2. Check for uncommitted changes
  3. `git push origin <branch>`
  4. Update `lastSynced`
  5. Release lock
- `switchBranch`:
  1. Check for uncommitted changes
  2. If changes exist: present options (commit, discard, stash & switch)
  3. `git checkout <branch>`
- Store mount configs in `data/config/vfs-mounts.json`
- Log at `info` for mount/unmount/fetch/push events, `warn` for stale cache or escape detection

**Acceptance criteria**:
- [ ] `mount("https://github.com/x/y.git", "/data/vfs/Projects/test", "main", auth)` clones and registers the mount
- [ ] `validateMountPath("/data/vfs/Projects/test")` returns `{ valid: true }`
- [ ] `validateMountPath("/data/vfs/apps/test")` returns `{ valid: false, error: "..." }`
- [ ] `validateMountPath("/Documents/test")` returns `{ valid: false, error: "..." }`
- [ ] `unmount("/data/vfs/Projects/test")` removes VFS path but preserves cache
- [ ] `fetch` updates status and ahead/behind counts
- [ ] `push` pushes changes and updates lastSynced
- [ ] `switchBranch` handles uncommitted changes with user options
- [ ] Symlink escape scan works and aborts mount if escapes detected
- [ ] Unit tests for all functions (mock git-ops, fs, SecretsStore)

---

### T014 [P] [F3] Sync status computation

Create the sync status module that computes ahead/behind counts for remotes and mounts.

**File**: `src/lib/gitops/mounts/sync-status.ts` (new)

**What to implement**:

```typescript
interface SyncStatus {
  localBranch: string
  remoteBranch: string
  ahead: number
  behind: number
  lastFetched?: string
  status: "up_to_date" | "ahead" | "behind" | "diverged" | "no_remote"
}

// Compute sync status for a remote
function getRemoteSyncStatus(
  repoPath: string,
  remote: string,
  branch: string,
  auth: GitAuth
): Promise<SyncStatus>

// Compute sync status for a mounted repository
function getMountSyncStatus(
  vfsPath: string,
  auth: GitAuth
): Promise<SyncStatus>

// Compute sync status for all remotes
function getAllRemoteSyncStatuses(
  repoPath: string,
  auth: GitAuth
): Promise<SyncStatus[]>

// Check for uncommitted changes that would conflict with fetch
function getStashableChanges(repoPath: string): Promise<{ hasChanges: boolean; description: string }>
```

**Implementation details**:
- Use `git rev-list --left-right --count <local>...<remote>` to count ahead/behind
- Use `git status --porcelain` to check for uncommitted changes
- `status` field:
  - `up_to_date`: ahead=0, behind=0
  - `ahead`: ahead>0, behind=0
  - `behind`: ahead=0, behind>0
  - `diverged`: ahead>0, behind>0 (conflict)
  - `no_remote`: remote doesn't exist or isn't accessible
- Log at `debug` for status computation, `info` for status changes

**Acceptance criteria**:
- [ ] `getRemoteSyncStatus(".", "origin", "main", auth)` returns correct ahead/behind
- [ ] `getMountSyncStatus("/data/vfs/Projects/test", auth)` returns correct status
- [ ] `getAllRemoteSyncStatuses(".", auth)` returns status for all remotes
- [ ] `getStashableChanges(".")` detects uncommitted changes
- [ ] Unit tests for all functions

---

### T015 [F3] Agent tools: mount management

Create the agent tools for VFS mount operations.

**File**: `src/lib/assistant/tools/server/git-mount.ts` (new)

**What to implement**:

```typescript
// git_mount
tool: git_mount
description: "Clone and mount an external repository to a VFS path"
params: {
  repoUrl: string,
  targetPath: string,
  branch?: string,
  authType: "oauth" | "token" | "ssh",
  token?: string,  // optional — if not provided, uses stored auth
  sshKey?: string  // NEVER via agent tool — collected via secure UI
}
returns: {
  status: "mounted" | "error",
  path: string,
  branches?: string[],
  error?: { code, message, suggestion? }
}

// git_unmount
tool: git_unmount
description: "Unmount a VFS-mounted repository"
params: { vfsPath: string }
returns: { status: "success" | "error", message?: string }

// git_list_mounts
tool: git_list_mounts
description: "List all VFS-mounted repositories"
returns: { mounts: VfsMount[] }

// git_mount_status
tool: git_mount_status
description: "Get sync status for a mounted repository"
params: { vfsPath: string }
returns: SyncStatus
```

**Implementation details**:
- Each tool must:
  1. Validate inputs (reject paths outside `data/vfs/`, reject `git://` URLs)
  2. Acquire `git-lock` for the mount path
  3. Call the corresponding function in `mount-manager.ts`
  4. Release lock
  5. Log at appropriate level
- `git_mount` with `targetPath` outside `data/vfs/` returns error immediately (before any git operation)
- Register tools in tool manifest, capability group: `git-mounts`

**Acceptance criteria**:
- [ ] `git_mount` clones and registers a mount correctly
- [ ] `git_mount` with invalid path returns error before git operation
- [ ] `git_unmount` removes VFS path and config entry
- [ ] `git_list_mounts` returns all mounts
- [ ] `git_mount_status` returns sync status
- [ ] Tools are registered and callable
- [ ] Unit tests for all tool functions

---

### T016 [F3] Settings UI: Mount manager

Create the Settings UI for managing VFS mounts. This goes in the Versions tab alongside the Git Remotes section.

**File**: `src/components/apps/settings/versions/GitRemotesTab.tsx` (extend existing component)

**What to implement**:
- A "VFS Mounts" section in the Versions settings tab (below Git Remotes section)
- List of mounted repositories showing: VFS path, URL, branch, provider, status
- "Mount Repository" button → opens modal with: URL, VFS path, branch, auth type
- For each mount: "Fetch" button, "Push" button, "Switch Branch" dropdown, "Unmount" button
- Status indicators: ● Mounted / ○ Stale / ⚠ Error

**Acceptance criteria**:
- [ ] VFS Mounts section appears in Settings → Versions tab
- [ ] Mounted repos are listed with correct metadata
- [ ] "Mount Repository" modal opens and submits correctly
- [ ] "Fetch" button triggers a fetch and updates status
- [ ] "Push" button triggers a push
- [ ] "Switch Branch" dropdown lists available branches
- [ ] "Unmount" button removes the mount (with confirmation)
- [ ] Status indicators update correctly

---

## Phase 4: Branch Sync & Conflict Resolution

**Goal**: Users can see sync status and resolve conflicts when local and remote have diverged. This is the final functional phase.

**Independent Test**: User resolves a merge conflict using `merge --squash` via agent tool, and the conflict is resolved correctly.

**Checkpoint**: Conflict resolution works for all three strategies. Auto-push on promote triggers correctly.

---

### T017 [F4] Agent tools: merge and sync operations

Create the agent tools for conflict resolution and sync operations.

**File**: `src/lib/assistant/tools/server/git-merge.ts` (new)

**What to implement**:

```typescript
// git_merge
tool: git_merge
description: "Resolve a branch conflict by merging remote changes"
params: {
  repoPath: string,
  remote: string,
  branch: string,
  strategy: "merge-squash" | "merge" | "commit",
  confirm: boolean  // MUST be true — requires explicit user approval
}
returns: {
  status: "success" | "cancelled" | "failed",
  method: string,
  commitMessage?: string,
  error?: { code, message, suggestion? }
}

// git_sync
tool: git_sync
description: "Fetch and optionally resolve conflicts for a remote branch"
params: {
  repoPath: string,
  remote: string,
  branch: string,
  conflictStrategy: "merge-squash" | "merge" | "commit" | "abort"
}
returns: {
  status: "success" | "conflict_detected" | "failed",
  ahead: number,
  behind: number,
  conflict?: { strategy: string; requiresConfirmation: boolean }
}
```

**Implementation details**:

`git_merge`:
1. **MUST check `confirm === true`** — if false, return `{ status: "cancelled", error: "requires user confirmation" }` without doing anything
2. Acquire `git-lock` for repo path
3. Fetch remote branch: `git fetch <remote> <branch>`
4. Based on strategy:
   - `"merge-squash"`: `git merge --squash <remote>/<branch>` then `git commit -m "Squashed merge of <branch>"`
   - `"merge"`: `git merge <remote>/<branch>` (creates merge commit)
   - `"commit"` (AD-002): `git stash --include-untracked` → `git merge --squash <remote>/<branch>` → `git commit -m "Fetch remote changes"` → `git stash pop`
5. If conflict detected during merge: return `{ status: "failed", error: { code: "MERGE_CONFLICT", message: "...", suggestion: "Resolve manually or try a different strategy" } }`
6. Release lock
7. Log at `info` for success, `warn` for conflicts

`git_sync`:
1. Acquire `git-lock` for repo path
2. Fetch: `git fetch <remote> <branch>`
3. Compute ahead/behind
4. If diverged: return `{ status: "conflict_detected", conflict: { strategy, requiresConfirmation: true } }`
5. If `conflictStrategy` is provided AND `confirm` is implicit (for auto-sync): execute the strategy
6. Release lock

**Acceptance criteria**:
- [ ] `git_merge` with `confirm: false` returns cancelled without doing anything
- [ ] `git_merge` with `confirm: true` and `strategy: "merge-squash"` squashes and commits
- [ ] `git_merge` with `strategy: "commit"` stashes local, commits remote, pops stash
- [ ] `git_merge` with `strategy: "merge"` creates a merge commit
- [ ] `git_merge` with conflicting branches returns structured error
- [ ] `git_sync` returns correct ahead/behind counts
- [ ] `git_sync` detects conflicts and returns `conflict_detected` status
- [ ] Tools are registered and callable
- [ ] Unit tests for all tool functions

---

### T018 [F4] Conflict resolution UI

Create the UI for conflict resolution dialogs.

**File**: `src/components/apps/settings/versions/ConflictResolutionDialog.tsx` (new component)

**What to implement**:
- Dialog that appears when a conflict is detected (via sync, push, or agent tool)
- Three resolution options as buttons:
  - **"Merge --squash (recommended)"** — squashes remote commits into a single commit
  - **"Merge"** — standard merge commit
  - **"Commit"** — stashes local, commits remote, pops stash
- "Abort" button to cancel without resolving
- Preview of the conflict (show conflicting files, diff)
- Confirmation message: "This will resolve the conflict. Are you sure?"

**Acceptance criteria**:
- [ ] Dialog appears when conflict is detected
- [ ] Three resolution options are displayed with descriptions
- [ ] "Merge --squash" button executes the squash merge
- [ ] "Merge" button executes the standard merge
- [ ] "Commit" button executes the commit strategy
- [ ] "Abort" button closes dialog without changes
- [ ] Preview shows conflicting files
- [ ] Confirmation message is displayed before execution

---

### T019 [F4] Auto-push on promote integration

Wire up the auto-push triggers into the Supervisor promote flow.

**File**: `tools/supervisor/supervisor.mjs` (modify existing)
**File**: `src/lib/gitops/auto-push.ts` (new)

**What to implement**:
- Create `auto-push.ts`:
  ```typescript
  // Get all remotes with auto-push enabled for a repo
  function getAutoPushRemotes(repoPath: string): GitRemoteConfig[]

  // Execute auto-push for all enabled remotes
  async function executeAutoPush(repoPath: string, branch: string): Promise<AutoPushResult[]>
  ```
- Modify `supervisor.mjs`:
  - After a successful promote (base branch is now the new HEAD), check for auto-push-enabled remotes
  - For each auto-push-enabled remote (non-origin only, per FR-015): call `executeAutoPush`
  - Log each push result
  - If any push fails, log warning but continue with other remotes
  - **Critical**: This happens within the same git-lock scope as the origin push to prevent collisions

**Acceptance criteria**:
- [ ] After promote, remotes with auto-push enabled receive the push
- [ ] Origin push happens first (if `BOS_PUSH_MODE=auto-on-promote`), then auto-push remotes
- [ ] Auto-push is serialized (no collision with user-initiated pushes)
- [ ] Failed auto-push is logged but doesn't block other auto-pushes
- [ ] Non-origin remotes only — origin is controlled by `BOS_PUSH_MODE`
- [ ] Unit tests for `auto-push.ts`

---

### T020 [F4] Settings UI: Branch sync status panel

Create the Branch Sync Status panel in the Settings UI.

**File**: `src/components/apps/settings/versions/BranchSyncPanel.tsx` (new component)

**What to implement**:
- A panel showing sync status for all remotes and mounts
- For each remote/mount: name, current branch, remote branch, ahead/behind counts, last fetched
- Color coding: green (up to date), yellow (ahead/behind), red (diverged/conflict)
- "Sync All" button to fetch all remotes and mounts
- "View Details" link for each entry → opens conflict resolution dialog if diverged
- Auto-refresh: update status every 5 minutes (configurable)

**Acceptance criteria**:
- [ ] Panel appears in Settings → Versions tab
- [ ] All remotes and mounts are listed with sync status
- [ ] Color coding is correct (green/yellow/red)
- [ ] "Sync All" button fetches all and updates status
- [ ] "View Details" opens conflict resolution dialog for diverged entries
- [ ] Status updates automatically (polling or event-based)

---

## Phase 5: E2E Tests & Polish

**Goal**: Comprehensive testing and user experience polish.

**Checkpoint**: All E2E tests pass. Performance benchmarks met. Documentation complete.

---

### T021 [P] [F5] E2E tests: Remote registration and push

Write E2E tests for remote registration and push operations.

**File**: `e2e/001-external-repo-integration.spec.ts` (extend existing test file)

**What to implement**:
- Test suite: "Register GitHub Remote with OAuth"
  - Register a GitHub remote via OAuth flow
  - Verify remote appears in Settings
  - Push to the remote
  - Verify push succeeded
- Test suite: "Register Generic Remote with Token"
  - Register a self-hosted git remote with PAT
  - Push to the remote
- Test suite: "Multi-Remote Push"
  - Register multiple remotes
  - Push to all remotes
  - Verify all pushes succeeded
- Test suite: "Remote Removal"
  - Remove a remote
  - Verify it's gone from Settings and git config

**Acceptance criteria**:
- [ ] All tests pass
- [ ] Tests cover OAuth flow, token auth, multi-remote push, removal
- [ ] Tests use mock git server or local git repos

---

### T022 [P] [F5] E2E tests: VFS mounts

Write E2E tests for VFS mount operations.

**File**: `e2e/001-external-repo-integration.spec.ts` (extend existing test file)

**What to implement**:
- Test suite: "Mount Repository to VFS Path"
  - Mount a repo to `/Projects/my-webapp`
  - Verify directory exists in VFS
  - Edit a file
  - Push changes
  - Verify changes appear in remote
- Test suite: "Unmount Repository"
  - Unmount a repo
  - Verify directory is removed
  - Verify cache is preserved
- Test suite: "Switch Branch"
  - Switch branch of mounted repo
  - Verify working tree updates
- Test suite: "Path Validation"
  - Try to mount to `/Documents/test` — should fail
  - Try to mount to `data/vfs/apps/test` — should fail

**Acceptance criteria**:
- [ ] All tests pass
- [ ] Tests cover mount, unmount, branch switch, path validation
- [ ] Tests verify file operations work in mounted directories

---

### T023 [P] [F5] E2E tests: Sync and conflict resolution

Write E2E tests for branch sync and conflict resolution.

**File**: `e2e/001-external-repo-integration.spec.ts` (extend existing test file)

**What to implement**:
- Test suite: "View Sync Status"
  - View sync status panel
  - Verify ahead/behind counts are correct
- Test suite: "Resolve Merge Conflict"
  - Create diverged branches
  - Trigger sync
  - Resolve conflict using `merge --squash`
  - Verify conflict is resolved
- Test suite: "Agent Conflict Resolution"
  - Agent calls `git_merge` with `confirm: true`
  - Verify resolution succeeds
  - Agent calls `git_merge` with `confirm: false`
  - Verify it's cancelled

**Acceptance criteria**:
- [ ] All tests pass
- [ ] Tests cover sync status, conflict resolution (all 3 strategies), agent tools
- [ ] Tests verify `confirm: false` is properly rejected

---

### T024 [F5] Performance benchmarks and polish

Run performance benchmarks and polish the user experience.

**What to implement**:
- Benchmark: push to 5 remotes within 30 seconds
- Benchmark: mount clone completes within 60 seconds for repos up to 500MB
- Benchmark: sync status fetch within 2 seconds
- Polish: error messages are clear and actionable
- Polish: loading states are shown during long operations
- Polish: progress indicators for large operations
- Documentation: user-facing guide for the feature

**Acceptance criteria**:
- [ ] All benchmarks meet targets
- [ ] Error messages include suggestions for resolution
- [ ] Loading states are shown for operations > 1 second
- [ ] Progress indicators for clone/push operations
- [ ] User-facing documentation is complete
