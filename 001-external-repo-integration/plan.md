# Implementation Plan: External Repository Integration

**Branch**: `bos/gitfs-external-repository-support` | **Date**: 2026-07-23 | **Spec**: [bos-system-specs/001-external-repo-integration/spec.md](bos-system-specs/001-external-repo-integration/spec.md)

**Input**: Feature specification from `/specs/001-external-repo-integration/spec.md`

## Summary

Add support for registering external git repositories as remotes on BOS repos (source + apps content), with OAuth (GitHub/GitLab), token, and SSH key authentication. Support multi-remote push, per-remote auto-push on promote, VFS mounting of arbitrary repos, branch sync status with conflict resolution, and full agent tool coverage.

## Technical Context

**Language/Version**: TypeScript (Node.js 20+)

**Primary Dependencies**: Git CLI (system), existing OAuth framework (`src/lib/integrations/oauth/manager.ts`), SecretsStore (AES-GCM)

**Storage**: 
- `data/config/git-remotes.json` — remote metadata (survives promote)
- SecretsStore — OAuth tokens, PATs, SSH keys (encrypted at rest)
- `data/.git-cache/<url-hash>.git` — bare clone cache for VFS mounts
- `data/logs/git-ops.log` — structured audit log (7-day rotation)

**Testing**: Playwright E2E tests, Jest unit tests, integration tests with local git server

**Target Platform**: BrowserOS desktop (Linux/macOS/Windows)

**Project Type**: Desktop application (BOS core)

**Performance Goals**: 
- Push to 5 remotes within 30 seconds
- Mount clone completes within 60 seconds for repos up to 500MB
- Branch sync status fetch within 2 seconds
- 90% OAuth flow success rate

**Constraints**:
- All git operations serialized via `git-lock`
- Conflict resolution runs the shared reconciliation pipeline (US6): rollback tag → configured strategy (merge / merge --squash / commit) → scripted rebase-based fallback on conflict → DevOps Agent escalation if both fail. Rebase IS used (as an automatic fallback, and as the mechanism the Developer sub-agent may use once escalated) — superseding the original "no rebases" restriction.
- Force-push is a separate, always-explicit, always-user-confirmed action (`--force-with-lease`), never part of the automatic pipeline
- VFS mount paths restricted to `data/vfs/` (not under `apps/` or `workflows/`)
- SSH passphrases never via agent tools — secure UI channel only
- `git://` protocol rejected; HTTPS/HTTP (opt-in)/SSH allowlist only

**Scale/Scope**:
- Up to 50 registered remotes
- Up to 20 VFS mounts
- 5 concurrent git operations max (serialized)
- 100MB+ bare cache size expected for large repos

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

[All gates pass — no constitution violations detected]

## Project Structure

### Documentation (this feature)

```text
specs/001-external-repo-integration/
├── spec.md              # Feature specification
├── plan.md              # This file
├── tasks.md             # Implementation tasks
├── e2e/
│   └── 001-external-repo-integration.spec.ts
└── test-results.md
```

### Source Code (repository root)

