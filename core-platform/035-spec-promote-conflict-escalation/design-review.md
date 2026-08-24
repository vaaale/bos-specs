# Design Review — 035 Git Conflict Resolution System

**Reviewer**: `architect-reviewer` (independent second pass)
**Artifact reviewed**: `design.md` (v1, live-stage snapshot)
**Verdict**: **Needs one revision round.** The classification (`bos-core`) is correct, and the architecture is sound: the session store, the repo-scoped server tools, the tool gate, park-and-rewake, the auto-launch, and all six call-site conversions verify out against real source. But the **conflict-snapshot mechanism as written is broken for this pipeline** (the pipeline aborts the merge/rebase before escalating, so there are no `:1/:2/:3` stages to snapshot), it cannot support restart-recovery of the snapshot (FR-002), and it's internally inconsistent between working-tree and plumbing modes. The fix is contained (switch to a refs-based snapshot), so this is a revision round, not a rework.

---

## Must-fix findings

### M1. The snapshot mechanism (§4.2 note, §5.1 step 2, §7.3, Risk #3) cannot be captured the way the design describes — and cannot support restart-recovery.

**What I checked:** `src/lib/gitops/reconcile.ts` (the actual pipeline) against the design's snapshot prose and Risk #3.

**What the source says:** The 5-step pipeline **aborts the conflict before escalating**:
- Step 3 `attemptStrategy` — on conflict it runs `merge --abort` (merge strategy) or `resetHardAndClean` (merge-squash/commit) and returns `{ status: "conflict" }`.
- Step 4 `attemptRebaseFallback` — on conflict it runs `rebase --abort` **plus** explicitly removes `.git/rebase-merge` and `.git/rebase-apply` state dirs, then returns `{ status: "conflict" }`.
- Step 5 `escalate()` runs *after* both of those, on a **clean** working tree.

**Why the design is wrong:** The design says (§5.1 step 2) "Captures the conflict snapshot from the *live conflicted state*" and (§4.2 note / §7.3) reads it via `git show :1:<path>` / `:2:<path>` / `:3:<path>` (the merge-index stages). But by the time `escalate()` runs, the merge/rebase has already been aborted, so **those stages no longer exist** — `git show :1:`/`:2:`/`:3:` would all fail. Risk #3 acknowledges this ("getting the snapshot after abort yields empty stages") but mislabels it as "an implementation detail that must be pinned in plan.md" when it is actually an architectural contradiction: you cannot both (a) keep "the 5-step pipeline unchanged in shape" (§5.1) and (b) snapshot live stages inside `escalate()`.

Two compounding problems:
- **Working-tree vs. plumbing are inconsistent.** Risk #3 says the plumbing snapshot "comes from the `merge-tree` output" while the working-tree snapshot comes from `:1/:2/:3`. These are two different sources that must produce **identical** 3-way content for the same conflict, or `conflict_read` and the UI's 3-way view disagree between the two modes. A split-brain snapshot is a correctness risk, and it's exactly the "least-tested branch" (Risk #2).
- **It can't support restart-recovery (FR-002).** A resumed session must re-derive the snapshot after a restart, when the live tree is gone. Live `:1/:2/:3` stages are not re-derivable; only refs are. This directly undermines §9's whole premise.

**What the design needs to say instead:** Capture the snapshot **from the three refs**, not live stages:
- `base` = `git merge-base <branch> <sourceRef>`; `ours` = `<branch>` tip; `theirs` = `<sourceRef>` tip.
- Per file, read each side via `git show <ref>:<rel>`. This is **uniform** across working-tree *and* plumbing (plumbing has no worktree at all — refs are the only source), it is **restart-safe** (refs + the recorded `sourceRef` re-derive identical content), and it works for **all six call-site rows** including the dry-run pre-check (row 1, which never materializes a worktree) and the plumbing rows.

