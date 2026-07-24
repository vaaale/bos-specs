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

6. **Given** a sync operation detects a conflict (local and remote have diverged), **When** the user triggers a sync, **Then** the system presents a conflict resolution dialog offering three resolution strategies: (a) **merge --squash** (recommended — squashes remote commits into a single commit), (b) **merge** (standard merge commit), or (c) **commit** (stash remote changes and commit locally). The system MUST ask the user for confirmation before the agent performs the resolution. The user MUST have the option to abort without resolving.

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

4. **Given** a remote with conflicting commits (both local and remote have diverged), **When** the user attempts to sync, **Then** the system reports the conflict and presents a resolution dialog offering: (a) **merge --squash** (recommended — squashes remote commits into a single commit), (b) **merge** (standard merge commit), or (c) **commit** (stash remote changes and commit locally). The user MUST confirm the resolution before the agent executes it.

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

5. **Given** an agent needs to resolve a branch conflict, **When** it calls `git_merge` with `confirm: true`, **Then** the user is presented with resolution options (merge --squash recommended, merge, commit) and the agent executes only after explicit user approval.

6. **Given** an agent encounters an auth error, **When** it reports the error, **Then** the error includes enough context for the user to fix it (e.g., "OAuth token expired — reconnect via Settings").

---

### Agent Tool Definitions

#### git_merge
```typescript
tool: git_merge
  description: "Resolve a branch conflict by merging remote changes"
  params: {
    repoPath: string,           // e.g. "." or "/Projects/my-webapp"
    remote: string,             // e.g. "origin", "github-work"
    branch: string,             // e.g. "main", "develop"
    strategy: "merge-squash" | "merge" | "commit",
    confirm: boolean            // MUST be true — requires explicit user approval
  }
  returns: {
    status: "success" | "cancelled" | "failed",
    method: string,             // e.g. "merge --squash"
    commitMessage: string,      // if strategy is commit
    error?: { code, message, suggestion? }
  }
```

**Behavior**:
- `strategy: "merge-squash"` → `git merge --squash <remote>/<branch>` then `git commit`
- `strategy: "merge"` → `git merge <remote>/<branch>` (standard merge commit)
- `strategy: "commit"` → `git stash --include-untracked` (save local changes), then commit remote changes, then `git stash pop`
- `confirm: false` → Return `status: "cancelled"` with error message "requires user confirmation"
- MUST serialize via git-lock to prevent concurrent operations

#### git_sync
```typescript
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
    ahead: number,              // commits ahead of remote
    behind: number,             // commits behind remote
    conflict?: {                // if status is "conflict_detected"
      strategy: string,
      requiresConfirmation: boolean
    }
  }
```

**Behavior**:
1. `git fetch <remote> <branch>`
2. Compare `HEAD` vs `<remote>/<branch>`
3. If ahead only → no action needed (report ahead count)
4. If behind only → `git pull --ff-only` (fast-forward)
5. If diverged → status: "conflict_detected" with `requiresConfirmation: true`
6. If `conflictStrategy` is provided and `confirm: true` → execute resolution

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
    Assistant->>Tools: git_merge(".", "origin", "main", "merge-squash", confirm: true)
    Tools->>GitLock: acquire(".")
    Tools->>GitOps: fetch then merge --squash
    GitOps-->>Tools: Merge success
    Tools->>GitLock: release()
    Tools-->>Assistant: Merge result
    Assistant-->>User: "Conflict resolved via merge --squash"
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
- **FR-015**: The per-remote auto-push toggles apply ONLY to non-origin remotes. The Supervisor's `BOS_PUSH_MODE` continues to control the origin push. When promoting, the Supervisor pushes to origin first (if `BOS_PUSH_MODE=auto-on-promote`), then triggers per-remote auto-pushes sequentially.
- **FR-012**: System MUST handle authentication errors gracefully, with clear error messages and suggestions for resolution.

### Key Entities

- **GitRemote**: Represents a remote repository connection. Attributes: name, URL, provider (github/gitlab/generic), authType (oauth/token/ssh), connected status, autoPush flag, remoteBranch (optional branch mapping for push target), defaultBranch (discovered via `git ls-remote --symref`), lastFetched, lastPushed, oauthTokenExpiresAt.
- **VfsMount**: Represents a mounted external repository in the VFS. Attributes: id, vfsPath, repoUrl, branch, authType, status (mounted/stale/error), lastFetched, lastSynced, isBareCache, bare (whether underlying repo is bare).
- **PushResult**: Represents the result of a push operation. Attributes: remoteName, branch, status ("success" | "failed" | "partial_success"), error (structured: { code, message, suggestion? }), timestamp.
- **SyncStatus**: Represents the sync state of a remote or mount. Attributes: localBranch, remoteBranch, ahead, behind, lastFetched.

## Success Criteria

### Measurable Outcomes

- **SC-001**: Users can register a GitHub or GitLab remote and push to it within 3 minutes (including OAuth flow).
- **SC-002**: Users can mount an external repository to a VFS path and push changes within 5 minutes.
- **SC-003**: Multi-remote push completes within 30 seconds for up to 5 remotes on a stable connection.
- **SC-004**: 90% of users successfully complete the OAuth flow on the first attempt.
- **SC-005**: Branch sync status is accurate within 1 minute of fetching remote updates.
- **SC-006**: Agent tools can perform all repository management operations without user intervention (except for initial OAuth authorization).

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

#### Test 4.2: Handle Sync Conflict
```typescript
test.describe("Handle Sync Conflict", () => {
  test("should report conflict when both local and remote have diverged", async ({ page }) => {
    // Create conflicting commits
    await tool.run("git checkout -b conflicting-branch");
    await tool.run("git commit --allow-empty -m 'local change'");
    
    // Trigger sync
    await page.getByRole("button", { name: /sync/i }).click();
    
    // Wait for conflict dialog
    await page.waitForSelector("[data-testid='conflict-dialog']");
    
    // Verify conflict message and resolution options
    await expect(page.getByText("Local and remote have diverged")).toBeVisible();
    await expect(page.getByRole("button", { name: /merge --squash/i })).toBeVisible();
    await expect(page.getByRole("button", { name: /merge/i })).toBeVisible();
    await expect(page.getByRole("button", { name: /commit/i })).toBeVisible();
    await expect(page.getByRole("button", { name: /abort/i })).toBeVisible();
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

#### Test 5.6: Agent Rejects Merge Without User Confirmation
```typescript
test.describe("Agent Rejects Merge Without Confirmation", () => {
  test("should refuse to merge without explicit user approval", async ({ agent }) => {
    // Agent attempts merge without confirmation flag
    const result = await agent.call("git_merge", {
      repoPath: ".",
      remote: "origin",
      branch: "conflict-test",
      strategy: "merge",
      confirm: false  // No user confirmation
    });
    
    expect(result.status).toBe("cancelled");
    expect(result.error).toContain("requires user confirmation");
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

