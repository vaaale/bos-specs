# Contract: `gitlab_start_feature` (server tool)

The non-interactive counterpart to the frontend `dev_branch_request`. A headless run **cannot** call `dev_branch_request` — it is a frontend elicitation (`runLocalHeadless` has no browser, so `awaitFrontendResult` → `{kind:"timeout"}`), and a headless agent cannot create an *ordinary* branch any other way. This tool closes that gap for the R8 approval path.

**Source**: `src/lib/assistant/tools/server/gitlab-feature-branch.ts` (new) · registered in `src/lib/assistant/registry.ts` · bound to the conversation via `setConversationActiveFeatureBranch` (`src/lib/assistant/conversation-store.ts`).

## Why it is a server tool (not a dispatcher-side pre-bind)

- The **dispatcher** knows the issue is at a *gate* (the note hook carries `issue.labels[]`), but **not the intent** — approval-vs-feedback is a semantic read (FR-027) that belongs to the model. Binding a branch on every gate-note would allocate a feature number per feedback comment and collide on the later approval. So the **agent** owns the bind, and it needs a tool it can call **mid-run**.
- The tool is **server** (`execution: "server"`) so it survives the `headlessGate` (which keeps all server tools) and can reach `setConversationActiveFeatureBranch` + the spec-store scan (both server-side). It imports **only** pure/spec/conversation helpers — never the integration's `dispatch.ts`/`runner` (which import the tool registry) — so it is **cycle-free** (design ADR-15 / §3.10).

## The mid-run bind (the linchpin)

The branch is read **fresh from the conversation file on every tool call**: `getConversationActiveFeatureBranch` is an *uncached* `vfs.readText`, and `runLocalHeadless` resolves the active branch from `opts.conversationId` (it **ignores** the `featureBranch` option for `local` agents). Therefore a branch bound **while the agent is already running on that conversation** is picked up by the agent's **next** `file_*` write and its `dev_delegate` — no re-rooting. The tool binds to `ctx.conversationId` (the ghost conversation the dispatcher seeded for this issue), and the run's subsequent spec-store writes / `dev_delegate` resolve `bos/<NNN-slug>`.

## Signature

```
gitlab_start_feature({ project: string, slug: string }): { branch, featurePath, store, featureId }
```

| Arg | Type | Notes |
|---|---|---|
| `project` | `string` | The chosen spec-store **project id** (v1: a `user-specs` project, e.g. `integrations`). The agent picks it (the method driver / project structure); the tool resolves it. |
| `slug` | `string` | Kebab-case feature slug (e.g. `log-viewer-improvement`). The agent derives it from the feature. |
| *(implicit)* | — | The **conversation** is not an arg — the tool reads `ctx.conversationId` (the run's ghost conversation) and binds there. |

## Returns

| Field | Value |
|---|---|
| `branch` | `bos/<NNN>-<slug>` (the full branch name, `bos/`-prefixed). |
| `featureId` | `<NNN>-<slug>` (the spec-dir id, no prefix). |
| `featurePath` | `/Specs/user-specs/<project>/<NNN>-<slug>` (the spec-store feature dir). |
| `store` | `user-specs` (v1 is bos-core only — FR-029; the tool is `user-specs`-scoped and would need a marketplace variant for items, out of scope). |

## The naming rule (FR-023 — ordinary, never self-heal)

1. Resolve the project's spec dir (`/Specs/user-specs/<project>`); scan its existing feature dirs for the **max `NNN`** (the spec-kit numbering rule — `NNN` resets per project, not per store). `NNN = max(existing) + 1` (or `001` for an empty project).
2. Compose `featureId = <NNN>-<slug>`, `branch = "bos/" + featureId`.
3. **Validate** via `isValidFeatureBranch` (`src/lib/agent/feature-branch.ts`); on failure, return an error result (the agent re-derives a valid slug) — never write an invalid ref.
4. **Bind** `setConversationActiveFeatureBranch(ctx.conversationId, branch)`. The **git ref is created lazily** by the first `dev_delegate` — this tool does not create a git branch itself.

**Never** the self-heal naming (`bos/self-heal-<caseId>`). (Self-heal's naming is to be aligned to the ordinary standard in a separate, later change — out of scope, FR-023.)

## Write-time re-check-and-bump (Risk 11)

Two concurrent approvals could compute the same `NNN`. The tool **re-reads** the project's feature dirs at bind time: if `<NNN>-<slug>` (or any `NNN` ≥ the computed one) was taken in the meantime, it **bumps** `NNN` and re-binds (bounded retry, a handful of attempts). A store-level reservation is a recorded future fix, not built speculatively (YAGNI). The returned `branch` is the **authoritative** name — the agent uses what the tool returns, not a pre-guessed one (spec edge case).

## Idempotency (re-approval of the same issue)

The ghost conversation is **stable per issue** (`c-gitlab-issue-<projectId>-<iid>`) and survives across the issue's comments. If it **already has** an `activeFeatureBranch` bound (a prior approval already ran the tool for this issue), the tool **reuses** that branch + feature dir rather than allocating a new number — re-approval after a revision, or a re-dispatch, does not burn a second `NNN`. "Same issue ⇒ same feature" is keyed on the ghost conversation's branch state, not re-derived from the slug.

## Headless-gate survival

`gitlab_start_feature` is in the agent's `tools` allowlist **and** is a server tool, so it survives the `headlessGate` filter (design §3.11 / ADR-16). It is one of the few tools the **fresh** (`gitlab.issue.opened`) run does *not* need (no branch on a fresh classify) — it is used only on the **approval** path (approval-1, and the tool is not called on approval-2 which reuses the bound branch).

## Boundary

The tool binds a **branch**; it does **not** promote, merge, or write to base. Promotion is the user's action (FR-028 boundary). The branch it creates is a normal BOS feature branch, fully visible in Build Studio (FR-030).