**Reviewer answers to the snapshot sub-questions (they validate the refs-based fix):**
- **(a) `git merge-tree` modern form — parseable hunks or tree+exit?** The codebase uses the `--write-tree` form: `tools/supervisor/lib/coupled-repos.mjs` `coupledConflicts` runs `git merge-tree --write-tree --merge-base=${mb} base branch` and treats the **non-zero exit as the conflict signal** — it does *not* parse per-file hunks. So `merge-tree` gives you a parseable conflicting-file set (and the merged tree sha on stdout line 1), but the existing code uses it as a boolean. The conflicting-file set can come from `merge-tree`'s output or from `git diff --name-only --diff-filter=U` (which `promoteFeature` already uses). Either is fine; the per-file *content* is more simply obtained via `git show <ref>:<rel>`.
- **(b) `add/add` — does `git merge-base` return a valid commit or empty?** It returns a **valid commit** (the common ancestor/fork point), because the two branches genuinely share history. Only the *file* is absent at that commit. So `git show <mb>:<rel>` will **throw** for the base side of an add/add — the design must catch that and treat base as empty. The schema is already shaped for this (`ConflictHunk.base?: string`, optional; the mockup shows `base: (empty)`), but the prose should state it explicitly, because the repro case *is* an add/add (`CONFLICT (add/add)` on `test-results.md`).
- **(c) Is `readFileAtRef` in `store-git.ts` a reusable `git show <ref>:<rel>`?** Confirmed: `src/lib/specs/store-git.ts:127` `readFileAtRef(root, ref, rel)` is literally `git(root, ["show", `${ref}:${norm}`])` — a plain `git show <ref>:<rel>` parameterized by `root`, so it generalizes across repos. Two caveats: it **throws when the file is absent at the ref** (the add/add base side), and it lives in the *spec-store* git module — pulling it into general gitops is a slightly odd dependency. `git show <ref>:<rel>` would more naturally go through `runGitCommand` (`src/lib/gitops/git-ops.ts`) than by importing a user-specs helper. Minor, but worth pinning.

### M2. FR-019/FR-020 (US4 — P2) surfaces are absent from the Component/file plan.

**What I checked:** `spec.md` FR-019/FR-020 + User Story 4 against `design.md` §3.3 (the Concrete File/Module Plan) and §12.

**What the source/spec says:** FR-019 requires `VersionControls.tsx`, `VersionsTab.tsx`, `ConflictResolutionDialog.tsx`, and `GitRemotesTab.tsx` to "show the session state (agent indicator, link into the resolution UI, conflicting file list, rollback tag) for a conflict escalated from any repo." All four files exist (e.g. `src/components/apps/settings/versions/ConflictResolutionDialog.tsx`, `src/components/apps/settings/versions/GitRemotesTab.tsx` — which today renders the static `rebaseConflict` message at `GitRemotesTab.tsx:693` — and `src/components/desktop/VersionControls.tsx`).

**Why it's a finding:** The design's §12 realization map *acknowledges* FR-019/020 ("the four surfaces read `session.status` … via the session API"), so it isn't silent. But the **§3.3 file plan does not list a single one of these four surfaces as a file to create or modify.** A design whose file plan omits the P2 surfaces it is supposed to wire up has an incomplete plan — the actual per-surface changes are unspecified.

**What it needs to say:** Add the four surfaces to the plan with the specific change in each, or explicitly scope them to `plan.md` with the per-surface contract. Given US4 is P2, this isn't blocking the core engine, but the file plan as written is incomplete against the spec.

---

## Should-improve findings

### S1. §7.2 — the tool handler does not receive a "conversation object." (Clarification)
**What I checked:** `src/lib/assistant/tools.ts` `ToolContext`, and how `activeFeatureBranch` is actually threaded.