```text
src/lib/gitops/
├── git-ops.ts           # Core git operations (clone, fetch, push, list branches)
├── auth.ts              # Auth resolution (OAuth, token, SSH) from SecretsStore
├── lock.ts              # git-lock serialization mechanism
├── mounts/
│   ├── mount-manager.ts # VFS mount lifecycle (mount, unmount, fetch, push)
│   └── sync-status.ts   # Sync status computation (ahead/behind)
├── oauth/
│   ├── github.ts        # GitHub OAuth provider config
│   └── gitlab.ts        # GitLab OAuth provider config
└── logging.ts           # Structured logging (levels, sanitization, rotation)

src/lib/gitops/
├── reconcile.ts          # Shared reconciliation pipeline (US6): rollback tag,
│                         # remote-sync, strategy attempt, rebase fallback,
│                         # DevOps Agent escalation trigger — used by git_merge,
│                         # git_sync, and (via HTTP) the Supervisor's promote()
└── rebase-onto-remote.ts # Scripted rebase-based fallback used by reconcile.ts

src/lib/assistant/tools/server/
├── git-remotes.ts       # git_add_remote, git_remove_remote, git_list_remotes
├── git-push.ts          # git_push, git_push_all_remotes
├── git-merge.ts         # git_merge, git_sync — now thin wrappers over reconcile.ts
├── git-mount.ts         # git_mount, git_unmount, git_list_mounts, git_mount_status
└── git-fetch.ts         # git_fetch, git_list_branches

seed/agents/devops/
└── AGENT.md             # DevOps Agent (type: local; tools: dev_delegate + read-only
                          # inspection; skills: devops-merge-conflict-resolution)

seed/skills/devops-merge-conflict-resolution/
└── SKILL.md              # What the DevOps Agent may/may not do (delegate to
                           # Developer, never push, never force-push, verify
                           # build/tests, respect the assigned worktree only)

src/lib/integrations/oauth/
├── github-git.ts        # GitHub git OAuth integration (new)
└── gitlab-git.ts        # GitLab git OAuth integration (new)

src/components/settings/
├── integrations/
│   └── GitProvidersTab.tsx    # OAuth provider management (new)
└── versions/
    └── GitRemotesTab.tsx      # Remote management + mounts + sync status (new)

data/config/
└── git-remotes.json       # Remote metadata (new)

data/.git-cache/             # Bare clone cache (new)
data/logs/
└── git-ops.log              # Structured audit log (new)
```

**Structure Decision**: Modular git operations layer (`src/lib/gitops/`) with clear separation between:
1. **Core git operations** (`git-ops.ts`) — thin wrappers around git CLI
2. **Auth resolution** (`auth.ts`) — SecretsStore-backed credential management
3. **VFS mounting** (`mounts/`) — clone, cache, VFS integration
4. **OAuth providers** (`oauth/`) — GitHub/GitLab OAuth configuration
5. **Agent tools** (`assistant/tools/server/`) — tool implementations for agent interaction
6. **Settings UI** (`components/settings/`) — user-facing management interfaces

## Complexity Tracking

> **No complexity violations** — this feature adds new functionality without breaking existing systems. Git remotes are tracked by git itself (no separate BOS config model), reducing complexity significantly.

## Implementation Phases

### Phase 0: Foundation (git-lock + git-ops + auth)

**Goal**: Core infrastructure for all git operations.

**Deliverables**:
1. `src/lib/gitops/lock.ts` — git-lock serialization
2. `src/lib/gitops/git-ops.ts` — core git operations
3. `src/lib/gitops/auth.ts` — auth resolution from SecretsStore
4. `src/lib/gitops/logging.ts` — structured logging
5. Unit tests for all modules

**Dependencies**: None (foundation layer)

### Phase 1: Git Remotes (Register, Authenticate, Push)

**Goal**: Users can register external remotes and push to them.

**Deliverables**:
1. `data/config/git-remotes.json` — remote metadata storage
2. `src/lib/assistant/tools/server/git-remotes.ts` — remote CRUD tools
3. `src/lib/assistant/tools/server/git-push.ts` — push tools
4. `src/lib/assistant/tools/server/git-fetch.ts` — fetch tools
5. Unit tests + integration tests
6. Settings UI: Git Remotes section in Versions tab

**Dependencies**: Phase 0 (git-lock, git-ops, auth)

### Phase 2: OAuth Providers (GitHub, GitLab)

**Goal**: OAuth authentication for GitHub and GitLab.

**Deliverables**:
1. `src/lib/integrations/oauth/github-git.ts` — GitHub git OAuth integration
2. `src/lib/integrations/oauth/gitlab-git.ts` — GitLab git OAuth integration
3. OAuth callback handling (reuse existing framework)
4. Token refresh logic
5. E2E tests for OAuth flow
6. Settings UI: Git Providers section in Integrations tab

