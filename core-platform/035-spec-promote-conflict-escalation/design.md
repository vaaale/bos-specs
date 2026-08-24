# Design — Git Conflict Resolution System (035)

**Feature Branch**: `035-spec-promote-conflict`
**Target**: `bos-core` (agrees with `spec.md`'s `App Target`)
**Binding UI contract**: `mockup.html` in this directory (D8). This design does NOT redesign the UI — it specifies the machinery that powers the pane.

---

## 1. Classification

**`bos-core`.** This is a cross-cutting system capability: new server-side plumbing in `src/lib/` (a session store, a set of repo-scoped agent tools, a generalized escalation), API routes, a generalization of the existing `reconcile()` pipeline, a new pane inside the *existing* `build-studio` built-in app, and a settings field. Nothing here is a new self-contained installable item, and the resolution pane lives inside the existing built-in `build-studio` app (`src/apps/build-studio/`) rather than as a new `src/apps/<id>/` window. The spec's `App Target: bos-core` is correct; no disagreement to reconcile.

Note the one place that *looks* like it could be a `marketplace-item`: the user-apps repo is GitFS content. But this feature does not create or modify user-apps content — it only makes a *conflict* that already arises in the user-apps repo route through the same core pipeline. So the whole feature is `bos-core`.

---

## 2. Constitution check

- **II. Server Authority & SSR Boundary** — all git work, the session store, and the agent tools are server-only (`"server-only"`); the UI talks to new `/api/gitops/sessions/*` routes. ✔
- **IV. Minimize Blast Radius** — every repo is only ever touched after a rollback tag is created (`reconcile.ts` step 1, `createTag` → `git tag -a`); the pipeline is already feature-branch-scoped. The new session store is additive under `data/` (gitignored runtime state, Constitution V). ✔
- **V. The VFS Is Not the Source** — the session store is runtime state under `data/gitops/sessions/`, *not* under `data/vfs/` (the user sandbox) and *not* a git-versioned spec store. It is the same class of state as the config store (`data/config/<ns>.json`) and the event store. ✔
- **VI. Specs & Docs Stay in Sync** — implementation must update `docs/dev` (reconcile/pipeline doc, Build Studio doc) in the same change. Flagged for `plan.md`.

No constitution conflicts. The one thing to watch (recorded in §11 Risks) is that "auto-launch" (FR-007) is a *small extension* of the 034 event system's UI-handler model, which is currently click-resolved — that extension must be scoped so it does not silently change 034's semantics.

---

## 3. Architecture (Context / Container / Component)

### 3.1 Context

A git conflict is detected by any of: a Build-Studio promote (`promoteFeature`), a Supervisor feature-promote (its coupled-repo merge + pre-check), a Supervisor app-candidate promote, a "Pull" or "Push-recovery" in Settings → Versions, or a future VFS-mount op. Instead of dead-ending with a static `resolve manually` error, the detection site calls **one canonical entry point** that (a) creates a persisted **resolution session**, (b) launches the **conflict-resolution agent** run with the session's working context, and (c) **auto-launches/focuses Build Studio's conflict pane** via an event. The agent resolves what it can, asks the user (through the existing DevOps conversation) when it can't, and when everything is resolved the **underlying operation completes**. The user can always abandon → rollback.

### 3.2 Container (BOS's real containers)

| Container | Role in this feature |
|---|---|
| **Next.js app process** (`next start`/`dev`) | Owns the session store, the conflict agent tools, the `reconcile()` pipeline, the API routes, and the agent run loop (`runManager`). The agent run is *in-process* here — it dies on restart (see §9). |
| **The agent run** (`runManager` → `runAgentLoop`) | The conflict-resolution agent executes inside a server-owned run attached to a persisted conversation (`/Documents/Chats/<id>.json`). Park-and-rewake (one run per decision turn). |
| **Supervisor** (`tools/supervisor/`, separate Node process) | Detects user-apps + coupled-repo conflicts during promote. It does **not** import BOS source; it reaches the generalized mechanism over **loopback HTTP** to `/api/gitfs/reconcile` (existing `reconcile-client.mjs`) — the same trust model as `/api/health`. |
| **Browsers (N)** | The Build Studio conflict pane (a client) + the topbar auto-launch subscriber. N viewers over the run's NDJSON event stream + the session store; never N executors. |
| **The managed repos** | source (`getSourceRepoRoot()`), each spec store (`data/specs/<store>` + worktrees at `<dataDir>/specs/.worktrees/<encoded-branch>`), user-apps (`dataDir()/user-apps`). The **working context** is the only per-repo variable. |

```mermaid
graph TD
  subgraph Detector["Detection (any repo)"]
    P1["promoteFeature (spec store worktree)"]
    P2["Supervisor coupledConflicts/promoteCoupled (loopback)"]
    P3["Supervisor appPromote (loopback)"]
    P4["/api/git-remotes fetch + push-recovery"]
  end
  EP["Canonical entry: startConflictSession + reconcile.escalate"]
  SES[("Session store  data/gitops/sessions/<id>.json")]
  AGT["Conflict-resolution agent run  (DevOps conversation)"]
  TOOLS["conflict_* server tools (repo-scoped)"]
  EV["com.bos.gitops.conflict.escalated"]
  BS["Build Studio conflict pane + chat  (DevOps conversation)"]
  SUB["Topbar auto-launch subscriber"]

  P1 --> EP
  P2 -->|loopback /api/gitfs/reconcile| EP
  P3 -->|loopback /api/gitfs/reconcile| EP
  P4 --> EP
  EP -->|create + snapshot + tag| SES
  EP -->|launch run w/ context| AGT
  EP -->|emit| EV
  AGT -->|conflict_read/write/decision| TOOLS
  TOOLS -->|validate vs| SES
  TOOLS -->|write working tree / record resolved| SES
  EV --> SUB
  SUB -->|launch('build-studio',{pane:'conflict',sessionId})| BS
  BS -->|openConversation(devopsConvId)| AGT
  BS -->|GET /api/gitops/sessions/:id + stream| SES
  BS -->|POST /api/gitops/sessions/:id/decision| EP
  EP -->|re-launch run (rewake)| AGT
```

### 3.3 Component (files this feature creates/modifies)

Only real paths. (Existing files the design *calls into* but does not create are in §6 Integration points.)

**New**
- `src/lib/gitops/conflict-session.ts` — the resolution-session store + state machine + snapshot capture + completion/abandon/rewake logic (see §4, §5, §9).
- `src/lib/assistant/tools/server/conflict-resolve.ts` — the `conflict_*` server tools (see §7).
- `src/app/api/gitops/sessions/route.ts` — `GET` (list, `?status=`/`?repo=`), `POST` (create, server-only).
- `src/app/api/gitops/sessions/[id]/route.ts` — `GET` (full session), `DELETE` (abandon/rollback).
- `src/app/api/gitops/sessions/[id]/decision/route.ts` — `POST` (user decision; re-wakes the agent).
- `src/apps/build-studio/ConflictPane.tsx` — the conflict-resolution pane (renders the mockup). New file in the existing built-in app.

**Modified**
- `src/lib/gitops/reconcile.ts` — add the `escalate` step that creates a session, captures the snapshot, emits the event, and launches the run with the configurable agent (replace hard-coded `DEVOPS_AGENT_ID`; see §5, §8).
- `src/lib/specs/promote.ts` — `promoteFeature` routes its conflict through the session pipeline (FR-012).
- `src/app/api/git-remotes/route.ts` — `case "fetch"` (FR-013) and `case "push"` recovery (FR-014) route conflicts through the session pipeline.
- `tools/supervisor/lib/coupled-repos.mjs` — `promoteCoupled` (working-tree + plumbing) routes conflicts through the pipeline over loopback (FR-015, the *reported bug's* site).
- `tools/supervisor/lib/app-candidate.mjs` — `appPromote` routes its merge conflict through the pipeline over loopback (FR-015).
- `src/apps/build-studio/manifest.ts` — declare the `com.bos.gitops.conflict.escalated` UI handler + `eventNamespaces` grant (FR-007).
- `src/apps/build-studio/index.tsx` — host the `ConflictPane` in the center column while a session is active (D5), read `params` to know which session, wire the existing chat to the DevOps conversation (FR-007a).
- `src/components/apps/settings/BuildStudioTab.tsx` — add the **Conflict resolution agent** field (`build-studio.conflictAgent`, default `"devops"`) (FR-025).
- `src/lib/config/registry.ts` — `build-studio` namespace `load()` returns `conflictAgent` (default `"devops"`) alongside `agent`.
- `src/components/desktop/EventBell.tsx` (or a new sibling in `src/components/desktop/`) — the auto-launch subscriber that listens to the event stream and launches BS (FR-007).
- `src/instrumentation.ts` (or the equivalent boot hook) — boot-time session-recovery sweep (FR-024).
- The `devops` agent definition (seeded agent under `data/agents/` or the subagents store) — add the `conflict_*` tools to its tool allowlist so it can call them (FR-025, §11).
- `src/lib/gitops/reconcile-jobs.ts` — thread `sessionId` through `ReconcileJob` so the Supervisor's poller can see it (FR-018).

---

## 4. The resolution session (FR-002)

### 4.1 Storage

`data/gitops/sessions/<sessionId>.json`, written with the atomic-write primitive (`src/os/atomic-write.ts`) — the same pattern as the config store and the persisted conversation files. Chosen over the VFS (it's system runtime state, not user content — Constitution V) and over a spec store (it's not versioned user-authored content; it must not be branch-coupled). It survives a process restart (FR-002) because it is a plain file under `data/`.

An in-process warm index (`globalThis`-backed `Map<sessionId, session>`, hot-reload-safe like `runManager`/the event kernel) holds live sessions; every transition writes the file. On boot, the sweep (§9) re-reads all non-terminal session files into the index.

### 4.2 Persisted schema

```ts
// src/lib/gitops/conflict-session.ts
type SessionStatus = "working" | "awaiting-user" | "resolved" | "failed" | "timed-out" | "abandoned";

interface WorkingContext {
  /** "source" | "user-specs" | "user-apps" | "vfs-mount" | "generic". */
  repoKind: string;
  /** Absolute working-tree path the agent reads/writes. */
  worktreePath: string;
  /** The repo root (may differ from worktreePath for a linked worktree). */
  repoRoot: string;
  /** The branch being reconciled (the "ours"/feature side). */
  branch: string;
  /** The ref being merged in (the "theirs"/base side) — a local ref or <remote>/<branch>. */
  sourceRef: string;
  /** "working-tree" (agent edits the live tree) or "plumbing" (no live tree; resolve from snapshot). */
  mode: "working-tree" | "plumbing";
  /** Optional Supervisor feature branch, for the source path (dev_delegate parity, FR-023 regression). */
  featureBranchForDelegate?: string;
}

interface ConflictHunk {
  hunkIndex: number;
  base?: string;   // merge-base side (undefined for add/add)
  ours: string;    // branch side
  theirs: string;  // sourceRef side
}

interface ConflictFile {
  path: string;               // repo-relative
  binary: boolean;            // FR-022
  conflictType: "add/add" | "modify/modify" | "delete/modify" | "add/modify";
  hunks: ConflictHunk[];
  /** Filled as the agent/user resolves: the final merged content. */
  resolvedContent?: string;
  /** "agent" | "user" | "unresolved". */
  resolvedBy?: "agent" | "user" | "unresolved";
}

interface Decision {
  id: string;
  ts: number;
  path: string;
  hunkIndex?: number;
  question: string;                       // the agent's question text
  options: string[];                      // ["theirs","ours","both","suggestion","manual"]
  suggestion?: string;                    // agent's suggested merged content, if any
  answer?: string;                        // user's chosen option (undefined while awaiting)
  answeredAt?: number;
}

interface ResolutionSession {
  id: string;
  createdAt: number;
  updatedAt: number;
  workingContext: WorkingContext;
  rollbackTag: string;                    // FR-017 — always set before any merge
  conversationId: string;                 // the DevOps/conflict-agent conversation
  agentId: string;                        // the conflictAgent read at escalation (FR-025)
  status: SessionStatus;
  statusReason?: string;                  // e.g. the error message on "failed"
  files: ConflictFile[];
  decisions: Decision[];                  // the decision timeline (FR-002)
  lastWorkingAt: number;                  // for the working-phase timeout (§9.3)
  /** What the operation completion must do (see §5.3). */
  completion: {
    kind: "ff-main" | "commit-merge" | "plumbing-merge" | "none";
    repoPath: string;
    /** for ff-main: the branch to fast-forward base to. */
    ffBranch?: string;
  };
}
```

`status` is the state machine from the spec: `working → awaiting-user → (working | resolved | failed | timed-out)`; `abandoned` is a terminal the user triggers. `awaiting-user` is **non-terminal** and parks **indefinitely** (D3) — only the *working* phases have the 25-min budget.

**How the snapshot is captured (the tree is NOT conflicted at escalation time).** The `reconcile()` pipeline *aborts* the merge/rebase the moment a conflict is detected (reconcile.ts step 3/4: `merge --abort` / `reset --hard` + `rebase --abort` + stray-state removal) *before* step 5 escalates — so by the time `escalate()` runs, the working tree is **clean** and the `:1/:2/:3` conflict stages are gone. The snapshot is therefore captured **from the three refs**, not from live stages:
- `base` = `git merge-base <branch> <sourceRef>` (merge-base commit);
- `ours` = `<branch>` (the feature side — HEAD of the worktree);
- `theirs` = `<sourceRef>` tip (the base/remote side being merged in).

Per conflicting file: read each side via `git show <ref>:<rel>` (the exact primitive `store-git.ts` `readFileAtRef` already uses) and derive the conflicting-file set + hunk boundaries from `git merge-tree <branch> <sourceRef>` (or `git diff --merge-base=<base> <branch> <sourceRef>`). A file present on only one side is an `add/add` or `add/modify` (base side empty — matching the mockup's `base: (empty)`). This is deterministic, restart-safe (refs + the recorded `sourceRef`), and works for **both** the working-tree and plumbing cases (plumbing has no working tree at all — refs are the only source). The session records the merge-base sha + `branch` + `sourceRef` so a resumed session re-derives identical content.

### 4.3 API routes

| Route | Method | Purpose |
|---|---|---|
| `/api/gitops/sessions` | GET | List sessions (`?status=non-terminal`/`?repo=`). Drives the pane's "which session is active?" query on launch/refresh (FR-007a, FR-024). |
| `/api/gitops/sessions` | POST | Create (server-only; the reconcile pipeline uses it in-process, not over HTTP). |
| `/api/gitops/sessions/:id` | GET | Full session (the pane renders from this). |
| `/api/gitops/sessions/:id` | DELETE | Abandon → rollback via `rollbackTag`, close as `abandoned` (FR-011, US5). |
| `/api/gitops/sessions/:id/decision` | POST | `{ decisionId, answer, editedContent? }` — records the user's answer, transitions `awaiting-user → working`, and **re-launches the agent run** with the answer (§5.2). |

The agent writes the session only *in-process* through the conflict tools (§7) — it never hits these HTTP routes. The browser hits them. The two writers (in-process tool handlers, HTTP decision route) are serialized by the warm index (single process); the file is the durable record.

---

## 5. Generalizing `reconcile()` and the escalation

### 5.1 What changes in `reconcile.ts`

The 5-step pipeline (rollback tag → sync → strategy → rebase fallback → escalate) is unchanged in shape. Step 5 (`escalate`) is generalized:

**Before** (source-repo-only, hard-coded): builds a task string mentioning `repoPath` + `featureBranchForDelegate`, creates a DevOps conversation hard-coded to `DEVOPS_AGENT_ID = "devops"`, calls `startAssistantRun`, and blocks on `waitForRun`.

**After**: step 5 calls a new `escalate()` that:
1. **Reads the configured agent** — `const agentId = await getConfigValue("build-studio","conflictAgent") ?? "devops"` (FR-025), replacing the `DEVOPS_AGENT_ID` constant. (Source-repo regression, FR-023: default is still `"devops"` and the conversation still pre-sets `activeFeatureBranch` = `featureBranchForDelegate`, so the source path behaves identically.)
2. **Captures the conflict snapshot** from the three **refs** (merge-base + `branch` tip + `sourceRef` tip — see §4.2 note: the pipeline has already *aborted* the merge/rebase, so the tree is clean, not conflicted) → `files[]`.
3. **Creates the session** (`workingContext` built from the repo — §7.1), `rollbackTag` from step 1, `conversationId` from a new conflict-agent conversation (generalized `createDevOpsConversation` to take `agentId`).
4. **Tags the conversation** with `conflictSessionId` (conversation metadata, analogous to the existing `activeFeatureBranch`) so the conflict tools can find the session by `ctx.conversationId` (§7.2).
5. **Emits `com.bos.gitops.conflict.escalated`** with `{ sessionId, repo, branch, conversationId, summary }` (§8).
6. **Launches the run** via `startAssistantRun({ conversationId, agentId, message: task })` — the task now says *"a resolution session `<id>` is open; use the conflict_* tools against your working context; resolve what you can, call `conflict_decision` when you need the user, call `conflict_complete` when done."*
7. Returns to the caller **non-blocking** with `ReconcileOutcome = { status: "escalated", rollbackTag, devopsConversationId, sessionId }` (FR-018 — the response carries the session id *and* the conversation id).

**`ReconcileOptions` gains** `workingContext?: WorkingContext` (or the raw `repoKind`/`worktreePath`/`mode` fields) and `onEscalate?: (conversationId, sessionId) => void` (the existing `onEscalate` gains the `sessionId`). `ReconcileOutcome` gains `sessionId?: string`.

### 5.2 The agent↔user decision loop (FR-006, FR-009, FR-010) — park-and-rewake

**The crux, resolved: the agent PARKS (its run ends) on a decision request, and is RE-WOKEN by a new run when the user answers.** It does *not* block the loop on an indefinitely-parked promise. Rationale, grounded in the real run mechanics:

- The run loop (`agent-loop.ts`) is a bounded, in-process loop; `runManager` holds at most one active run per conversation and the run is in-memory (dies on restart). A run that blocks forever on `awaitFrontendResult` (the existing mechanism used by `elicit`/`dev_branch_request`) **cannot survive a restart** and would pin the conversation's run slot indefinitely — incompatible with D3 (park indefinitely) + FR-024 (restart recovery).
- Park-and-rewake reuses the **exact existing channel** (D1): the DevOps conversation. The agent's question is a normal assistant message + a typed tool call; the user's answer is submitted through the pane and re-launches a run whose first user message *is the answer*. The transcript (persisted at `/Documents/Chats/<id>.json`) is continuous, so the agent "continues the same conversation" (the spec's "continues the same run" is satisfied at the conversation level — a single in-process run genuinely cannot park indefinitely and resume).

**Flow, with the real symbols:**

1. **Agent asks.** Agent calls the `conflict_decision` server tool (§7) with `{ path, hunkIndex?, question, options, suggestion? }`. The tool handler:
   - appends a `Decision` (unanswered) to `session.decisions`,
   - sets `session.status = "awaiting-user"`, persists,
   - publishes a session-stream event (so the pane flips to `awaiting-user` — the amber card),
   - **returns** `JSON.stringify({ parked: true, decisionId, message: "Waiting for the user's decision. End your turn; you will be resumed with their answer." })`.
2. **Run ends.** The agent sees `parked:true` and ends its turn (no more tool calls) → `runAgentLoop` returns `{ reason: "completed" }` → `runManager.finish`. Because the session is `awaiting-user` (non-terminal), the pipeline does **not** complete the operation (§5.3). This run is cheap and already settled — nothing is pinned.
3. **User answers.** The pane (which is bound to the conversation, so the agent's question card is visible in the *existing* chat — FR-009) calls `POST /api/gitops/sessions/:id/decision` with `{ decisionId, answer, editedContent? }`. The route:
   - records `answer`/`answeredAt` on the `Decision`, applies it to `session.files` (the `resolvedContent`/`resolvedBy:"user"` for that file/hunk),
   - sets `session.status = "working"`, `lastWorkingAt = now`, persists,
   - **re-launches the run**: `startAssistantRun({ conversationId, agentId, message: \`The user answered decision \`${decisionId}\` with \`\${answer}\`…\` })` (FR-010 — the agent continues).
4. **Agent continues.** New run, same conversation → the agent reads the transcript (sees the answer), proceeds to the next file. When all files are resolved it calls `conflict_complete`; on a hard failure it calls `conflict_abandon` (or its run errors → session `failed`).

**The per-hunk controls (FR-010) and the chat decision card are the same answer.** The pane's per-file/per-hunk buttons (accept theirs/ours/keep-both/edit/accept-suggestion) and the chat's decision card both POST to the *same* `/decision` route with the matching `decisionId`. Answering from either surface is one code path. (The mockup's `data-decide` and `data-qopt` handlers both map to this.)

### 5.3 Operation completion (resolving → the underlying op finishes; FR-011)

The session carries a `completion` plan (§4.2) captured at creation. When a transition makes the session terminal, the session store's completion handler performs the op:

- `resolved` → **complete the merge**:
  - `working-tree`: the pipeline already **aborted** the merge, so there is no in-progress merge to continue — the worktree is on `<branch>` at its pre-merge tip. The completion: write each `files[].resolvedContent` to its path in `worktreePath`, `git add -A` + a single `git commit` recording the resolution, then `completion.kind`: `ff-main` → `git merge --ff-only <branch>` on the repo root (FR-017 — main fast-forwards; it was never conflicted), `commit-merge` → nothing more. Then prune the worktree + delete the feature branch (promote cleanup).
  - `plumbing` (Supervisor busy-checkout case): build a tree from `files[].resolvedContent` via `git commit-tree` (two parents = base tip + branch tip) and `git update-ref refs/heads/<base>` — the working directory is untouched (this is `promoteCoupled`'s existing plumbing shape, generalized to consume resolved content).
- `failed` / `timed-out` / `abandoned` → **rollback**: `git reset --hard <rollbackTag>` (+ `git merge --abort` / clear rebase state as needed) in `worktreePath`; main is never left conflicted (FR-017). The op reports failure with the rollback tag + conflicting files.

Because completion is owned by the session store (in the Next.js process), it runs correctly even when the *detector* was the Supervisor (a separate process) — the Supervisor's reconcile job just sees the terminal outcome when it polls.

---

## 6. Integration points (existing mechanisms this design calls into — not deliverables)

- **`startAssistantRun`** (`src/lib/assistant/start-run.ts`) + **`runManager`** (`run-manager.ts`) — the run launch/rewake. No changes; called with the conflict conversation + agent.
- **`runAgentLoop`** (`agent-loop.ts`) — the loop the agent runs in. Unchanged.
- **`getConfigValue("build-studio", …)`** (`src/lib/config/registry.ts`) — read the `conflictAgent` at escalation (FR-025). The namespace already exists; only `load()` gains the field.
- **The event system** (`src/lib/events/*`): `api.emit` (in-process) to emit `com.bos.gitops.conflict.escalated`; `registerAllUiHandlers`/`registerAppUiHandlers` (`register-ui-handlers.ts`) surfaces the BS manifest handler; the **NDJSON stream** (`/api/events/stream`, client `subscribeEventStream`) is what the auto-launcher and the pane subscribe to; `ownsNamespace`/`eventNamespaces` grant (see §8). No core dispatch changes.
- **`runGitCommand`** (`src/lib/gitops/git-ops.ts`) — the generic raw-git escape hatch the conflict tools and snapshot capture use (`git merge-base`, `git show <ref>:<rel>`, `git merge-tree` / `git diff --merge-base`, `git add`, `git commit`, `git merge --ff-only`, `git commit-tree`, `git update-ref`, `git reset --hard`).
- **`gitLock()`** (`src/lib/gitops/lock.ts`) — every session mutation that touches a repo takes the per-`repoPath` lock (the existing `inFlightEscalations` guard, generalized to all repo paths — the concurrency invariant, US5).
- **`getGitFsInstance` / `getSourceRepoRoot`** (`src/lib/gitops/filesystems.ts`) — resolve a repo id to its absolute root for the working context, so the Supervisor's loopback caller and the in-process callers agree on paths (the Supervisor never hard-codes a BOS path).
- **The Build Studio app + `AssistantChatV2` + `openConversation`/`attachToRun`** (`run-client.ts`) — the pane reuses the existing chat bound to the DevOps conversation, and re-attaches to a live run on (re)load. Unchanged; the pane drives it.
- **`launch(appId, params)`** (`src/store/os-store.ts`) — the auto-launcher launches/focuses the singleton BS window with `params` (merged on an existing window). Unchanged.
- **`/api/gitfs/reconcile`** (`src/app/api/gitfs/reconcile/route.ts`) + `reconcile-jobs.ts` — the Supervisor's existing loopback job endpoint; `reconcile-client.mjs` (`reconcileViaApi`) is the Supervisor's existing client. No new Supervisor transport.

---

## 7. Working-context tooling (FR-003 / FR-004 / FR-005) — THE central decision

### 7.1 The working context

The escalation builds a `WorkingContext` (§4.2) **at execution time, per repo** — this is the only per-repo variable (PR-1). It is created by `escalate()` from the repo the conflict was detected in:

| `repoKind` | `worktreePath` | `repoRoot` | `mode` |
|---|---|---|---|
| `source` | the active feature-branch worktree (`featureBranchForDelegate`) | `getSourceRepoRoot()` | `working-tree` |
| `user-specs` | `path.join(dataDir(), "specs", ".worktrees", encodeBranchDir(branch))` (the exact path `promoteFeature` uses) | `userSpecRoot()` | `working-tree` |
| `user-apps` | `dataDir()/user-apps` (or the coupled worktree `dst`) | `getGitFsInstance("user-apps").root` | `working-tree` **or** `plumbing` (when the primary checkout is busy) |
| `vfs-mount` / `generic` | the mount path | the mount's repo root | `working-tree` |

There is **no per-repo special-casing of the access mechanism**: the tools below operate on `worktreePath`/`repoRoot` exactly the same way for every kind. (This directly resolves the spec's Q1: we do *not* extend `run_command`'s container sandbox, and we do *not* use the VFS-backed `file_*` tools — neither can do three-way git reads. We add a small set of **repo-path-scoped server tools**.)

### 7.2 How the context is threaded to the agent

Tools are process-global (`assistantTools()`); the per-repo context is *not* in the tool schema. Instead the escalation **tags the conversation with `conflictSessionId`** (conversation metadata, exactly analogous to the existing `activeFeatureBranch` that `featureBranchHook` in `start-run.ts` reads). Each conflict tool:
1. takes a `repo_path` argument,
2. looks up `session = getSession(conversation.conflictSessionId)`,
3. **validates `repo_path === session.workingContext.worktreePath`** — this is the access-control guard (FR-021's precondition) and the "no per-repo special-casing" guarantee: the agent can only operate on *this* session's repo, nothing else. A mismatch → the tool returns a loud error.

### 7.3 The tools (names + schemas) — `conflict-resolve.ts`

All are `serverTool(...)` (`src/lib/assistant/tools/server/util.ts`), registered in `registry.ts`, and take `{ repo_path }` (plus below). The handler enforces the §7.2 validation, then does git work under `gitLock()`.

- **`conflict_read`** — FR-005 three-way READ.
  - Input: `{ repo_path, path }`.
  - Reads the file's three-way content from the **session snapshot** (so it works on a resumed session even if the live tree was cleaned): returns `{ path, binary, conflictType, hunks: [{ base, ours, theirs }] }`. If the file is `binary`, returns `{ binary: true }` (FR-022 — the agent surfaces it, does not try to resolve).
  - The snapshot was captured from the three **refs** (merge-base / `branch` tip / `sourceRef` tip) via `git show <ref>:<rel>` (the `readFileAtRef` primitive in `store-git.ts`) — *not* from live `:1/:2/:3` stages, because the pipeline already aborted the merge/rebase before escalating (see §4.2 note). So `conflict_read` returns identical content whether the session is fresh or resumed after a restart.
- **`conflict_write`** — FR-005 hunk-level WRITE.
  - Input: `{ repo_path, path, content, hunkIndex? }`.
  - `working-tree` mode: writes the resolved content to `worktreePath/<path>` (full-file or the one hunk), and records `files[path].resolvedContent`, `resolvedBy = "agent"`.
  - `plumbing` mode: does **not** touch a working tree — records `files[path].resolvedContent` only (the plumbing completion in §5.3 builds the tree from it).
  - This is also how "edit the hunk manually" lands from the agent's side; the user's manual edit lands via the `/decision` route with `editedContent`.
- **`conflict_decision`** — FR-006 typed decision request (the park; §5.2).
  - Input: `{ repo_path, path, hunkIndex?, question, options, suggestion? }`.
  - Transitions the session to `awaiting-user`, appends the `Decision`, returns `{ parked: true, decisionId }`.
- **`conflict_status`** — read-only.
  - Input: `{ repo_path }`.
  - Returns `{ status, files: [{ path, resolved, by }], decisions: [{ id, answered, answer }] }` — lets the agent know what's left and where the user is.
- **`conflict_complete`** — terminal success.
  - Input: `{ repo_path, summary? }`.
  - Requires **every** file `resolved` (or `binary` and explicitly waived). Sets `status="resolved"`, triggers the §5.3 completion (commit + ff-main / plumbing merge). If any file is still unresolved, it returns an error listing them (the agent must finish or `conflict_abandon`).
- **`conflict_abandon`** — terminal failure/rollback.
  - Input: `{ repo_path, reason }`.
  - Sets `status="failed"` (or the caller uses `DELETE /sessions/:id` for user-initiated `abandoned`), triggers the §5.3 rollback.

**Access-control / loud-fail (FR-021):** on the *first* tool call against a session, the handler verifies the working context is actually usable: `repoRoot` has a `.git`, `worktreePath` resolves, and (for `working-tree`) the path is writable. If not, it fails loudly — sets the session `failed` with a clear message + the rollback tag — never silently pretends success. This is the "no suitable working context" edge case (US5).

**Binary (FR-022):** `conflict_read` reports `binary:true`; the agent is instructed (task text + tool description) that a binary conflict is **not** agent-resolvable — it calls `conflict_decision` (or `conflict_abandon`) surfacing "binary file requires manual handling (rollback tag provided)", never writes one.

### 7.4 Why this shape (ADR-1)

- **Repo-path-scoped server tools** (chosen): reuses the real git CLI via `runGitCommand`, does the only thing the feature needs (three-way read + hunk write + decision), is trivially access-controlled (§7.2), and works uniformly for every `repoKind`. Small, testable, no new dependencies.
- **Extending `run_command`'s sandbox** (rejected): the sandbox is a *container* keyed on `(conversation, agent)` with `/workspace` + `/tmp` only; admitting an arbitrary data-dir path would be a large, security-sensitive change to give the agent raw shell in a git repo — far more surface than "read three refs, write a file," and `run_command` is off-by-default anyway.
- **VFS-backed `file_*` tools** (rejected): they address VFS paths, not git refs/stages; they cannot produce `:1/:2/:3:` three-way content, and the working trees are not VFS mounts.
- A **hybrid** was considered but the tools above *are* the minimal sufficient hybrid (server tools + the existing git plumbing + the existing run/decision channel). No new external dependency is introduced.

---

## 8. Auto-launch via the event system (FR-007, D1)

### 8.1 The event

- **Type**: `com.bos.gitops.conflict.escalated`.
- **Emitter**: `escalate()` (in the Next.js process) via `api.emit({ type, payload: { sessionId, repo, branch, conversationId, summary }, source: { appId: "gitops", name: "GitOps" } })`.
- **Namespace ownership**: a UI handler may only be registered for its owned root (`com.bos.<ownerId>.*`) or a granted `eventNamespaces` prefix (`ownsNamespace`, `src/lib/events/types.ts`). Build Studio's owned root is `com.bos.build-studio.*`, so the manifest **must also grant** `eventNamespaces: ["com.bos.gitops.*"]`. The event type is valid per `isValidEventType` (lowercase dot-separated).

### 8.2 The Build Studio manifest entry (FR-007)

```ts
// src/apps/build-studio/manifest.ts
eventHandlers: [{
  id: "conflict-escalated",
  type: "com.bos.gitops.conflict.escalated",
  displayName: "Open conflict resolution",
  description: "Launch the Build Studio conflict-resolution pane",
  icon: "GitMerge",
}],
eventNamespaces: ["com.bos.gitops.*"],
```

`registerAppUiHandlers` (`register-ui-handlers.ts`) surfaces this at boot as a `ui` handler with `launch: { appId: "build-studio" }`. Because there is **no headless handler** for this type, `kernel.emit` marks it `processed/no-active-handlers` immediately (it is a pure UI signal) — which is exactly why §9's boot recovery does **not** rely on `redispatchPendingOnBoot` for it (see §9.2; this is the one place the spec's framing and the 034 mechanism differ, flagged in §11).

### 8.3 How it opens/focuses BS and passes the session

**Auto-launch (the extension, FR-007):** 034's UI handlers are *click-resolved* (the Event Viewer resolves a clicked event → `launch`). This feature needs *auto*-launch (the user takes no action). The mechanism: a **topbar auto-launch subscriber** (new, sibling to `EventBell`, mounted in `src/components/desktop/`) that already has the exact needed capability — `EventBell` shows that the topbar already `subscribeEventStream(...)`s the NDJSON stream. On a `kind:"new"` event of type `com.bos.gitops.conflict.escalated`, it calls:

```ts
launch("build-studio", { pane: "conflict", sessionId: payload.sessionId });
```

`launch` (`os-store.ts`) **focuses** the singleton BS window if open and **merges** `params` into it, or opens it. This is the auto-launch: no user action. (This is the *one* new client component; it does not modify 034's dispatch.)

**Session-id hand-off — component-hint + query (resolving D1's open detail):** the pane reads `params.sessionId` if present (the direct, reliable path). It **also** re-queries on load via `GET /api/gitops/sessions?status=non-terminal` (the "what session is active?" query) — this is what makes a **browser refresh** restore the pane without any event re-emit (FR-024): the pane re-derives the active session from the durable store on mount. So the component-hint is the fast path; the query is the robustness path. Both converge on the same `sessionId`.

**BS wiring (FR-007a, D5):** while a session is active, `index.tsx` renders `ConflictPane` in the **center** column (replacing the artifact viewer; the left tree stays), and points the **existing** right-side `AssistantChatV2` at the DevOps conversation via `openConversation(session.conversationId)` (loads the transcript + `attachToRun` on the live run — so the agent's question cards and the user's answers flow through the *existing* chat, not a new mechanism). When the session goes terminal, the center reverts to the artifact viewer.

---

## 9. Restart recovery (FR-024, D3)

The session is durable (file). The agent **run** is in-process and dies on restart. Two cases on boot:

### 9.1 Boot-time sweep (the re-entry point)

A new `recoverSessions()` in `conflict-session.ts`, called from the boot hook (`src/instrumentation.ts`, after the event kernel starts — the same place `registerAllUiHandlers` runs):

1. Load every `data/gitops/sessions/*.json` with a **non-terminal** status into the warm index.
2. For each:
   - **`working` whose run is dead** (always, after a restart — `runManager` is empty on boot) → **re-launch**: `startAssistantRun({ conversationId: session.conversationId, agentId: session.agentId, message: "Resuming conflict session <id> — continue resolving." })`. The transcript is intact, so the agent continues. (This is the "working + dead run → re-launch" case.)
   - **`awaiting-user`** → **restore only** (no run launch). It stays parked indefinitely (D3); the agent re-wakes only when the user answers (§5.2).
3. **Re-emit** `com.bos.gitops.conflict.escalated` for each **non-terminal** session, so the auto-launch subscriber opens BS with the pane for any session that survives a restart.

### 9.2 Honest note on `redispatchPendingOnBoot` (flagged)

The spec says restart re-emit "reuses `redispatchPendingOnBoot`." **It does not, and cannot, do the whole job**: `redispatchPendingOnBoot` (`dispatch.ts`) re-enqueues *pending* events to *active headless handlers*. `com.bos.gitops.conflict.escalated` has **no headless handler** → it is `processed/no-active-handlers` at emit time, so it is **not** "pending" and `redispatchPendingOnBoot` will never re-fire it. The actual re-emit is the explicit step in §9.1(3), and the **UI** re-opens because the pane re-queries the session store on load (FR-024's own wording: "the pane re-queries the session store on load, so a browser refresh restores it without re-emitting the event"). I am flagging this so `plan.md`/`tasks.md` don't wire the wrong hook. (If you want the 034 redispatch to *also* participate, the only way would be to give the event a no-op headless handler so it stays `pending` — not worth it; the explicit re-emit is cleaner and correct.)

### 9.3 The timeout, and why park-and-rewake makes it tractable (D3)

The 25-min budget applies to **working phases only**, never to a parked `awaiting-user` (D3). Because the agent parks (its run ends) on each decision request, "working time" is naturally the sum of active runs, not wall-clock. `lastWorkingAt` is updated whenever the session is `working`; the sweep (and a lightweight in-process timer) marks a `working` session `timed-out` only if it has been continuously `working` (a run live) past the budget with no progress. A session that is `awaiting-user` for hours is never timed out. This is clean precisely *because* the agent doesn't hold a run while parked — a blocking-wait design would have had to special-case "don't count park time," which is brittle.

---

## 10. Call-site conversions + the completeness sweep (FR-012…FR-016)

**The canonical entry** every site calls: build the `WorkingContext` for the repo, then either (in-process) call `reconcile({ …, workingContext })` or (Supervisor) `reconcileViaApi({ repoPath, sourceRef, strategy, workingContext, … })`. On `status:"escalated"` the site returns the session/conversation ids (FR-018) instead of a static error.

| # | Call site | File / function | Repo / working context | Current dead-end → new |
|---|---|---|---|---|
| 1 | **The reported bug** — feature-promote coupled-repo **pre-check** | `tools/supervisor/lib/coupled-repos.mjs` `coupledConflicts` (called from `promote.mjs` `promote()`) | **each spec store AND user-apps**, via the coupled worktree | `throw new Error(\`promote blocked — ${conflict}\`)` (the exact string in the spec's repro) → create a session + route through `/api/gitfs/reconcile` (loopback); on escalation, report `{ sessionId, devopsConversationId }` instead of throwing. |
| 2 | Feature-promote coupled-repo **merge** | `tools/supervisor/lib/coupled-repos.mjs` `promoteCoupled` | spec stores + user-apps | `catch { merge --abort; warnings.push("… merge manually in <root>") }` (working-tree) and the **plumbing** path (no live tree) → route through the pipeline; working-tree → `mode:"working-tree"`, plumbing → `mode:"plumbing"` (agent resolves from snapshot; completion via `commit-tree`+`update-ref`, §5.3). |
| 3 | **App-candidate promote** (user-apps global draft) | `tools/supervisor/lib/app-candidate.mjs` `appPromote` | user-apps, `mode` = `working-tree` if the primary checkout is free else `plumbing` | raw `git merge --no-edit APP_CANDIDATE_BRANCH` (throws on conflict) → route through `/api/gitfs/reconcile` with `repoPath = APPS_REPO`, `sourceRef = APP_CANDIDATE_BRANCH`. |
| 4 | **Build-Studio promote** (user-specs worktree) | `src/lib/specs/promote.ts` `promoteFeature` | user-specs, `mode:"working-tree"`, `worktreePath = <dataDir>/specs/.worktrees/<encodeBranchDir(branch)>` | `merge --no-edit <base>` → `catch { merge --abort; return {kind:"conflict",files} }` → route through `reconcile({ workingContext })`; return the session/conversation ids on escalation. |
| 5 | **Pull** | `src/app/api/git-remotes/route.ts` `case "fetch"` | target repo (resolved by `resolveRepoPath(filesystem)`) | inline `rebaseOntoRemote` → `rebaseConflict:true, "resolve manually, or force-push"` → on rebase conflict, route through `reconcile({ workingContext: { repoKind: fsId, worktreePath: repoPath, mode:"working-tree" } })`; respond with the session/conversation ids. |
| 6 | **Push-recovery** (non-FF) | `src/app/api/git-remotes/route.ts` `case "push"` | target repo | inline `rebaseOntoRemote` → `rebaseConflict:true` → same as #5. |

### FR-016 completeness sweep — the *complete* set of dead-end conflict paths

The spec named four; the source sweep found the real set. **All** are covered above (rows 1–6). Enumerated by searching for conflict-tolerance in the git paths:

- `promoteFeature` (`src/lib/specs/promote.ts`) — row 4. ✔
- `git-remotes` `fetch` + `push` (`src/app/api/git-remotes/route.ts`) — rows 5–6. ✔
- **`coupledConflicts` + `promoteCoupled`** (`tools/supervisor/lib/coupled-repos.mjs`) — rows 1–2. ✔ *(the reported bug's site; not named in the spec's table — "partly in the Supervisor," which this pins.)*
- **`appPromote`** (`tools/supervisor/lib/app-candidate.mjs`) — row 3. ✔
- `git-ops.ts` `mergeBranch` / `rebaseOntoRemote` / `merge-tree`-style helpers: these *throw* `MERGE_CONFLICT`/return `{status:"conflict"}` — they are **leaves**, not call sites; their callers are the six rows above (or the `reconcile` pipeline, which already escalates). No *other* caller tolerates a conflict and dead-ends. **Invariant satisfied: no git conflict path returns a static "resolve manually" with no agent.**

> **FR-015 (user-apps) pinned (D4):** the user-apps conflict paths are exactly row 3 (`appPromote`) and row 2 (the user-apps entry in `promoteCoupled`, both `working-tree` and `plumbing`). They reach the generalized mechanism by the Supervisor's existing **loopback** call to `/api/gitfs/reconcile` (`reconcile-client.mjs` `reconcileViaApi`), passing `repoPath` (from `APPS_REPO`/`getGitFsInstance("user-apps").root`) and the working context — the Supervisor never imports BOS source and never hard-codes a BOS path.

---

## 11. ADRs, risks, open questions

### ADR-1 — Working-context tooling: repo-scoped server tools (see §7.4). **Decision:** new `conflict_*` server tools over `runGitCommand`, access-controlled by the session's working context; not a `run_command` sandbox extension, not the VFS `file_*` tools.

### ADR-2 — Decision loop: park-and-rewake, not block-the-loop (see §5.2). **Decision:** the agent's run ends on `conflict_decision` and is re-launched on the user's answer, through the existing DevOps conversation. **Why:** a blocked run (the existing `awaitFrontendResult`/`elicit` shape) is in-memory and cannot survive a restart, and would pin the conversation's run slot indefinitely — incompatible with D3 (park forever) + FR-024 (restart). Park-and-rewake reuses the existing channel (D1) and makes the working-phase timeout (§9.3) tractable. **Consequence:** "continues the same run" (spec) is satisfied at the *conversation* level, not a single in-process loop; the transcript is continuous so this is invisible to the user.

### ADR-3 — Auto-launch is a small, scoped extension of 034 (see §8.3). **Decision:** a topbar stream subscriber calls `launch("build-studio", {pane, sessionId})`; the manifest declares the UI handler + `com.bos.gitops.*` grant. **Consequence/risk:** 034's UI handlers are click-resolved; this adds an *automatic* launch path. It is isolated (one new component, no dispatch change) but must not be allowed to generalize into "every UI event auto-launches." Keep it specific to the conflict type.

### ADR-4 — Session storage under `data/gitops/sessions/` (see §4.1). **Decision:** file-per-session + in-process warm index, atomic writes. Not VFS (system state, Constitution V), not a spec store (not versioned content).

### ADR-5 — Operation completion owned by the session store (see §5.3). **Decision:** the in-process session store performs commit/ff/plumbing/rollback, so it works identically whether the detector was the Next.js process or the Supervisor (separate process). The Supervisor's reconcile job just observes the terminal outcome.

### Risks / open questions (for the reviewer to push on)

1. **The reported bug's true site is the Supervisor, not `promoteFeature`.** The spec's repro string (`"promote blocked — user-specs: branch … conflicts with master"`) is emitted by `coupledConflicts` in `tools/supervisor/lib/coupled-repos.mjs`, called from `promote()`. `promoteFeature` (`src/lib/specs/promote.ts`) is a *separate* (Build-Studio-driven) path with a *different* error shape (`{kind:"conflict",files}`). Fixing only FR-012 would NOT fix the reported repro. **Both** (rows 1/4) are in scope. *(Flagged so implementation doesn't "fix" the wrong one.)*
2. **Plumbing conflicts have no live working tree.** `promoteCoupled`'s busy-checkout path merges via `merge-tree`/`commit-tree`/`update-ref` without touching a checkout. There is no tree for the agent to edit. The design handles this with `mode:"plumbing"` — the agent resolves from the snapshot into `files[].resolvedContent`, and completion builds the tree (§5.3). This is the least-tested branch; it needs explicit test coverage (the mockup's user-apps scenario exercises it).
3. **The snapshot is derived from refs, not a live conflicted tree** — because the pipeline aborts the merge/rebase before escalating (see §4.2), there are no `:1/:2/:3` stages to read. This is the *robust* choice: deterministic, restart-safe, and identical for the working-tree and plumbing cases (plumbing has no working tree at all). The one subtlety for `plan.md`: the conflicting-file set + hunk boundaries must be computed consistently (via `git merge-tree <branch> <sourceRef>` or `git diff --merge-base=<mb> <branch> <sourceRef>`) so `conflict_read` and the UI 3-way view agree on the same hunks. Pin the exact command in `tasks.md`. *(Corrected from an earlier draft that assumed a live conflicted tree.)*
4. **The `devops` agent must be given the `conflict_*` tools.** `gateFor`/`gateFromAgent` (`gate.ts`) gate tools by the agent's `tools` allowlist; a run only sees tools the agent lists. The seeded `devops` agent (and any user-selected `conflictAgent`) must have the `conflict_*` tool ids added, else the agent can't act. FR-025's "agents without git capability simply fail" is the *intended* behavior for agents lacking these tools — but the **default** `devops` agent must have them. *(Open: exactly where the devops agent's tool list is seeded — `data/agents/` vs. the subagents store — to be confirmed in `plan.md`.)*
5. **`redispatchPendingOnBoot` does not re-emit the conflict event** (it's UI-only → `processed`, not `pending`). Boot recovery re-emits explicitly (§9.2). The spec's wording should be read as "re-emit on boot," implemented by the sweep, not by the 034 redispatch. *(Flagged; see §9.2.)*
6. **`awaiting-user` is indefinite (D3)** — a parked session can outlive the run forever. The in-flight guard (`inFlightEscalations`, generalized) is keyed by `repoPath` and cleared on terminal; a long-parked `awaiting-user` session must not block a *new* conflict on the same repo from being detected — it should re-point to the existing session (US5 AS2). Confirm the guard's semantics for `awaiting-user` vs `working` in `plan.md`.
7. **Auto-launch when no browser is open.** The event + subscriber are in-process/client; if no BOS browser tab is connected, the auto-launch has nothing to launch into (the pane re-queries on next open instead). This is acceptable (the user is in the app when they trigger a promote — the spec says so), but the `working` run still runs headlessly and the session is resumable on next open.

---

## 12. Requirement realization map

| FR | Where |
|---|---|
| FR-001 | §10 canonical entry (all six rows call the same `reconcile`/`reconcileViaApi` + session create). |
| FR-002 | §4 session store + schema; durable under `data/gitops/sessions/`. |
| FR-003/004/005 | §7 working context (per-repo, execution-time) + `conflict_read`/`conflict_write`; uniform across `repoKind`. |
| FR-006 | §5.2 park-and-rewake decision loop; `conflict_decision`. |
| FR-007/007a | §8 manifest handler + topbar auto-launcher + `launch(params)`; pane reuses the existing chat via `openConversation`. |
| FR-008/009/010/011 | §8.3 pane (`ConflictPane`) renders the session (3-way from snapshot), chat decision cards, per-hunk controls (all → `/decision`), status + abandon. |
| FR-012/013/014 | §10 rows 4, 5, 6. |
| FR-015/016 | §10 rows 1–3 + completeness sweep; user-apps pinned (D4). |
| FR-017 | §5.3 (rollback tag from step 1; conflict on the feature branch; main `--ff-only`); never left conflicted. |
| FR-018 | §5.1 — `ReconcileOutcome.sessionId` + `devopsConversationId` flow to every caller (incl. the Supervisor job via `reconcile-jobs.ts`). |
| FR-019/020 | the four surfaces read `session.status` (distinguishing `awaiting-user` from terminal) via the session API. |
| FR-021 | §7.3 loud-fail on unusable working context. |
| FR-022 | §4.2 `binary` flag; §7.3 `conflict_read` reports it; never silently committed. |
| FR-023 | §5.1 — default agent `"devops"`, `activeFeatureBranch` pre-set; source path unchanged. |
| FR-024 | §9 boot sweep (re-launch `working`+dead, restore `awaiting-user`, re-emit) + pane re-query on load. |
| FR-025 | §5.1 `getConfigValue("build-studio","conflictAgent")`; §3.3 settings field + registry `load()`. |