**What the source says:** `ToolContext` exposes `conversationId` (a **string**), `agentId`, `runId`, `signal`, `onEvent`, `elicit`, `delegationDepth` — **no conversation object, no `conflictSessionId`.** The existing `activeFeatureBranch` mechanism works by (1) persisting it as a **top-level field on the conversation JSON file** at `/Documents/Chats/<id>.json`, and (2) reading it via `getConversationActiveFeatureBranch(conversationId)` (`src/lib/agent/conversations-server.ts:28`), which does a VFS read of that file. Critically, `ConversationFile` in `src/lib/assistant/conversation-store.ts` has `[key: string]: unknown`, and `saveConversationMessages` preserves all existing top-level fields (it only overwrites `.messages`) — so a `conflictSessionId` field **would survive the park→rewake boundary exactly like `activeFeatureBranch`**.

**Why it matters:** The design's §7.2 "looks up `session = getSession(conversation.conflictSessionId)`" and §5.1(4) "tags the conversation with `conflictSessionId` … so the conflict tools can find the session by `ctx.conversationId`" conflate two things. The mechanism is **viable** (the design is right that it's "analogous to `activeFeatureBranch`"), but the design should name the actual accessors it will add — a `getConversationConflictSessionId(conversationId)` getter (mirroring `getConversationActiveFeatureBranch`) that the tool calls with `ctx.conversationId`, and a write (mirroring `setConversationActiveFeatureBranch`, or just the `createDevOpsConversation` file-write that already sets `activeFeatureBranch`) in `escalate()`. As written, a reader can't tell *how* the tool reads the tag.

### S2. §9.1(3) — boot re-emit can double-fire the auto-launch for a session created just before a restart.
**What I checked:** `src/lib/events/dispatch.ts` / `types.ts` event semantics + `os-store.ts` `launch`.

**What the source says:** `api.emit` creates a **new** `EventRecord` each call (per-type sequence increments) — a boot re-emit is a *new* event, not a de-duplicated duplicate. If a session was created just before the restart and its original emit was already delivered, the boot sweep re-emits and the auto-launch subscriber fires `launch("build-studio", {pane, sessionId})` **twice**. This is **benign** — `launch` on the singleton (`os-store.ts`) focuses + merges params, so a second call is idempotent — but the design doesn't acknowledge it.

**What it should say:** Note that the boot re-emit may double-fire the auto-launch, that this is harmless because `launch` on the singleton is idempotent, and (optionally) dedup by `sessionId` in the subscriber. Low severity.

### S3. §10 — "generalize `inFlightEscalations` to all repo paths" is imprecise.
**What I checked:** `src/lib/gitops/reconcile.ts` `inFlightEscalations`.

**What the source says:** `inFlightEscalations` is **already** `Map<repoPath, devopsConversationId>` — keyed by `repoPath`, not source-repo-only. So it's already "per repo path." What actually changes is (a) the value grows to carry a `sessionId`, and (b) the guard must now reason about a **parked `awaiting-user`** session (non-terminal) vs. an active `working` one. Risk #6 correctly flags the `awaiting-user`-vs-`working` guard semantics as an open question — that's the real issue. The "generalize to all repo paths" framing should be corrected to "extend the guard's value + semantics to `awaiting-user`," so a plan reader doesn't go looking for a keying change that doesn't need to happen.

