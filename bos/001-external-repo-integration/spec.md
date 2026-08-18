# Feature Specification: External Repository Integration

**Feature Branch**: `001-external-repo-integration`

**Created**: 2026-07-23

**Status**: Draft

**Input**: User wants to add external repository push support to BrowserOS, including:
- Register external git repositories and connect them to BOS git instances (BOS source repo and apps content repo)
- Select which branch to push to per remote
- GitHub and GitLab OAuth integration for authentication
- Support for generic git providers (self-hosted, Gitea, Bitbucket, etc.)
- Agent tools for programmatic interaction
- VFS mount capability: clone arbitrary repos to any VFS path (e.g., `/Projects/my-webapp`)
- Multi-remote push: each registered remote is automatically added to the git repo, enabling push to multiple remotes
- Branch sync tracking: monitor remote branches and show sync status

## User Scenarios & Testing

### User Story 1 - Register External Git Remotes with Authentication (Priority: P1)

Users can register external git repositories as remotes on their BOS repos (BOS source and apps content). Each remote is authenticated via OAuth (GitHub/GitLab) or manual credentials (token or SSH key).

**Why this priority**: This is the foundational capability. Without registration and authentication, no push or mount operations are possible. It delivers immediate value by enabling users to push their BOS source to backup remotes (GitHub, GitLab, self-hosted).

**Independent Test**: User registers a GitHub remote on the BOS source repo, authenticates via OAuth, and successfully pushes a test commit to a test branch.

**Acceptance Scenarios**:

1. **Given** a BOS repo exists (source or apps content), **When** user registers a new remote via Settings or agent tool with a valid URL, **Then** the remote is added to git (`git remote add`) and appears in the Settings UI.

2. **Given** a registered remote using GitHub or GitLab, **When** user selects OAuth authentication, **Then** they are redirected to the provider's authorize page, and upon approval, tokens are stored in SecretsStore and the remote shows as "connected."

3. **Given** a registered remote using a generic provider (self-hosted git, Gitea, Bitbucket), **When** user selects token authentication and enters a PAT, **Then** the token is stored encrypted and the remote shows as "connected."

4. **Given** a registered remote, **When** user selects SSH key authentication and provides a private key via a secure UI channel (password prompt dialog, never JSON), **Then** the key is stored encrypted in SecretsStore and can be used for git operations via `GIT_SSH_COMMAND`. SSH passphrases MUST never be transmitted via agent tool calls, conversation history, or JSON arguments — they are collected exclusively through a secure UI channel.

5. **Given** a registered remote, **When** user views it in Settings, **Then** they see the URL, provider icon, auth status, and can test the connection or remove it.

---

### User Story 2 - Push to External Remotes (Priority: P1)

Users can push to any registered remote, including multi-remote push (push to all remotes simultaneously). Auto-push can be enabled per remote to trigger on promote.

**Why this priority**: This is the primary user value. Users want to push their work to backup remotes, not just the local origin. Multi-remote push and auto-push make this frictionless.

**Independent Test**: User pushes to two registered remotes (one GitHub, one GitLab) from the Settings UI and verifies both remotes receive the commit.

**Acceptance Scenarios**:

1. **Given** one or more registered remotes, **When** user clicks "Push" on a specific remote, **Then** the current branch is pushed to that remote and the result is shown (success or error).

2. **Given** multiple registered remotes, **When** user clicks "Push All," **Then** the current branch is pushed to all remotes sequentially, and a summary shows which succeeded and which failed.

3. **Given** a remote with auto-push enabled, **When** the user promotes a feature branch, **Then** the base branch is automatically pushed to that remote after the promote completes.

4. **Given** a push to a remote, **When** the authentication expires or fails, **Then** the error is clearly reported with a suggestion to re-authenticate (e.g., "OAuth token expired — reconnect via OAuth").

5. **Given** a remote that is configured with a different default branch (e.g., "main" vs "master"), **When** the user pushes, **Then** the push targets the correct remote branch (not necessarily the local branch name).

---

### User Story 3 - VFS Mount External Repos to Arbitrary Paths (Priority: P2)

Users can clone external repositories to any path within their VFS (e.g., `/Projects/my-webapp`) and treat them as regular filesystem directories. Mounted repos support fetch, push, and branch switching.

**Why this priority**: This enables users to work on unrelated git projects within BOS as if they were local directories. It's a significant quality-of-life improvement but not required for the core push functionality.

**Independent Test**: User mounts a GitHub repository to `/Projects/my-webapp`, edits a file, pushes changes, and verifies they appear in the remote.

**Acceptance Scenarios**:

1. **Given** a user has access to an external git repo, **When** they mount it to a VFS path (e.g., `/Projects/my-webapp`), **Then** the repo is cloned (full history) to that path and appears as a regular directory in the VFS. The system MUST reject mounts to protected VFS roots (`Documents`, `Pictures`, `Desktop`, `Workflows`, `Chats`) without explicit "danger mode" confirmation. The system MUST check if the target path or any subpath contains a `.git` directory and warn the user before proceeding.

2. **Given** a mounted repo, **When** the user fetches updates, **Then** the local working tree is updated with the latest remote commits without overwriting uncommitted changes.

3. **Given** a mounted repo with uncommitted changes, **When** the user pushes, **Then** their changes are pushed to the remote and the working tree is clean.

4. **Given** a mounted repo, **When** the user switches branches, **Then** the working tree updates to the new branch and any conflicting uncommitted changes are either stashed or cause a warning.

5. **Given** a mounted repo, **When** the user unmounts it, **Then** the VFS directory is removed but the bare clone cache is preserved for quick remounting.

6. **Given** a sync operation detects a conflict (local and remote have diverged), **When** the user triggers a sync, **Then** the system resolves it automatically without a blocking confirmation dialog, per the shared reconciliation pipeline (User Story 6): create a rollback tag, attempt the configured strategy (merge --squash / merge / commit), and if that hits a real conflict, attempt a scripted rebase-based fallback. If both automatic attempts fail, escalate to the DevOps Agent rather than asking the user to choose a strategy synchronously.

7. **Given** a user mounts a repo to a path within `data/vfs/`, **When** they view it in the Files app, **Then** it appears as a regular directory with git-aware tooling available.

---

### User Story 4 - Branch Sync Status and Monitoring (Priority: P2)

Users can see the sync status of all remotes and mounted repos, including how many commits they are ahead/behind, and can initiate merges (standard or --squash) or commit (stash and commit locally).

**Why this priority**: Users need visibility into sync state to know when their local work is out of date with remotes. This prevents accidental overwrites and keeps workflows smooth.

**Independent Test**: User views the Branch Sync Status panel, sees that "origin" is 3 commits behind, fetches, merges, and verifies the status updates to "up to date."

**Acceptance Scenarios**:

1. **Given** one or more remotes or mounted repos, **When** the user views the Branch Sync Status panel, **Then** they see the current branch, the remote branch, and the ahead/behind counts.

2. **Given** a remote that is behind the local branch, **When** the user clicks "Sync," **Then** the remote is fetched and the user is prompted to merge (--squash recommended), merge (standard), or discard local changes.

3. **Given** a remote that is ahead of the local branch, **When** the user clicks "Sync," **Then** the local branch is updated with the remote's commits.

4. **Given** a remote with conflicting commits (both local and remote have diverged), **When** the user attempts to sync, **Then** the system runs the shared reconciliation pipeline (User Story 6) automatically: rollback tag → configured strategy (merge --squash / merge / commit) → scripted rebase fallback on conflict → DevOps Agent escalation if both automatic attempts fail. No synchronous strategy-choice dialog is presented; the user's oversight point is the DevOps Agent's chat (visible, stoppable, resumable), not a blocking confirm step.

5. **Given** auto-sync is enabled for a remote, **When** the user opens the Files app or triggers a push, **Then** the remote is automatically fetched and the status is updated.

---

### User Story 5 - Agent Tools for Repository Management (Priority: P1)

Agents can interact with all repository management features via tools, including registering remotes, pushing, mounting, and checking sync status.

**Why this priority**: Agents are the primary interface for BOS self-modification. Without tools, agents cannot manage external repos or VFS mounts, which breaks the workflow for developers who want to push their work or work on external projects.

**Independent Test**: User delegates a task to an agent: "Push my current changes to GitHub and GitLab." The agent uses tools to push to both remotes and reports success.

**Acceptance Scenarios**:

1. **Given** an agent is running, **When** it calls `git_add_remote` with a URL and auth config, **Then** the remote is registered and authenticated.

2. **Given** an agent needs to push to multiple remotes, **When** it calls `git_push_all_remotes`, **Then** all remotes receive the push and the agent gets a summary of results.

3. **Given** an agent needs to work on an external project, **When** it calls `git_mount` with a URL and VFS path, **Then** the repo is cloned to that path and the agent can read/write files as usual.

4. **Given** an agent needs to check sync status, **When** it calls `git_mount_status` or `git_list_remotes`, **Then** it gets the current state of all remotes and mounts.

5. **Given** an agent needs to resolve a branch conflict, **When** it calls `git_merge`, **Then** the tool runs the shared reconciliation pipeline (rollback tag → strategy → rebase fallback → DevOps Agent escalation) automatically and returns either a resolved result or a `devopsConversationId` pointing at the escalated agent conversation — never a synchronous confirmation prompt.

6. **Given** an agent encounters an auth error, **When** it reports the error, **Then** the error includes enough context for the user to fix it (e.g., "OAuth token expired — reconnect via Settings").

---

### User Story 6 - Automated Conflict Resolution via DevOps Agent (Priority: P1)

Every git-backed content root in BOS (GitFS instance) — the BOS source repo, spec stores, the installed-apps content repo, and VFS-mounted repos alike — resolves push/pull conflicts through **one shared reconciliation pipeline**, not per-surface bespoke logic and not a synchronous confirmation dialog. The pipeline tries increasingly capable automatic steps and only falls back to a supervised, autonomous DevOps Agent when scripted reconciliation can't complete safely — at which point the user's oversight is a visible, stoppable, resumable agent chat rather than a modal choice between merge strategies.

**Why this priority**: Conflicts are the primary reason pushes/promotes fail, and every prior design (per-surface confirm dialogs) pushed the resolution burden back onto the user synchronously, at the exact moment they're blocked. A single pipeline used everywhere means one place to get conflict handling right, and a fallback that actually resolves things (a coding agent with full file/bash tools) instead of a dialog offering the same three strategies that already failed.

**Independent Test**: A feature branch and the base branch are made to diverge (both have unique commits). Promoting is triggered. The pipeline creates a rollback tag, syncs base with origin, attempts `merge --squash`, hits a conflict, attempts a scripted rebase, hits a conflict again, and escalates to the DevOps Agent. The agent (via `dev_delegate`) resolves the conflict in the existing Supervisor-tracked preview worktree, commits, and the promote pipeline resumes to completion. The resulting conversation remains in the Assistant app's conversation list afterward.

**Acceptance Scenarios**:

1. **Given** any GitFS instance (BOS source, a spec store, the apps repo, or a VFS mount) is being reconciled with its remote, **When** reconciliation begins, **Then** a rollback tag is created on the current HEAD before any merge/rebase is attempted, so the pre-reconciliation state is always recoverable.

2. **Given** a rollback tag has been created, **When** reconciliation proceeds, **Then** the local branch is first synced with the remote (fetch + fast-forward where possible) before any feature/local content is merged in, reducing the odds of a conflict that a plain fast-forward would have avoided.

3. **Given** the configured strategy (merge --squash / merge / commit) completes cleanly, **When** reconciliation finishes, **Then** no fallback or escalation occurs and the operation reports success exactly as before.

4. **Given** the configured strategy hits a real conflict, **When** the pipeline retries, **Then** it aborts the failed attempt cleanly (`git reset --hard`/`git clean -fd` for a failed `merge --squash`, which never sets `MERGE_HEAD`; `git rebase --abort` plus removal of `.git/rebase-merge`/`.git/rebase-apply` for a failed rebase) before trying the next strategy, so the working tree is never left in a half-resolved state between attempts.

5. **Given** both the configured strategy and the scripted rebase fallback fail with conflicts, **When** the pipeline escalates, **Then** it creates (or reuses) a persisted Assistant conversation scoped to the "devops" agent, pre-sets that conversation's active feature branch so `dev_delegate` can target the existing Supervisor-tracked worktree without an interactive branch-setup step, and starts a run with full context (repo, remote, branch, base commit, conflicting files).

6. **Given** an escalated conversation, **When** the DevOps Agent works the conflict, **Then** it delegates the actual file-level resolution to the Developer sub-agent via `dev_delegate` rather than editing files itself, verifies the result (no conflict markers remain, build/tests pass if present), and reports success or failure — it never force-pushes and never pushes at all itself; the final push remains the caller's responsibility (e.g. the Supervisor's point-of-no-return step, or a subsequent `git_push` call).

