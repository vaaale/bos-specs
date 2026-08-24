# Implementation Plan: Git Conflict Resolution System (035)

**Branch**: `bos/035-spec-promote-conflict` | **Date**: 2026-08-24 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/Specs/user-specs/core-platform/035-spec-promote-conflict-escalation/spec.md`

## Summary

When a git conflict in a BOS-managed repo (source, user-specs, user-apps, VFS-mounted) cannot be resolved by deterministic steps (merge / rebase), the system MUST escalate to a **user-configurable conflict-resolution agent** (default: DevOps) that works in the *exact repo and worktree* where the conflict was detected, resolving what it can autonomously and asking the user only on genuine ambiguity — through an auto-launched Build Studio pane. This generalizes the existing `reconcile()` escalation (which is hard-wired to the source repo) into a **repo-parameterized** mechanism, adds a first-class persisted **resolution session**, an agent↔user **decision loop**, and converts **every** deterministic-conflict dead-end call site to route through it.

**Technical approach**: generalize `reconcile()`'s step 5 (escalate) with a per-repo working-context object; add a durable session store under `data/gitops/sessions/`; expose `conflict_*` server tools gated to the configured agent; park-and-rewake the agent run across decision requests; emit `com.bos.gitops.conflict.escalated` consumed by a Build Studio UI handler + topbar subscriber; boot-time sweep for restart recovery. Full machinery is specified in [design.md](./design.md).

## Technical Context

**Language/Version**: TypeScript, Node (server) + React (client), Next.js App Router.

**Primary Dependencies**:
- Git plumbing via `src/lib/gitops/git-ops.ts` (`runGitCommand`, `createTag`, `fastForwardMerge`, `isAncestor`, …) — extended with a small `readFileAtRef(repoPath, ref, rel)` helper (`git show <ref>:<rel>` + add/add base-side catch) and a `merge-tree --write-tree` conflicting-file-list parser (the latter already used by `coupledConflicts`).
- 034 event system (`src/lib/events/*`): `emitEvent`, `register-ui-handlers`, `AppManifest.eventHandlers`/`eventNamespaces` grant.
- Assistant run machinery (`src/lib/assistant/start-run.ts`, `agent-loop.ts`, `run-manager.ts`, `tools.ts` `ToolContext`, `gate.ts`): park-and-rewake continues an existing conversation; `serverTool(...)` registration in `src/lib/assistant/tools/server/util.ts`.
- OS window/launch (`src/store/os-store.ts` `launch`), config (`src/lib/config/registry.ts`), atomic write (`src/os/atomic-write.ts` `writeFileAtomic`).

**Storage**: Durable session records under `data/gitops/sessions/<sessionId>.json` (runtime state, gitignored, NOT VFS, NOT a spec store — the correct class of state per constitution V). Conflict snapshot is **derived from git refs** (not stored content) and re-derived on demand / after restart.

**Testing**: Playwright e2e (`e2e/035-*.spec.ts`) for the end-to-end promote→conflict→escalate→auto-launch→answer→resolve flow; unit tests for `reconcile()` generalization, the snapshot-from-refs helper, the session store transitions, and the boot sweep. Self-tests must pass before promotion (constitution IV/VII).

**Target Platform**: BOS server (Linux) + web UI.

**Project Type**: web-service (Next.js app) with server authority + client React.

**Performance Goals**: Escalation → event emit → BS auto-launch perceived within a few seconds of conflict detection (FR-007). Session read is a single small JSON read (sub-ms). The agent run itself is long-lived by design (up to the 25-min escalation timeout; `awaiting-user` waits indefinitely) — no latency budget on that.

**Constraints**:
- `main`/base is **never** left in a conflicted state; a rollback tag is created before any merge/rebase (FR-017).
- The working context (repo identity + worktree path + read/write means) is a **parameter** configured at execution time — **no per-repo special-casing of the access mechanism** (FR-003/FR-004).
- The BS chat reuses the **existing** DevOps conversation as the agent↔user channel — no new communication mechanism (FR-007a, D1).
- The Supervisor is a separate Node process; it reaches the pipeline only via loopback HTTP (`/api/gitfs/reconcile`), never by importing BOS `@/` source (FR-010).
- No `package.json` / lockfile / build-config changes (constitution VII).

**Scale/Scope**: One session per active conflict; the mechanism is repo-agnostic. In scope: 6 call sites (spec Path B table), the session store, the agent tools, the decision loop, auto-launch, restart recovery, Settings field, and the 4 FR-019 UI surfaces.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Verdict | Notes |
|---|---|---|
| I. Spec-Driven | ✅ PASS | `spec.md` exists and is agreed; `design.md` + this plan derive from it. |
| II. Server Authority & SSR Boundary | ✅ PASS | `conflict_*` tools are `execution: "server"`; session API lives in `src/app/api/**`; the working-context repo path + git ops run server-side and are **never** exposed to the client; no secrets to the client. |
| III. Always Delegate; Claude Codes | ✅ PASS | Implementation delegated to the Developer (Claude) sub-agent via `dev_delegate`. |
| IV. Minimize Blast Radius | ✅ PASS | All changes on `bos/035-spec-promote-conflict`; candidate self-tests before promote. |
| V. The VFS Is Not the Source | ✅ PASS | Session store is runtime state under `data/` (not VFS, not a spec store). Repo edits happen in the managed worktrees (source/spec-store/user-apps) via the git layer, never via VFS file tools. |
| VI. Specs & Docs Stay in Sync | ✅ PASS (task T-docs) | Adds `docs/dev/features/git-conflict-resolution.md` and a note in `docs/dev/build-studio.md` (Settings field + conflict pane). Recorded as an explicit task. |
| VII. Respect Boundaries | ✅ PASS | No `package.json`/lockfile/build changes. The `devops` seed tool-list change is a source edit to the bundled seed (see Risk R-upgrade below). Quality gates (`tsc --noEmit`, `lint`) run in `tasks`. |

**Gate result**: PASS — no unjustified violations. (No Complexity Tracking table required.)

## Project Structure

### Documentation (this feature)

```text
specs/user-specs/core-platform/035-spec-promote-conflict-escalation/
├── spec.md            # agreed specification (App Target: bos-core)
├── design.md          # v2 — machinery, refs-based snapshot, tool schemas, call sites
├── design-review.md   # architect-reviewer verification (M1, M2, S1–S4)
├── plan.md            # THIS file
├── research.md        # decision log (refs snapshot vs live stages, park-and-rewake, etc.)
├── data-model.md      # ConflictSession / ConflictHunk / ConflictDecision + state machine
├── quickstart.md      # end-to-end validation scenarios (drives the e2e phase)
└── mockup.html        # accepted UI contract (Build Studio conflict pane)
```

### Source Code (concrete file map — from design.md §3.3, verified against source)

```text
# --- Session store (NEW) ---
src/lib/gitops/sessions/
├── store.ts          # create/read/update/transition; data/gitops/sessions/<id>.json via writeFileAtomic
└── types.ts          # ConflictSession, ConflictSnapshot, ConflictHunk, ConflictDecision, status enum

# --- Snapshot helper (NEW, general gitops) ---
src/lib/gitops/git-ops.ts   # + readFileAtRef(repoPath, ref, rel); + merge-tree conflicting-file-list parser

# --- Pipeline generalization (MODIFY) ---
src/lib/gitops/reconcile.ts # step 5: build per-repo working context; create session; tag conversation
                            #   with conflictSessionId; read build-studio.conflictAgent; emit event;
                            #   extend inFlightEscalations value to carry sessionId + awaiting-user guard

# --- Session API (NEW) ---
src/app/api/gitops/sessions/route.ts      # GET list/get, POST create, PATCH transition (server authority)

# --- Conflict-resolution agent tools (NEW, server tools, gated) ---
src/lib/assistant/tools/server/conflict-resolve.ts  # conflict_read, conflict_write, conflict_decision,
                                                   #   conflict_status, conflict_complete, conflict_abandon
src/lib/assistant/registry.ts   # spread the new module into combined server tools
src/lib/agent/conversations-server.ts  # + getConversationConflictSessionId(conversationId) (mirror activeFeatureBranch)

# --- Agent seed (MODIFY) ---
src/.../devops seed AGENT.md + tools array   # add the six conflict_* tool ids (exact path pinned in design.md §-S4)
src/lib/agent/... store.ts                   # backfill tools array for a pre-existing data/agents/devops (R-upgrade)

# --- Config (MODIFY) ---
src/lib/config/registry.ts   # build-studio namespace load(): + conflictAgent (default "devops")
src/components/apps/settings/BuildStudioTab.tsx   # second dropdown "Conflict resolution agent" (FR-025)

# --- Auto-launch (NEW + MODIFY) ---
src/components/desktop/ConflictLaunch.tsx  # NEW topbar subscriber (sibling of EventBell): on event → launch("build-studio",{pane:"conflict",sessionId})
src/apps/build-studio/manifest.json        # + eventHandlers: com.bos.gitops.conflict.escalated; + eventNamespaces grant
src/apps/build-studio/index.tsx            # read pane/sessionId param → render the conflict pane; existing chat re-pointed at the session conversation

# --- Build Studio conflict pane (NEW) ---
src/apps/build-studio/conflict/            # ConflictPane (header/status, files list, 3-way view, per-hunk controls, rollback)

# --- Call-site conversions (MODIFY) ---
tools/supervisor/lib/coupled-repos.mjs    # coupledConflicts (pre-check: create+escalate instead of throw); promoteCoupled (route through pipeline) [FR-012a]
tools/supervisor/lib/app-candidate.mjs    # appPromote (route through pipeline) [FR-015]
tools/supervisor/lib/promote.mjs          # return sessionId in promote response (FR-018); emit via loopback
src/lib/specs/promote.ts                  # promoteFeature (route through pipeline w/ user-specs context) [FR-012]
src/app/api/git-remotes/route.ts          # case "fetch" + case "push" recovery (route through pipeline, return sessionId) [FR-013/014]

# --- Restart recovery (NEW) ---
src/instrumentation.ts                     # + recoverSessions() boot hook (re-emit for working sessions whose run is dead; restore awaiting-user)

# --- FR-019 surfaces (MODIFY) ---
src/components/desktop/VersionControls.tsx
src/components/apps/settings/VersionsTab.tsx
src/components/apps/settings/versions/ConflictResolutionDialog.tsx
src/components/apps/settings/versions/GitRemotesTab.tsx

# --- Docs (ADD/EDIT, principle VI) ---
docs/dev/features/git-conflict-resolution.md   # NEW
docs/dev/build-studio.md                        # + Settings field + conflict pane note
```

**Structure Decision**: single Next.js project. New code is grouped by concern (`gitops/sessions`, `assistant/tools/server`, the Build Studio `conflict/` pane) and reuses the existing gitops/events/assistant/config subsystems — no new package, no new process.

## Interface Contracts (summary — full schemas in design.md §3/§7/§8, data in data-model.md)

- **Agent tools** (server, gated to the configured agent): `conflict_read(file?)`, `conflict_write(file, resolution)`, `conflict_decision(question, options, file?, hunk?)`, `conflict_status(file?)`, `conflict_complete(summary)`, `conflict_abandon(reason?)`. Each takes `repo_path` (and reads `conflictSessionId` from `ctx.conversationId`).
- **Session API** (`/api/gitops/sessions`): `GET ?id=` / `GET` (list non-terminal) / `POST` (create) / `PATCH ?id=` (status transition, decision answer).
- **Event**: `com.bos.gitops.conflict.escalated` — payload `{ sessionId, repoLabel, featureBranch, rollbackTag, repoKind }`. BS declares a UI handler; topbar subscriber auto-launches.
- **Config**: `build-studio.conflictAgent` (default `"devops"`), read at escalation time.
- **Promote/git responses**: gain `sessionId` on a conflict escalation (FR-018).

## Risks (carried from design.md + review; owned in tasks)

- **R-upgrade**: `applySeedAgent` only seeds a missing `data/agents/devops/AGENT.md`; a pre-existing one won't auto-gain the `conflict_*` tools → the default path (FR-023) silently `failed`-sessions. Mitigation: a backfill in the agent store (or a versioned seed check). **Must be a task.**
- **add/add base side**: `git show <merge-base>:<rel>` throws → catch → empty. The repro case is an add/add; covered by a unit test.
- **awaiting-user guard**: `inFlightEscalations` must treat a parked `awaiting-user` session as "still active" for the concurrent-op guard without re-escalating it.
- **plumbing paths** (no worktree): the least-tested branch; the refs-based snapshot is exactly what makes these work — covered by e2e.

## Out of Scope

- Changing the 034 event system's core dispatch (we emit + declare a UI handler only).
- A per-repo conflict agent (one user-configurable agent resolves all repos, FR-025).
- Non-git conflict resolution (bundled-asset keep/replace conflicts are a separate, already-resolved mechanism).