**Dependencies**: Phase 0 (auth, logging), Phase 1 (remote metadata)

### Phase 3: VFS Mounts

**Goal**: Users can mount external repos to VFS paths.

**Deliverables**:
1. `src/lib/gitops/mounts/mount-manager.ts` — mount lifecycle
2. `src/lib/gitops/mounts/sync-status.ts` — sync status computation
3. VFS integration (register mount points)
4. Bare cache management (`data/.git-cache/`)
5. Agent tools: `git_mount`, `git_unmount`, `git_list_mounts`, `git_mount_status`
6. E2E tests for mount/unmount/fetch/push
7. Settings UI: Mount manager in Versions tab

**Dependencies**: Phase 0 (git-lock, git-ops, auth), Phase 1 (remote metadata)

### Phase 4: Branch Sync & Conflict Resolution

**Goal**: Users can see sync status and resolve conflicts.

**Deliverables**:
1. `src/lib/assistant/tools/server/git-merge.ts` — merge/sync tools
2. Conflict resolution UI (merge --squash, merge, commit)
3. Branch sync status panel
4. Auto-push on promote integration
5. E2E tests for sync and conflict resolution
6. Supervisor integration (auto-push triggers)

**Dependencies**: Phase 2 (OAuth), Phase 3 (mounts)

### Phase 5: Polish & E2E

**Goal**: Comprehensive testing and user experience polish.

**Deliverables**:
1. All E2E tests (6 suites, 20+ tests)
2. Performance benchmarks (push speed, mount speed, sync status)
3. Error message polish
4. Documentation (user-facing)
5. Final review

**Dependencies**: All previous phases

## Risk Register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| git-lock deadlock | HIGH | HIGH | Per-repo locking with alphabetical ordering + 30s force-release |
| OAuth token refresh failure | MEDIUM | MEDIUM | Proactive refresh when < 24h remaining; "re-authorize" flow on expiry |
| VFS symlink escape | LOW | HIGH | Post-clone symlink scan + escape replacement |
| Large repo performance | MEDIUM | MEDIUM | Bare cache with LRU eviction; progress indicators |
| Auto-push collision with Supervisor | MEDIUM | HIGH | Single atomic sequence under one lock hold |
| SSH key temp file exposure | LOW | HIGH | `data/.ssh-keys/` with 0700 permissions; `finally` block cleanup |

## Success Metrics

- **SC-001**: Users register GitHub/GitLab remote and push within 3 minutes
- **SC-002**: Mount external repo to VFS path within 5 minutes
- **SC-003**: Multi-remote push to 5 remotes within 30 seconds
- **SC-004**: 90% OAuth flow success rate
- **SC-005**: Branch sync status accurate within 1 minute of fetch
- **SC-006**: Agent tools perform all operations without user intervention (except OAuth)

## Open Questions

1. **Q1**: Where does `git-remotes.json` live? → **AD-001**: `data/config/git-remotes.json` (config namespace, survives promote)
2. **Q2**: What's the exact semantic of `commit` conflict strategy? → **AD-002**: Fetch remote, commit fetched changes, apply local stash on top
3. **Q3**: How does OAuth callback distinguish integrations? → **AD-003**: Use `github-git` and `gitlab-git` as integration IDs
4. **Q4**: Should mounted repos be subject to DataFS isolation? → **AD-004**: Yes — mount paths within `data/vfs/`, NOT under `apps/` or `workflows/`
5. **Q5**: How does bare cache handle force-pushes? → **AD-005**: Force-update bare cache ref on fetch; cache keyed by URL hash
6. **Q6**: Git provider UI location? → **AD-006**: Git Providers in Integrations tab, Git Remotes in Versions tab

All open questions resolved in Architecture Decisions section.