7. **Given** a caller (e.g. the Supervisor's `promote()`) is blocked waiting on an escalated run, **When** the browser tab that initiated the operation refreshes or disconnects, **Then** the server-side wait is unaffected (it is not tied to that HTTP connection) and continues polling for run completion; the UI reflects the in-progress escalation via polled state (a distinct status plus the conversation id) rather than an ambiguous "stuck" indicator, and a repeat attempt on the same target while an escalation is in progress is rejected/re-surfaces the same conversation instead of starting a second parallel one.

8. **Given** an escalated run exceeds a maximum wait duration with no terminal outcome, **When** the timeout is reached, **Then** the caller stops waiting and reports a distinct "escalation timed out" outcome (not a generic failure) — the conversation is left exactly as it was (still resumable/stoppable) so the user can pick it back up whenever they return.

9. **Given** a DevOps Agent conversation, **When** it is created, **Then** it is a normal, persisted conversation (visible in the Assistant app's conversation list) from the moment it's created — never an ephemeral or hidden run — so it remains discoverable and resumable regardless of what happens to the operation that triggered it.

---

### Agent Tool Definitions

#### git_merge
```typescript
tool: git_merge
  description: "Resolve a branch conflict by merging remote changes. Runs the shared reconciliation pipeline (User Story 6) automatically — no confirmation flag; on unresolvable conflict it escalates to the DevOps Agent instead of failing back to the caller."
  params: {
    repoPath: string,           // e.g. "." or "/Projects/my-webapp"
    remote: string,             // e.g. "origin", "github-work"
    branch: string,             // e.g. "main", "develop"
    strategy: "merge-squash" | "merge" | "commit"
  }
  returns: {
    status: "success" | "escalated" | "failed",
    method: string,             // e.g. "merge --squash", "rebase-fallback"
    commitMessage: string,      // if strategy is commit
    devopsConversationId?: string, // present when status is "escalated"
    error?: { code, message, suggestion? }
  }
```

**Behavior** (shared reconciliation pipeline — see User Story 6):
1. Create a rollback tag on the current HEAD.
2. Sync the local branch with `<remote>/<branch>` first (fetch + fast-forward where possible).
3. Attempt the requested strategy: `merge-squash` → `git merge --squash <remote>/<branch>` then commit; `merge` → `git merge <remote>/<branch>`; `commit` → `git stash --include-untracked`, commit remote changes, `git stash pop`.
4. On conflict, abort cleanly (`git reset --hard`/`git clean -fd` for a failed `merge --squash` — it never sets `MERGE_HEAD`) and attempt a scripted rebase-based fallback.
5. On conflict again, abort the rebase (`git rebase --abort`, remove `.git/rebase-merge`/`.git/rebase-apply`) and escalate to the DevOps Agent — return `status: "escalated"` with `devopsConversationId` rather than blocking or asking the caller to choose a strategy.
6. MUST serialize via git-lock to prevent concurrent operations.

#### git_sync
```typescript
tool: git_sync
  description: "Fetch and reconcile a remote branch via the shared reconciliation pipeline (User Story 6). No confirmation flag; unresolvable conflicts escalate to the DevOps Agent."
  params: {
    repoPath: string,
    remote: string,
    branch: string,
    conflictStrategy: "merge-squash" | "merge" | "commit"
  }
  returns: {
    status: "success" | "escalated" | "failed",
    ahead: number,              // commits ahead of remote
    behind: number,             // commits behind remote
    devopsConversationId?: string // present when status is "escalated"
  }
```

**Behavior**:
1. `git fetch <remote> <branch>`
2. Compare `HEAD` vs `<remote>/<branch>`
3. If ahead only → no action needed (report ahead count)
4. If behind only → `git pull --ff-only` (fast-forward)
5. If diverged → run the shared reconciliation pipeline with `conflictStrategy` as the initial strategy; return `status: "escalated"` with `devopsConversationId` if it can't resolve automatically

#### git_fetch
```typescript
tool: git_fetch
  description: "Fetch updates from a remote repository"
  params: {
    repoPath: string,
    remote?: string,            // if omitted, fetch all remotes
    branch?: string             // if omitted, fetch all branches
  }
  returns: {
    status: "success" | "failed",
    updates: {
      newBranches: string[],
      updatedBranches: string[],
      deletedBranches: string[]
    },
    error?: string
  }
```

#### git_push
```typescript
tool: git_push
  description: "Push current branch to a remote"
  params: {
    repoPath: string,
    remote: string,
    branch?: string             // if omitted, push current branch
  }
  returns: {
    status: "success" | "failed",
    pushed: boolean,
    error?: { code, message, suggestion? }
  }
```

#### git_push_all_remotes
```typescript
tool: git_push_all_remotes
  description: "Push current branch to all registered remotes"
  params: {
    repoPath: string,
    remoteNames?: string[],     // if omitted, push to all remotes
    branch?: string
  }
  returns: {
    results: PushResult[]
  }
```

#### git_add_remote
```typescript
tool: git_add_remote
  description: "Register a new git remote with authentication"
  params: {
    repoPath: string,
    name: string,
    url: string,
    provider: "github" | "gitlab" | "generic",
    authType: "oauth" | "token" | "ssh"
  }
  returns: {
    status: "success" | "error",
    name: string,
    url: string,
    autoPush: boolean
  }
```

#### git_remove_remote
```typescript
tool: git_remove_remote
  description: "Remove a git remote"
  params: {
    repoPath: string,
    name: string
  }
  returns: {
    status: "success" | "error",
    message: string
  }
```

#### git_mount
```typescript
tool: git_mount
  description: "Clone and mount an external repository to a VFS path"
  params: {
    repoUrl: string,
    targetPath: string,
    branch?: string,
    authType: "oauth" | "token" | "ssh",
    auth?: {                   // optional auth details
      token?: string,
      sshKey?: string
    }
  }
  returns: {
    status: "mounted" | "error",
    path: string,
    branches: string[],
    error?: string
  }
```

#### git_unmount
```typescript
tool: git_unmount
  description: "Unmount a mounted repository"
  params: {
    vfsPath: string
  }
  returns: {
    status: "success" | "error",
    message: string
  }
```

#### git_list_mounts
```typescript
tool: git_list_mounts
  description: "List all mounted repositories"
  returns: VfsMount[]
```

#### git_mount_status
```typescript
tool: git_mount_status
  description: "Get sync status for a mounted repository"
  params: {
    vfsPath: string
  }
  returns: SyncStatus
```

#### git_list_remotes
```typescript
tool: git_list_remotes
  description: "List all git remotes for a repository"
  params: {
    repoPath: string
  }
  returns: GitRemote[]
```

---

### Edge Cases

- What happens when a user registers a remote with the same name as an existing one? → System auto-renames (e.g., "origin-2") to avoid conflicts.
- What happens when a remote's authentication expires mid-push? → Push fails with a clear error; user can re-authenticate via Settings or agent tool.
- What happens when a user mounts a repo to a path that already has files? → System warns the user and asks if they want to overwrite or merge.
- What happens when a mounted repo has uncommitted changes and the user tries to fetch? → Before stashing, the system checks `git stash show` for uncommitted changes. If present, a dialog presents options: (a) Commit, (b) Discard, (c) Stash & switch. Never auto-stash silently.
- What happens when a user tries to push to a remote that is not connected? → System shows an error with a link to the auth settings.
- What happens when a user removes a remote that is currently mounted? → System warns the user that the mount will become orphaned and asks for confirmation.
- What happens when a remote URL changes (e.g., repo is moved)? → User can update the URL in Settings or via agent tool; git remote set-url handles the update.
- What happens when SSH key passphrase is required but not provided? → System prompts the user for the passphrase before pushing.
- What happens when the DevOps Agent itself can't resolve a conflict (e.g. semantically ambiguous, or it stops itself)? → The conversation remains exactly as left — visible, resumable, and stoppable in the Assistant app — and the caller's wait ends in an explicit "escalated, unresolved" outcome rather than being silently retried or discarded.
- What happens when the escalation wait exceeds its maximum duration? → The caller reports a distinct "escalation timed out" outcome (not a generic failure); the conversation is untouched and can be resumed later, and the rollback tag from step 1 of the pipeline remains available if a manual revert is needed.
- What happens if a second reconciliation is requested for a target that already has an in-progress escalation? → The request is rejected/short-circuited to the existing `devopsConversationId` rather than starting a second parallel pipeline against the same repo.
- What happens if the DevOps Agent (via `dev_delegate`) attempts to force-push? → Refused by the agent's Skill instructions; force-push is never part of the DevOps Agent's or the pipeline's automatic behavior, only ever a separate, explicit, user-confirmed action (see US2/US4 force-push UI).

## Architecture Decisions

The following decisions were resolved during the architectural review and are incorporated into this specification.

### AD-001: Configuration Storage Location

`git-remotes.json` lives in `data/config/git-remotes.json` as a dedicated config namespace (not in `data/integrations/`). This is configuration, not a SaaS integration, and must survive feature promote.

### AD-002: `commit` Conflict Strategy Semantics

The `commit` strategy means: fetch remote changes, commit the fetched changes as a new local commit, then apply the locally stashed changes on top. This preserves both histories as separate commits, rather than the alternative interpretation of stashing local first, committing remote, then popping stash.

### AD-003: OAuth Integration IDs

GitHub and GitLab git integrations use `github-git` and `gitlab-git` as their integration IDs in the existing OAuth callback route. These are namespaced to avoid collision with existing SaaS integrations (gsuite, telegram, etc.).

### AD-004: Mount Path Restrictions

Mounted repo paths MUST be within `data/vfs/` and MUST NOT be under `data/vfs/apps/` or `data/vfs/workflows/` (reserved for GitFS/DataFS). Validated in `git_mount` at call time.

### AD-005: Bare Cache Force-Push Handling

On fetch, if the remote ref has moved forward (force-push), the bare cache ref is force-updated and mounted working trees are notified to re-fetch. The cache is keyed by URL hash so URL changes trigger cache invalidation.

### AD-006: UI Layout for Providers vs Remotes

- **Settings → Integrations**: "Git Providers" section for OAuth setup (connect/disconnect GitHub/GitLab accounts).
- **Settings → Versions**: "Git Remotes" section for remote management (add/edit/remove remotes, auto-push toggles, mount manager, sync status).

### AD-007: Automatic Reconciliation Supersedes the Synchronous Confirm Gate

Superseded: the original design (`git_merge`/`git_sync` `confirm: true` requirement, and the US3/US4 "resolution dialog" acceptance scenarios) required a human to synchronously choose a merge strategy before any conflict resolution executed. This is replaced, for every GitFS instance, by the shared reconciliation pipeline (User Story 6): automatic strategy attempt → automatic rebase-based fallback → escalation to a supervised, autonomous DevOps Agent. The safety property moves from "a human approves the specific git operation before it runs" to "a human can observe, interrupt (Stop), and resume the agent doing the work, and every attempt is preceded by a rollback tag." This is a deliberate, uniform behavior change across all GitFS instances (BOS source, spec stores, apps repo, VFS mounts) — not a promote-only special case. Force-push remains a separate, always-explicit, always-user-confirmed action (AD-008) — it is never part of the automatic pipeline or the DevOps Agent's own behavior.

### AD-008: Force-Push Is Explicit, Confirmed, and Uses `--force-with-lease`

When automatic reconciliation genuinely can't apply (e.g. the DevOps Agent itself reports it can't proceed, or the user chooses to override), the UI offers a distinct, explicitly-labeled "Force push" action, gated behind a confirmation dialog that states what will be discarded from the remote. It always uses `git push --force-with-lease` (never bare `--force`), so it still fails safely if the remote has moved again since the last fetch — it cannot silently clobber a concurrent push it hasn't seen. Neither the reconciliation pipeline nor the DevOps Agent may invoke this on their own.

## Flow Diagrams

### US1: Register External Git Remotes with Authentication

```mermaid
sequenceDiagram
    participant User
    participant Settings
    participant Tool
    participant SecretsStore
    participant GitOps
    
    User->>Settings: Open Settings → Versions
    Settings->>Settings: Show Git Remotes section
    User->>Settings: Click "Add Remote"
    Settings->>Settings: Show Add Remote modal
    User->>Settings: Enter name, URL, select provider (GitHub)
    User->>Settings: Select auth type (OAuth)
    Settings->>Tool: git_add_remote(name, url, "github", "oauth")
    Tool->>Tool: Validate URL (reject git://)
    Tool->>Tool: Acquire git-lock
    Tool->>SecretsStore: Resolve auth (startOAuth flow)
    SecretsStore-->>Tool: OAuth callback with token
    Tool->>GitOps: addRemote(".", name, url)
    GitOps-->>Tool: Remote added
    Tool->>Tool: Store token in SecretsStore
    Tool->>Tool: Add remote config to git-remotes.json
    Tool-->>Settings: Remote registered
    Settings-->>User: Show "● Connected"
    Settings-->>User: List remotes with provider icon
```

### US2: Push to External Remotes (Single & Multi-Remote)

```mermaid
sequenceDiagram
    participant User
    participant Settings
    participant Tool
    participant GitLock
    participant GitOps
    participant Remotes
    
    alt Single Remote Push
        User->>Settings: Click "Push" on remote card
        Settings->>Tool: git_push(repoPath, remote, branch)
        Tool->>GitLock: acquire(repoPath)
        Tool->>GitOps: pushRepo(repoPath, remote, branch, auth)
        GitOps->>Remotes: git push origin branch
        Remotes-->>GitOps: Push success
        GitOps-->>Tool: Result
        Tool->>Tool: Update lastPushed in config
        Tool->>GitLock: release()
        Tool-->>Settings: Show "✓ Pushed to github-work"
        Settings-->>User: Status updated
    else Multi-Remote Push
        User->>Settings: Click "Push All" button
        Settings->>Tool: git_push_all_remotes(repoPath, remotes)
        Tool->>GitLock: acquire(repoPath)
        loop For each remote
            Tool->>GitOps: pushRepo for remote
            GitOps-->>Tool: Push result
        end
        Tool->>GitLock: release()
        Tool-->>Settings: Show summary (✓ origin, ✓ github, ✗ gitlab)
        Settings-->>User: Summary with individual results
    end
```

### US3: VFS Mount External Repos to Arbitrary Paths

```mermaid
sequenceDiagram
    participant User
    participant Settings
    participant Tool
    participant GitLock
    participant MountManager
    participant GitOps
    participant VFS
    
    User->>Settings: Click "Mount Repository"
    Settings->>Tool: git_mount(url, targetPath, branch, auth)
    Tool->>Tool: Validate targetPath (within data/vfs/)
    Tool->>Tool: Reject git:// URLs
    Tool->>GitLock: acquire(targetPath)
    Tool->>MountManager: mount(url, path, branch, auth)
    MountManager->>MountManager: Validate mount path
    MountManager->>GitOps: cloneRepo(url, path, branch, auth)
    GitOps-->>MountManager: Clone complete (full history)
    MountManager->>MountManager: scanSymlinkEscapes(path)
    alt Escapes detected
        MountManager-->>Tool: Abort with warning
        Tool-->>Settings: Show "Symlink escape detected"
    else No escapes
        MountManager->>VFS: Register mount point
        MountManager->>Tool: Mount entry created
    end
    Tool->>Tool: Update vfs-mounts.json
    Tool->>GitLock: release()
    Tool-->>Settings: Show mounted with status
    Settings-->>User: Directory appears in Files app
    Note over User,VFS: User can now read/write files in mounted repo
```

### US4: Branch Sync Status and Monitoring

```mermaid
sequenceDiagram
    participant User
    participant Settings
    participant Tool
    participant GitLock
    participant GitOps
    
    User->>Settings: Open Branch Sync Status panel
    Settings->>Tool: Get sync status for all remotes/mounts
    loop For each remote/mount
        Tool->>GitLock: acquire(repoPath)
        Tool->>GitOps: fetchRepo(repoPath, remote, branch)
        GitOps-->>Tool: Fetch result
        Tool->>Tool: Compute ahead/behind
        Tool->>Tool: Determine status (up_to_date/ahead/behind/diverged)
        Tool->>GitLock: release()
    end
    Tool-->>Settings: Sync status for all entries
    Settings-->>User: Show color-coded status (green/yellow/red)
    
    alt User clicks Sync on diverged remote
        Settings->>Tool: git_sync(repoPath, remote, branch, "merge-squash")
        Tool->>GitLock: acquire(repoPath)
        Tool->>GitOps: fetchRepo then merge --squash
        GitOps-->>Tool: Merge result
        Tool->>GitLock: release()
        Tool-->>Settings: Conflict resolved
        Settings-->>User: Status updated to "up to date"
    end
```

### US5: Agent Tools for Repository Management

```mermaid
sequenceDiagram
    participant User
    participant Assistant
    participant Tools
    participant GitLock
    participant GitOps
    
    User->>Assistant: "Push my changes to GitHub and GitLab"
    Assistant->>Tools: git_list_remotes(".")
    Tools-->>Assistant: List of remotes (origin, github-work, gitlab-personal)
    Assistant->>Tools: git_push_all_remotes(".", ["github-work", "gitlab-personal"], "main")
    Tools->>GitLock: acquire(".")
    Tools->>GitOps: pushRepo for github-work
    GitOps-->>Tools: Success
    Tools->>GitOps: pushRepo for gitlab-personal
    GitOps-->>Tools: Success
    Tools->>GitLock: release()
    Tools-->>Assistant: Summary of results
    Assistant-->>User: "Pushed to github-work and gitlab-personal successfully"
    
    User->>Assistant: "Mount https://github.com/user/project.git to /Projects/my-webapp"
    Assistant->>Tools: git_mount(url, "/Projects/my-webapp", "main", "oauth")
    Tools->>GitLock: acquire("/Projects/my-webapp")
    Tools->>GitOps: cloneRepo with OAuth auth
    GitOps-->>Tools: Clone complete
    Tools->>GitLock: release()
    Tools-->>Assistant: Mount success
    Assistant-->>User: "Mounted repository to /Projects/my-webapp"
    
    User->>Assistant: "Resolve merge conflict on origin/main using --squash"
    Assistant->>Tools: git_merge(".", "origin", "main", "merge-squash")
    Tools->>GitLock: acquire(".")
    Tools->>GitOps: fetch then merge --squash
    GitOps-->>Tools: Merge success
    Tools->>GitLock: release()
    Tools-->>Assistant: Merge result (status: success)
    Assistant-->>User: "Conflict resolved via merge --squash"
```

### US6: Automated Conflict Resolution via DevOps Agent

```mermaid
sequenceDiagram
    participant Caller as Caller (Supervisor promote / git_merge / git_sync)
    participant Pipeline as Reconciliation Pipeline
    participant GitOps
    participant Conversations
    participant DevOpsAgent as DevOps Agent (type: local)
    participant Dev as Developer sub-agent (dev_delegate)

    Caller->>Pipeline: reconcile(repoPath, remote, branch, strategy)
    Pipeline->>GitOps: tag current HEAD (rollback anchor)
    Pipeline->>GitOps: fetch + fast-forward sync with remote
    Pipeline->>GitOps: attempt strategy (merge --squash / merge / commit)
    alt Strategy succeeds
        GitOps-->>Pipeline: clean
        Pipeline-->>Caller: status: success
    else Strategy conflicts
        Pipeline->>GitOps: abort (reset --hard + clean -fd; no MERGE_HEAD to abort)
        Pipeline->>GitOps: attempt scripted rebase fallback
        alt Rebase succeeds
            GitOps-->>Pipeline: clean
            Pipeline-->>Caller: status: success (method: rebase-fallback)
        else Rebase conflicts
            Pipeline->>GitOps: git rebase --abort; rm -rf .git/rebase-merge .git/rebase-apply
            Pipeline->>Conversations: create persisted conversation, agentId "devops"
            Pipeline->>Conversations: pre-set activeFeatureBranch
            Pipeline->>DevOpsAgent: startAssistantRun(conversationId, task with conflict context)
            Pipeline-->>Caller: status: escalated, devopsConversationId
            DevOpsAgent->>Dev: dev_delegate(task: resolve conflict in existing preview worktree)
            Dev->>Dev: resolve conflict, build/test, commit (never push)
            Dev-->>DevOpsAgent: result
            DevOpsAgent-->>Conversations: report outcome (run finishes)
            Note over Caller,Conversations: Caller polls run status independently of any browser connection
            Caller->>Conversations: poll run status
            Conversations-->>Caller: terminal state reached
            Caller-->>Caller: resume (e.g. Supervisor build + health-check + point-of-no-return)
        end
    end
```

## Settings UI Layout

### Settings → Versions → Git Remotes Tab

```
┌─────────────────────────────────────────────────────────────┐
│ Settings → Versions                                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─ Git Remotes ──────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  [Add Remote]                      [Push All Remotes]   │ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │ ● Connected  github-work                        │   │ │
│  │  │   GitHub • OAuth • https://github.com/user/repo │   │ │
│  │  │   Last pushed: 2 hours ago • Auto-push: ON      │   │ │
│  │  │   [Push] [Fetch] [Edit] [Remove] [Test Conn]    │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │ ● Connected  gitlab-personal                    │   │ │
│  │  │   GitLab • Token • https://gitlab.com/user/repo │   │ │
│  │  │   Last pushed: 1 day ago • Auto-push: OFF       │   │ │
│  │  │   [Push] [Fetch] [Edit] [Remove] [Test Conn]    │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌─ VFS Mounts ───────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  [Mount Repository]                                     │ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │ ● Mounted  /Projects/my-webapp                  │   │ │
│  │  │   https://github.com/user/project.git • main    │   │ │
│  │  │   Last fetched: 30 min ago • Status: up to date │   │ │
│  │  │   [Fetch] [Push] [Switch Branch ▼] [Unmount]    │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌─ Branch Sync Status ───────────────────────────────────┐ │
│  │                                                         │ │
│  │  origin/main           ● Up to date                     │ │
│  │  github-work/main      ● 3 commits behind              │ │
│  │  gitlab-personal/dev   ● Diverged (3/2)                │ │
│  │  /Projects/my-webapp   ● Up to date                     │ │
│  │                                                         │ │
│  │  [Sync All]                    [Refresh Status]         │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Settings → Integrations → Git Providers Tab

```
┌─────────────────────────────────────────────────────────────┐
│ Settings → Integrations                                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─ Git Providers ────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │ GitHub                                          │   │ │
│  │  │ ● Connected • user@example.com                  │   │ │
│  │  │ Scope: repo                                     │   │ │
│  │  │ Token expires: 2024-02-15                       │   │ │
│  │  │ [Reconnect] [Disconnect]                        │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐   │ │
│  │  │ GitLab                                          │   │ │
│  │  │ ○ Not connected                                 │   │ │
│  │  │ [Connect]                                        │   │ │
│  │  └─────────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌─ Other Integrations ───────────────────────────────────┐ │
│  │ (Existing integrations: Telegram, GSuite, etc.)        │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Add Remote Modal

```
┌─────────────────────────────────────────────────────────────┐
│ Add Remote                                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Remote name: [github-work___________________________]      │
│                                                             │
│  Repository URL: [https://github.com/user/repo.git______]   │
│                                                             │
│  Provider: [GitHub ▼]                                       │
│                                                             │
│  Authentication: [OAuth ▼]                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ OAuth: Will redirect to GitHub for authorization   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─ Token Auth (alternative) ───────────────────────────┐  │
│  │                                                         │  │
│  │  Personal Access Token: [___________________________]  │  │
│  │                                                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─ SSH Key Auth (alternative) ──────────────────────────┐  │
│  │                                                         │  │
│  │  SSH Private Key: [___________________________]        │  │
│  │  Passphrase:       [___________________________]       │  │
│  │                                                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                             │
│  [Cancel]                                        [Add]      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Logging

All git operations MUST be logged with structured, level-appropriate entries. Logs go to `data/logs/git-ops.log` with 7-day rotation.

| Log Level | Events |
|---|---|
| `debug` | Internal lock acquisition/release, cache hits/misses, SSH key temp file creation/deletion, URL parsing results, bare-clone dedup lookups |
| `info` | Remote registration/removal, successful push/fetch/clone/merge/checkout operations, mount/unmount lifecycle events, OAuth token refresh success, config save/load |
| `warn` | Connection test failures (retryable), stale bare cache detected, force-push detected on fetch, symlink escape detected (non-fatal), slow operations (> 30s), partial multi-remote push results, rate limit headers from providers |
| `error` | Push/fetch/clone/merge failures, auth failures (token expired, SSH key rejected), git-lock timeout, bare cache corruption, symlink escape (fatal — mount aborted), VFS path validation failure, unhandled git CLI errors |
| `critical` | SecretsStore encryption key not available, git-lock deadlock detected, data corruption detected in `.git/config` or `git-remotes.json` |

**Log entry format**: Structured JSON with `timestamp`, `level`, `op` (operation name), `repoPath`, `remote` (if applicable), `user` (session ID), `durationMs`, `success`, `error` (if any).

**Security-sensitive logging**:
- URLs are logged **without embedded credentials** (strip `user:pass@` from URLs before logging).
- SSH keys, tokens, and passphrases are **NEVER** logged (not even truncated).
- Auth failures log the remote name and error code, but not the token value or key fingerprint.

### Functional Requirements

- **FR-001**: System MUST allow users to register external git repositories as remotes on the BOS source repo and apps content repo.
- **FR-002**: System MUST support OAuth authentication for GitHub and GitLab, reusing the existing integrations framework.
- **FR-003**: System MUST support manual authentication via personal access tokens (PAT) and SSH keys, stored encrypted in SecretsStore.
- **FR-004**: System MUST allow users to push to a single remote or all registered remotes simultaneously.
- **FR-005**: System MUST support auto-push on promote, configurable per remote.
- **FR-006**: System MUST allow users to mount external repositories to any path within the VFS (e.g., `/Projects/my-webapp`).
- **FR-007**: System MUST support fetch, push, and branch switching for mounted repositories.
- **FR-008**: System MUST display branch sync status (ahead/behind counts) for all remotes and mounted repos.
- **FR-009**: System MUST provide agent tools for all repository management operations (register, push, mount, sync).
- **FR-010**: System MUST support generic git providers (self-hosted, Gitea, Bitbucket, etc.) via token or SSH authentication.
- **FR-011**: System MUST validate VFS mount paths to prevent writes outside the user's VFS. Mount paths must be within `data/vfs/` and NOT under `data/vfs/apps/` or `data/vfs/workflows/` (reserved for GitFS/DataFS). After cloning, symlinks MUST be scanned for escape attempts before exposing files through the Files app. `git://` protocol URLs MUST be rejected; allowlist only `https://`, `http://` (with explicit user opt-in for self-hosted), and `git@` (SSH).
- **FR-013**: System MUST canonicalize resolved VFS paths and reject mount targets outside `data/vfs/` or protected roots (`Documents`, `Pictures`, `Desktop`, `Workflows`, `Chats`). After cloning, symlinks MUST be scanned for escape attempts before exposing files through the Files app. `git://` protocol URLs MUST be rejected.
- **FR-014**: All git operations (push, fetch, clone, checkout, stash) across all remotes and mounts MUST be serialized through a system-wide `git-lock` mechanism to prevent concurrent access conflicts (e.g., push-All mid-fetch, simultaneous mounts, auto-push on promote colliding with user push).
- **FR-015**: The per-remote auto-push toggles apply ONLY to non-origin remotes. The Supervisor's `BOS_PUSH_MODE` continues to control the origin push. When promoting, the Supervisor pushes to origin first (if `BOS_PUSH_MODE=auto-on-promote`), then triggers per-remote auto-pushes sequentially. Any push failure (origin or per-remote) MUST be logged and returned in the promote result — never swallowed silently.
- **FR-012**: System MUST handle authentication errors gracefully, with clear error messages and suggestions for resolution.
- **FR-016**: System MUST apply the same shared reconciliation pipeline (rollback tag → remote-sync → configured strategy → scripted rebase fallback → DevOps Agent escalation) to every GitFS instance uniformly — the BOS source repo (including the Supervisor's promote flow), spec stores, the installed-apps content repo, and VFS-mounted repos. No GitFS instance gets bespoke conflict-handling logic.
- **FR-017**: Before any merge/rebase attempt in the reconciliation pipeline, System MUST create a rollback tag on the current HEAD of the target branch, so the pre-reconciliation state is always recoverable regardless of how the pipeline proceeds.
- **FR-018**: When the reconciliation pipeline cannot resolve a conflict automatically (configured strategy and scripted rebase fallback both fail), System MUST escalate to the DevOps Agent: create or reuse a normal, persisted Assistant conversation scoped to the "devops" agent, pre-set its active feature branch, and start a run — never present a synchronous strategy-choice dialog instead.
- **FR-019**: The DevOps Agent MUST delegate all file-level conflict resolution to the Developer sub-agent (`dev_delegate`) rather than editing files itself, MUST NOT push (the final push remains the triggering caller's responsibility), and MUST NOT force-push under any circumstance.
- **FR-020**: A caller blocked waiting on an escalated DevOps Agent run (e.g. the Supervisor's `promote()`) MUST NOT tie that wait to the lifecycle of any single client connection — the wait and its eventual resolution MUST survive a browser refresh or disconnect, with the escalation's live/final state discoverable via polled server state (not solely via the original request's response) and via the persisted conversation itself.
- **FR-021**: An escalation wait MUST be bounded by a maximum duration; on timeout, the caller MUST report a distinct "escalation timed out" outcome (not a generic failure) and MUST leave the conversation untouched and resumable.
- **FR-022**: System MUST reject (or transparently re-point to the existing conversation) a second concurrent reconciliation request against a target that already has an in-progress escalation, rather than starting a parallel pipeline against the same repo/branch.
- **FR-023**: System MUST offer force-push as a separate, always-explicit, always-user-confirmed UI action (never automatic, never invoked by the reconciliation pipeline or the DevOps Agent), implemented via `git push --force-with-lease`.

### Key Entities

- **GitRemote**: Represents a remote repository connection. Attributes: name, URL, provider (github/gitlab/generic), authType (oauth/token/ssh), connected status, autoPush flag, remoteBranch (optional branch mapping for push target), defaultBranch (discovered via `git ls-remote --symref`), lastFetched, lastPushed, oauthTokenExpiresAt.
- **VfsMount**: Represents a mounted external repository in the VFS. Attributes: id, vfsPath, repoUrl, branch, authType, status (mounted/stale/error), lastFetched, lastSynced, isBareCache, bare (whether underlying repo is bare).
- **PushResult**: Represents the result of a push operation. Attributes: remoteName, branch, status ("success" | "failed" | "partial_success"), error (structured: { code, message, suggestion? }), timestamp.
- **SyncStatus**: Represents the sync state of a remote or mount. Attributes: localBranch, remoteBranch, ahead, behind, lastFetched.
- **ReconciliationOutcome**: Represents the result of the shared pipeline. Attributes: status ("success" | "escalated" | "timed-out" | "failed"), method (which strategy resolved it, if any), rollbackTag, devopsConversationId (when escalated), startedAt, completedAt.
- **DevOps Agent**: A `type: "local"` subagent (reuses the standard agent-v2 run infrastructure — Stop button, streaming, persistence) whose sole job is to drive conflict escalations by delegating file-level work to the Developer sub-agent (`dev_delegate`) and reporting the outcome. Attributes as any `Agent` (id, systemPrompt, tools — limited to `dev_delegate` plus read-only inspection tools, skills — the devops-merge-conflict-resolution skill).

## Success Criteria

### Measurable Outcomes

- **SC-001**: Users can register a GitHub or GitLab remote and push to it within 3 minutes (including OAuth flow).
- **SC-002**: Users can mount an external repository to a VFS path and push changes within 5 minutes.
- **SC-003**: Multi-remote push completes within 30 seconds for up to 5 remotes on a stable connection.
- **SC-004**: 90% of users successfully complete the OAuth flow on the first attempt.
- **SC-005**: Branch sync status is accurate within 1 minute of fetching remote updates.
- **SC-006**: Agent tools can perform all repository management operations without user intervention (except for initial OAuth authorization).
- **SC-007**: A conflict resolved by the configured strategy or the scripted rebase fallback (no escalation needed) completes within the same timeframe as the equivalent single git command — the reconciliation pipeline adds no perceptible overhead over the strategy that resolves it.
- **SC-008**: When escalation to the DevOps Agent occurs, a distinct, discoverable status (with a conversation link) is visible within one polling interval (2.5s) of the escalation starting — never an ambiguous "stuck" state, with or without a browser refresh in between.

## E2E Test Plan

### Test Architecture

All E2E tests follow the BOS Playwright convention: browser automation via `@playwright/mcp`, scripted via `@@e2e` directives. Tests are organized by user story and cover UI interactions, agent tool invocations, and edge cases.

**Test File**: `e2e/001-external-repo-integration.spec.ts`

**Test Categories**:
1. **Remote Registration** (US1): Register GitHub/GitLab/generic remotes with OAuth, token, SSH auth
2. **Push Operations** (US2): Single push, multi-remote push, auto-push on promote
3. **VFS Mounts** (US3): Mount, fetch, push, branch switch, unmount
4. **Branch Sync** (US4): Sync status display, conflict handling, auto-sync
5. **Agent Tools** (US5): Tool invocation for all operations
6. **Edge Cases**: Auth expiry, naming conflicts, path validation, orphaned mounts
7. **Automated Conflict Resolution** (US6): Rollback tag creation, rebase fallback, DevOps Agent escalation, refresh-resilience, timeout, force-push as a separate confirmed action

### Test Suite 1: Remote Registration (US1)

#### Test 1.1: Register GitHub Remote with OAuth
```typescript
test.describe("Register GitHub Remote with OAuth", () => {
  test("should register a new GitHub remote via OAuth flow", async ({ page, tool }) => {
    // 1. Open Settings → Versions
    await tool.run("open settings versions");
    await page.waitForSelector("[data-testid='versions-tab']");
    
    // 2. Click "Add Remote" button
    await page.getByRole("button", { name: /add remote/i }).click();
    
    // 3. Fill in remote form
    await page.getByLabel("Remote name").fill("github-work");
    await page.getByLabel("Repository URL").fill("https://github.com/test-user/test-repo.git");
    await page.getByRole("combobox").getByText("GitHub").click();
    await page.getByRole("combobox").getByText("OAuth").click();
    await page.getByRole("button", { name: /register/i }).click();
    
    // 4. Wait for OAuth redirect
    await page.waitForURL(/github\.com.*authorize/);
    
    // 5. Simulate OAuth approval (in test environment)
    await page.getByRole("button", { name: /authorize/i }).click();
    
    // 6. Verify remote appears in Settings
    await page.waitForSelector("[data-testid='remote-card']");
    await expect(page.getByText("github-work")).toBeVisible();
    await expect(page.getByText("● Connected")).toBeVisible();
  });
});
```

#### Test 1.2: Register Generic Remote with Token
```typescript
test.describe("Register Generic Remote with Token", () => {
  test("should register a self-hosted git remote with PAT", async ({ page, tool }) => {
    await tool.run("open settings versions");
    await page.getByRole("button", { name: /add remote/i }).click();
    
    await page.getByLabel("Remote name").fill("gitea-self-hosted");
    await page.getByLabel("Repository URL").fill("https://gitea.example.com/user/repo.git");
    await page.getByRole("combobox").getByText("Generic").click();
    await page.getByRole("combobox").getByText("Token").click();
    
    // Enter PAT
    await page.getByLabel("Personal Access Token").fill("glpat-xxxxxxxxxxxxxxxxxxxx");
    await page.getByRole("button", { name: /register/i }).click();
    
    // Verify connection status
    await page.waitForSelector("[data-testid='remote-card']");
    await expect(page.getByText("gitea-self-hosted")).toBeVisible();
    await expect(page.getByText("● Connected")).toBeVisible();
  });
});
```

#### Test 1.3: Register Remote with SSH Key
```typescript
test.describe("Register Remote with SSH Key", () => {
  test("should register a remote with SSH key authentication", async ({ page, tool }) => {
    await tool.run("open settings versions");
    await page.getByRole("button", { name: /add remote/i }).click();
    
    await page.getByLabel("Remote name").fill("ssh-backup");
    await page.getByLabel("Repository URL").fill("git@github.com:user/repo.git");
    await page.getByRole("combobox").getByText("Generic").click();
    await page.getByRole("combobox").getByText("SSH Key").click();
    
    // Paste SSH private key
    await page.getByLabel("SSH Private Key").fill("-----BEGIN OPENSSH PRIVATE KEY-----\n...");
    await page.getByRole("button", { name: /register/i }).click();
    
    // Verify SSH key is stored
    await expect(page.getByText("● Connected")).toBeVisible();
  });
});
```

#### Test 1.4: Test Remote Connection
```typescript
test.describe("Test Remote Connection", () => {
  test("should validate remote connection and show available branches", async ({ page }) => {
    await page.getByRole("button", { name: /test connection/i }).click();
    
    // Wait for fetch response
    await page.waitForSelector("[data-testid='branch-list']");
    
    // Verify branches are listed
    await expect(page.getByText("main")).toBeVisible();
    await expect(page.getByText("develop")).toBeVisible();
  });
});
```

### Test Suite 2: Push Operations (US2)

#### Test 2.1: Single Remote Push
```typescript
test.describe("Single Remote Push", () => {
  test("should push current branch to a specific remote", async ({ page }) => {
    // Click push button on remote card
    await page.getByRole("button", { name: /push/i }).click();
    
    // Wait for push completion
    await page.waitForSelector("[data-testid='push-result']");
    await expect(page.getByText("✓ Pushed to github-work")).toBeVisible();
  });
});
```

#### Test 2.2: Multi-Remote Push
```typescript
test.describe("Multi-Remote Push", () => {
  test("should push to all registered remotes", async ({ page }) => {
    // Click "Push All" button
    await page.getByRole("button", { name: /push all/i }).click();
    
    // Wait for push completion
    await page.waitForSelector("[data-testid='push-summary']");
    
    // Verify all remotes received the push
    await expect(page.getByText("origin: ✓")).toBeVisible();
    await expect(page.getByText("github-work: ✓")).toBeVisible();
    await expect(page.getByText("gitlab-personal: ✓")).toBeVisible();
  });
});
```

#### Test 2.3: Auto-Push on Promote
```typescript
test.describe("Auto-Push on Promote", () => {
  test("should push to auto-push remotes after promote", async ({ tool }) => {
    // Enable auto-push for a remote
    await tool.run("toggle auto-push for github-work");
    
    // Trigger promote
    await tool.run("promote feature-branch");
    
    // Verify push occurred
    await tool.expect("pushed to github-work");
  });
});
```

### Test Suite 3: VFS Mounts (US3)

#### Test 3.1: Mount Repository to VFS Path
```typescript
test.describe("Mount Repository to VFS Path", () => {
  test("should clone and mount a repository to a VFS path", async ({ page, tool }) => {
    // Click "Mount Repository" button
    await page.getByRole("button", { name: /mount repository/i }).click();
    
    // Fill mount form
    await page.getByLabel("Repository URL").fill("https://github.com/user/project.git");
    await page.getByLabel("Target VFS Path").fill("/Projects/my-webapp");
    await page.getByLabel("Branch").fill("main");
    await page.getByRole("button", { name: /mount/i }).click();
    
    // Wait for clone to complete
    await page.waitForSelector("[data-testid='mount-status']");
    await expect(page.getByText("Mounted")).toBeVisible();
    
    // Verify directory exists in VFS
    await tool.run("ls /Projects/my-webapp");
    await tool.expect("package.json");
  });
});
```

#### Test 3.2: Fetch Updates for Mounted Repo
```typescript
test.describe("Fetch Updates for Mounted Repo", () => {
  test("should fetch latest changes from remote", async ({ page }) => {
    await page.getByRole("button", { name: /fetch/i }).click();
    
    // Wait for fetch completion
    await page.waitForSelector("[data-testid='fetch-result']");
    await expect(page.getByText("✓ Fetched 3 commits")).toBeVisible();
  });
});
```

#### Test 3.3: Push Changes from Mounted Repo
```typescript
test.describe("Push Changes from Mounted Repo", () => {
  test("should push local changes to remote", async ({ tool }) => {
    // Create a test file in mounted repo
    await tool.run("write /Projects/my-webapp/test.txt 'hello world'");
    await tool.run("git add /Projects/my-webapp/test.txt");
    await tool.run("git commit -m 'test commit' /Projects/my-webapp/test.txt");
    
    // Push to remote
    await tool.run("git push origin main /Projects/my-webapp");
    
    // Verify push success
    await tool.expect("Pushed to origin");
  });
});
```

#### Test 3.4: Unmount Repository
```typescript
test.describe("Unmount Repository", () => {
  test("should remove VFS directory but preserve cache", async ({ page, tool }) => {
    await page.getByRole("button", { name: /unmount/i }).click();
    
    // Wait for unmount confirmation
    await page.getByRole("button", { name: /confirm/i }).click();
    
    // Verify directory is removed
    await tool.run("ls /Projects/my-webapp");
    await tool.expect("Error: Directory not found");
    
    // Verify cache still exists
    await tool.run("ls data/.git-cache");
    await tool.expect("github.com/user/project.git");
  });
});
```

### Test Suite 4: Branch Sync (US4)

#### Test 4.1: View Sync Status
```typescript
test.describe("View Sync Status", () => {
  test("should display ahead/behind counts for all remotes", async ({ page }) => {
    await page.getByRole("button", { name: /sync status/i }).click();
    
    // Verify sync panel is visible
    await page.waitForSelector("[data-testid='sync-panel']");
    
    // Check specific remote status
    await expect(page.getByText("origin/main: 3 commits behind")).toBeVisible();
    await expect(page.getByText("github-work/main: up to date ✓")).toBeVisible();
  });
});
```

#### Test 4.2: Sync Conflict Resolves Automatically (No Dialog)
```typescript
test.describe("Sync Conflict Resolves Automatically", () => {
  test("should run the reconciliation pipeline without a strategy-choice dialog", async ({ page }) => {
    // Create a conflict resolvable by the scripted rebase fallback (not a real
    // content conflict — the configured strategy is expected to fail cleanly first)
    await tool.run("git checkout -b conflicting-branch");
    await tool.run("git commit --allow-empty -m 'local change'");

    // Trigger sync
    await page.getByRole("button", { name: /sync/i }).click();

    // No blocking confirm dialog is ever shown
    await expect(page.getByTestId("conflict-dialog")).toHaveCount(0);

    // Status resolves to success via the pipeline
    await page.waitForSelector("[data-testid='sync-result']");
    await expect(page.getByText(/up to date/i)).toBeVisible();
  });
});
```

### Test Suite 5: Agent Tools (US5)

#### Test 5.1: Agent Registers Remote
```typescript
test.describe("Agent Registers Remote", () => {
  test("should allow agent to register a new remote", async ({ agent }) => {
    const result = await agent.call("git_add_remote", {
      repoPath: ".",
      name: "github-work",
      url: "https://github.com/user/repo.git",
      provider: "github",
      authType: "oauth"
    });
    
    expect(result.status).toBe("success");
    expect(result.name).toBe("github-work");
  });
});
```

#### Test 5.2: Agent Pushes to Multiple Remotes
```typescript
test.describe("Agent Pushes to Multiple Remotes", () => {
  test("should push to all remotes and return summary", async ({ agent }) => {
    const result = await agent.call("git_push_all_remotes", {
      repoPath: ".",
      remoteNames: ["origin", "github-work"],
      branch: "main"
    });
    
    expect(result.results).toHaveLength(2);
    expect(result.results[0].status).toBe("success");
    expect(result.results[1].status).toBe("success");
  });
});
```

#### Test 5.3: Agent Mounts Repository
```typescript
test.describe("Agent Mounts Repository", () => {
  test("should clone and mount a repository to VFS path", async ({ agent }) => {
    const result = await agent.call("git_mount", {
      repoUrl: "https://github.com/user/project.git",
      targetPath: "/Projects/my-webapp",
      branch: "main",
      authType: "oauth"
    });
    
    expect(result.status).toBe("mounted");
    expect(result.path).toBe("/Projects/my-webapp");
  });
});
```

#### Test 5.4: Agent Resolves Merge Conflict
```typescript
test.describe("Agent Resolves Merge Conflict", () => {
  test("should resolve diverged branches using merge --squash", async ({ agent, page }) => {
    // Create conflicting commits on local and remote
    await tool.run("git commit --allow-empty -m 'local change'");
    await tool.run("git push origin HEAD:conflict-test");
    
    // Agent calls merge tool with --squash
    const result = await agent.call("git_merge", {
      repoPath: ".",
      remote: "origin",
      branch: "conflict-test",
      strategy: "merge-squash"
    });
    
    expect(result.status).toBe("success");
    expect(result.method).toBe("merge --squash");
    
    // Verify commit history shows squashed merge
    await tool.run("git log --oneline -3");
    await tool.expect("Squashed merge of conflict-test");
  });
});
```

#### Test 5.5: Agent Syncs with Conflict Resolution
```typescript
test.describe("Agent Syncs with Conflict Resolution", () => {
  test("should fetch and resolve conflicts via commit strategy", async ({ agent }) => {
    // Create diverged commits
    await tool.run("git commit --allow-empty -m 'local diverge'");
    
    const result = await agent.call("git_sync", {
      repoPath: ".",
      remote: "origin",
      branch: "main",
      conflictStrategy: "commit"
    });
    
    expect(result.status).toBe("success");
    expect(result.method).toBe("commit — stashed remote changes, committed locally");
  });
});
```

#### Test 5.6: Agent Merge Escalates on Unresolvable Conflict
```typescript
test.describe("Agent Merge Escalates on Unresolvable Conflict", () => {
  test("should escalate to the DevOps Agent when strategy and rebase fallback both conflict", async ({ agent }) => {
    // Create a real content conflict unresolvable by strategy or rebase
    await tool.run("write conflicting-file.txt 'local version'");
    await tool.run("git commit -am 'local change'");
    await tool.run("git push origin HEAD:conflict-test-remote-diverges-too");

    const result = await agent.call("git_merge", {
      repoPath: ".",
      remote: "origin",
      branch: "conflict-test-remote-diverges-too",
      strategy: "merge-squash"
    });

    expect(result.status).toBe("escalated");
    expect(result.devopsConversationId).toBeTruthy();

    // The conversation is a normal, persisted, discoverable one
    const conv = await agent.call("assistant_get_conversation", { id: result.devopsConversationId });
    expect(conv.agentId).toBe("devops");
  });
});
```

#### Test 5.7: Agent Handles Auth Error
```typescript
test.describe("Agent Handles Auth Error", () => {
  test("should report clear error when OAuth token expires", async ({ agent }) => {
    // Simulate expired token
    await tool.run("invalidate oauth tokens");
    
    const result = await agent.call("git_push", {
      repoPath: ".",
      remoteName: "github-work",
      branch: "main"
    });
    
    expect(result.status).toBe("failed");
    expect(result.error).toContain("OAuth token expired");
    expect(result.error).toContain("reconnect");
  });
});
```

### Test Suite 6: Edge Cases

#### Test 6.1: Remote Naming Conflict
```typescript
test.describe("Remote Naming Conflict", () => {
  test("should auto-rename remote when name already exists", async ({ page, tool }) => {
    // Create first remote
    await tool.run("git_add_remote origin https://github.com/user/repo1.git");
    
    // Try to create second remote with same name
    await tool.run("git_add_remote origin https://github.com/user/repo2.git");
    
    // Verify auto-renamed
    await tool.expect("renamed to origin-2");
  });
});
```

#### Test 6.2: VFS Path Validation
```typescript
test.describe("VFS Path Validation", () => {
  test("should reject mount outside user VFS", async ({ agent }) => {
    const result = await agent.call("git_mount", {
      repoUrl: "https://github.com/user/repo.git",
      targetPath: "/etc/system",  // Invalid path
      branch: "main",
      authType: "oauth"
    });
    
    expect(result.status).toBe("error");
    expect(result.error).toContain("path must be within");
  });
});
```

#### Test 6.3: Orphaned Mount Removal
```typescript
test.describe("Orphaned Mount Removal", () => {
  test("should warn when removing remote that is mounted", async ({ page }) => {
    // Mount a remote
    await page.getByRole("button", { name: /mount/i }).click();
    
    // Try to remove the remote
    await page.getByRole("button", { name: /remove/i }).click();
    
    // Verify warning appears
    await page.waitForSelector("[data-testid='warning-dialog']");
    await expect(page.getByText("This mount will become orphaned")).toBeVisible();
    await expect(page.getByRole("button", { name: /confirm/i })).toBeVisible();
  });
});
```

### Test Suite 7: Automated Conflict Resolution (US6)

#### Test 7.1: Rollback Tag Created Before Reconciliation
```typescript
test.describe("Rollback Tag Created Before Reconciliation", () => {
  test("should create a tag on HEAD before any merge/rebase attempt", async ({ tool }) => {
    const before = await tool.run("git rev-parse HEAD");

    await tool.run("git_sync . origin main merge-squash");

    const tags = await tool.run("git tag --points-at HEAD~1");
    await tool.expect(/rollback|pre-reconcile/);
  });
});
```

#### Test 7.2: Failed Squash-Merge Aborts Cleanly Before Rebase Fallback
```typescript
test.describe("Failed Squash-Merge Aborts Cleanly", () => {
  test("should reset --hard + clean -fd (not merge --abort) after a failed merge --squash", async ({ tool, agent }) => {
    // merge --squash never sets MERGE_HEAD — verify no lingering squash state
    const result = await agent.call("git_merge", {
      repoPath: ".", remote: "origin", branch: "conflicting-branch", strategy: "merge-squash",
    });

    const status = await tool.run("git status --porcelain");
    // Whatever the outcome (success/escalated), the working tree must not be
    // left mid-squash (no unmerged paths still marked from the failed attempt
    // if a fallback or escalation subsequently proceeded).
    expect(status).not.toContain("Squash commit -- not updating HEAD");
  });
});
```

#### Test 7.3: Unresolvable Conflict Escalates to DevOps Agent
```typescript
test.describe("Unresolvable Conflict Escalates", () => {
  test("should escalate and expose a discoverable, persisted conversation", async ({ agent }) => {
    const result = await agent.call("git_sync", {
      repoPath: ".", remote: "origin", branch: "hard-conflict-branch", conflictStrategy: "merge-squash",
    });

    expect(result.status).toBe("escalated");
    expect(result.devopsConversationId).toBeTruthy();
  });
});
```

#### Test 7.4: Escalation Survives a Browser Refresh
```typescript
test.describe("Escalation Survives a Browser Refresh", () => {
  test("should keep reconciling server-side and show a distinct escalated state after reload", async ({ page }) => {
    // Trigger an escalating promote via the Supervisor control endpoint
    await page.getByRole("button", { name: /promote/i }).click();

    // Simulate a refresh mid-escalation
    await page.reload();

    // The refreshed page shows the escalated state (from polled state), not a
    // generic spinner or an idle "Promote" button
    await page.waitForSelector("[data-testid='promote-escalated']");
    await expect(page.getByRole("link", { name: /view devops conversation/i })).toBeVisible();
  });
});
```

#### Test 7.5: Escalation Timeout Reports a Distinct Outcome
```typescript
test.describe("Escalation Timeout", () => {
  test("should report a distinct timed-out outcome and leave the conversation resumable", async ({ tool }) => {
    // Simulate a DevOps Agent run that never reaches a terminal state within the max wait
    await tool.run("simulate stalled devops run");

    await tool.expect("escalation timed out");
    await tool.run("open devops conversation");
    await tool.expect("Stop"); // still a live, interactable run
  });
});
```

#### Test 7.6: Concurrent Escalation Rejected
```typescript
test.describe("Concurrent Escalation Rejected", () => {
  test("should not start a second pipeline while one is already escalated for the same target", async ({ tool }) => {
    await tool.run("promote feature-branch"); // first attempt escalates
    const second = await tool.run("promote feature-branch"); // repeat click

    await tool.expect(/already in progress|existing conversation/i);
  });
});
```

#### Test 7.7: DevOps Agent Never Pushes or Force-Pushes
```typescript
test.describe("DevOps Agent Never Pushes", () => {
  test("should leave the final push to the caller and never force-push", async ({ agent }) => {
    const result = await agent.call("git_merge", {
      repoPath: ".", remote: "origin", branch: "hard-conflict-branch", strategy: "merge-squash",
    });
    expect(result.status).toBe("escalated");

    // After the escalated run finishes, the branch is committed locally but not pushed
    const ahead = await tool.run("git rev-list --count origin/hard-conflict-branch..HEAD");
    expect(Number(ahead)).toBeGreaterThan(0);
  });
});
```

### Test Execution

**Run Tests**:
```bash
npx playwright test e2e/001-external-repo-integration.spec.ts
```

**Test Coverage**:
- Unit tests for `git-ops.ts`, `auth.ts`, `mount-manager.ts`
- Integration tests for OAuth flow, token refresh
- E2E tests for UI interactions and agent tools
- Edge case tests for error handling

**Mock Strategy**:
- GitHub/GitLab OAuth: Mock with test credentials
- Remote git servers: Use local git server or mock git operations
- VFS: Use test VFS path in `data/vfs/test/`

### Test Results

After running E2E tests, results will be written to:
- `user-specs/001-external-repo-integration/test-results.md`
- `user-specs/001-external-repo-integration/e2e/test-results.md`