### S4. Risk #4 — where the `devops` agent is seeded is left open, but FR-023's regression guarantee depends on it.
The design correctly identifies (Risk #4) that the default `devops` agent **must** have the `conflict_*` tool ids in its allowlist, because `gate.ts` (`gateFor`/`gateFromAgent`) gates on `agent.tools` (`visibleTools` in `tools.ts` skips any registry tool not in `gate.allow`). This is necessary and correct. But it defers to `plan.md` "exactly where the devops agent's tool list is seeded." Since FR-023 (source-repo regression, default `"devops"`) is a hard "must behave identically" requirement, the seeded location + the tool-allowlist edit should be resolved during this design revision, not pushed to plan — otherwise the default path is a silent `failed` session until someone remembers to add the tools.

---

## What the design got right (safe to leave alone in a revision)

These all verified out against real source and should **not** be re-litigated in a revision round:

- **Classification = `bos-core`.** Correct and consistent with `spec.md`'s `App Target`. The "user-apps looks like a marketplace-item but isn't" note (§1) is the right call.
- **`serverTool(...)` + registration pattern (§7.3).** `src/lib/assistant/tools/server/util.ts` `serverTool(name, description, parameters, execute)` returns an `AssistantTool` with `execution: "server"`; server tool modules are spread into the `combined` map in `assistantTools()` (`src/lib/assistant/registry.ts`). A new `conflict-resolve.ts` exported and spread in is exactly how tools are registered. **Correct.**
- **The tool gate (§7.3, Risk #4).** `gate.ts` `gateFromAgent` builds `allow` from `agent.tools`; `visibleTools` enforces it. A run only sees tools the agent lists. **Correct.**
- **Park-and-rewake (§5.2).** All three sub-points verify: `startAssistantRun({conversationId, agentId, message})` **continues** an existing conversation — `runAgentLoop` does `io.loadMessages()` (full transcript) then appends the new user message (`agent-loop.ts`); `runManager.create` throws `ActiveRunError` only while a run is `"running"`, and `finish` frees the slot immediately, so park (run ends) → rewake (new run) has no race; the re-wake's first user message *is* the answer **and** the agent re-reads the transcript, so "continues" is satisfied. **Correct — this is the design's strongest section.**
- **Auto-launch (§8).** `launch` (`os-store.ts`) **focuses the singleton and merges `params`** (`build-studio` is `singleton: true`) or opens it — correct. `ownsNamespace` (`src/lib/events/types.ts:189`) returns true **only with** the `eventNamespaces: ["com.bos.gitops.*"]` grant, because build-studio owns `com.bos.build-studio.*` not `com.bos.gitops.*` — the grant is genuinely required. `eventHandlers`/`eventNamespaces` are real `AppManifest` fields (`os/types.ts:39,44`), threaded to `api.register` by `register-ui-handlers.ts:30`. `EventBell` (`src/components/desktop/EventBell.tsx`) really does `subscribeEventStream(...)`, so the "new sibling" mount point is right; `subscribeEventStream` is per-caller, so a separate subscriber — not extending EventBell's connection — is the correct choice. **All correct.**
- **Restart-recovery boot hook (§9.1).** `src/instrumentation.ts` `register()` is the documented "Next.js server-boot hook, runs once per server process," and it's exactly where `startEventKernel()` + `registerAllUiHandlers()` run. `recoverSessions()` there is the right place. **Correct.** And §9.2's honest note that `redispatchPendingOnBoot` **cannot** re-emit this event is correct: `dispatch.ts` `redispatchPendingOnBoot` re-enqueues *pending* events to *active headless* handlers only; a UI-only event is `processed/no-active-handlers` at emit, so it's never "pending." The explicit re-emit is the right call.
- **All six call-site conversions (§10).** Verified each against source:
  - Row 1: `coupledConflicts` **does** use `git merge-tree --write-tree --merge-base=${mb} base branch` (modern dry-run) — and it's the **reported bug's true site** (`promote.mjs` `promote()` → `const conflict = await coupledConflicts(...); if (conflict) throw new Error(\`promote blocked — ${conflict}\`)`). **Risk #1 is correct and important.**
  - Row 2: `promoteCoupled` **has both** a working-tree path and a plumbing path. **Correct.**
  - Row 3: `appPromote` **does** a raw `git merge --no-edit APP_CANDIDATE_BRANCH` with no conflict handling (throws). **Correct.**
  - Row 4: `promoteFeature` is `git merge --no-edit <base>` → catch → `git diff --name-only --diff-filter=U` → `merge --abort` → `return { kind: "conflict", files }`. **Correct.**
  - Rows 5–6: `git-remotes/route.ts` `case "fetch"` and the `case "push"` non-FF recovery **both** call `rebaseOntoRemote` and dead-end on `rebaseConflict: true`. **Correct.**
  - The Supervisor loopback path is real: `promote.mjs` imports `reconcileViaApi` from `reconcile-client.mjs` → `/api/gitfs/reconcile` → `reconcile-jobs.ts` `startReconcileJob`. The Supervisor never imports BOS `@/` source. **Correct.**
- **FR-025 config (§5.1, §3.3).** `src/lib/config/registry.ts` has the `build-studio` namespace with `load()` returning `{ agent: (s.agent) || "build-studio" }` — adding `conflictAgent` (default `"devops"`) to that `load()` is a one-line change, and `getConfigValue(namespace, key)` **exists**. The two fields have **different** defaults (`agent`→`"build-studio"`, `conflictAgent`→`"devops"`) and the design handles that correctly. **Correct.**
- **Session storage (§4.1).** `src/os/atomic-write.ts` `writeFileAtomic` exists and is the right primitive; `data/gitops/sessions/` under `data/` (not VFS, not a spec store) is the correct class of state. **Correct.**

---

## Verification matrix

| # | Design claim | Source checked | Verdict |
|---|---|---|---|
| 1a | `serverTool(...)` registration pattern | `tools/server/util.ts`, `registry.ts` | ✅ Correct |
| 1b | Agent must have `conflict_*` in allowlist | `gate.ts`, `tools.ts` `visibleTools` | ✅ Correct |
| 1c | Tool reads `conversation.conflictSessionId` | `tools.ts` `ToolContext` (string only), `conversation-store.ts` | ⚠️ Viable but underspecified — tool gets `conversationId` (string), must add a getter; see S1 |
| 2a | `startAssistantRun` continues an existing conversation | `start-run.ts`, `agent-loop.ts` | ✅ Correct |
| 2b | No run-slot race on park→rewake | `run-manager.ts` `create`/`finish`/`ActiveRunError` | ✅ Correct |
| 2c | Re-wake's first message is the answer | `agent-loop.ts` | ✅ Correct |
| 3 | Snapshot from `:1/:2/:3` at escalate time | `reconcile.ts` (aborts before escalate) | ❌ **Must-fix** — stages gone by escalate; see M1 |
| 4a | `launch` focuses + merges `params` on singleton | `os-store.ts` `launch` | ✅ Correct |
| 4b | `eventNamespaces` grant required | `types.ts` `ownsNamespace`, `register-ui-handlers.ts` | ✅ Correct |
| 4c | New sibling to `EventBell` in `desktop/` | `EventBell.tsx`, `subscribeEventStream.ts` | ✅ Correct |
| 5a | `instrumentation.ts` is the boot hook | `instrumentation.ts` `register()` | ✅ Correct |
| 5b | Boot re-emit double-emit risk | `dispatch.ts`, `os-store.ts` `launch` (idempotent) | ⚠️ Benign but unacknowledged — see S2 |
| 6a | `coupledConflicts` uses `merge-tree --write-tree`; `promoteCoupled` both paths | `coupled-repos.mjs` | ✅ Correct |
| 6b | `appPromote` raw `merge --no-edit` | `app-candidate.mjs` | ✅ Correct |
| 6c | `promoteFeature` `merge --no-edit <base>` + `{kind:"conflict"}` | `promote.ts` | ✅ Correct |
| 6d | `fetch`/`push` use `rebaseOntoRemote`, dead-end `rebaseConflict` | `git-remotes/route.ts` | ✅ Correct |
| 7a | `build-studio` namespace `load()` can return `conflictAgent` | `config/registry.ts` | ✅ Correct (one-line add) |
| 7b | `getConfigValue(namespace, key)` exists | `config/registry.ts` | ✅ Correct |

**Net:** 1 must-fix architectural (M1, the snapshot), 1 must-fix coverage (M2, the FR-019 file plan), and 4 should-improve clarifications (S1–S4). The design's core is genuinely solid — the revision round is narrowly scoped to re-deriving the snapshot from refs, closing the P2 surface plan, and tightening the access/guard details.

*One thing the reviewer could not verify from source alone:* the exact seeding location of the `devops` agent (S4) — left as an open item rather than asserting a location.
