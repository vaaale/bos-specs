# Data Model: 035 Git Conflict Resolution System

Persisted entities + the session status state machine. Full tool schemas and event payload are in `design.md`; this file is the single reference for the **persisted** shapes.

## Entities

### `ConflictSession` (persisted: `data/gitops/sessions/<sessionId>.json`)

The first-class, durable, resumable unit of a conflict resolution. One per active conflict.

| Field | Type | Notes |
|---|---|---|
| `id` | `string` | `ses-<ts36><rand>` |
| `createdAt` / `updatedAt` | `number` (epoch ms) | |
| `status` | `SessionStatus` | see state machine below |
| `workContext` | `WorkContext` | the repo the conflict is in (parameter, not special-cased) |
| `featureBranch` | `string` | branch being reconciled (source case: also the `dev_delegate` target) |
| `baseBranch` | `string` | the branch it reconciles onto (e.g. `main`) |
| `rollbackTag` | `string` | `bos/pre-reconcile-<stamp>` — created **before** any merge/rebase (FR-017) |
| `conversationId` | `string` | the (existing, resumable) assistant conversation = the agent↔user channel (FR-007a) |
| `agentId` | `string` | the configured conflict-resolution agent that owns the run (default `devops`) |
| `snapshot` | `ConflictSnapshot` | **refs + file list only** — content re-derived via `git show <ref>:<rel>` (research R1/R6) |
| `decisions` | `ConflictDecision[]` | the decision timeline (agent questions + user answers) |
| `pendingDecision` | `ConflictDecision \| null` | the currently-open `awaiting-user` request (set → status `awaiting-user`; cleared on answer) |
| `runId` | `string \| null` | the live agent run id (null when parked in `awaiting-user` or dead) |
| `result` | `SessionResult \| null` | terminal outcome (summary / error / reason) |
| `warnings` | `string[]` | best-effort cleanup failures surfaced, not dropped |

### `WorkContext` (the FR-003/FR-004 parameter — NO per-repo special-casing of the access mechanism)

| Field | Type | Notes |
|---|---|---|
| `repoKind` | `"source" \| "user-specs" \| "user-apps" \| "vfs-mount"` | for labeling / the delegate note only |
| `repoPath` | `string` | **absolute** path to the working tree the agent operates in (source worktree, `…/specs/user-specs`, user-apps repo, VFS mount root) |
| `baseRef` | `string` | `git merge-base` commit sha (empty string → add/add base absent) |
| `oursRef` | `string` | `baseBranch` tip |
| `theirsRef` | `string` | `featureBranch` / sourceRef tip |

The **access mechanism is identical for every kind** — the agent reads/writes through `conflict_*` tools that take `repo_path` and run `git`/file ops in that path. Only this object varies per repo (FR-004).

### `ConflictSnapshot`

| Field | Type | Notes |
|---|---|---|
| `base` | `string` | `baseRef` (commit sha) |
| `ours` | `string` | `oursRef` |
| `theirs` | `string` | `theirsRef` |
| `files` | `string[]` | conflicting file paths (from `diff --name-only --diff-filter=U` or `merge-tree` output) |

Per-file 3-way content is **not stored** — `conflict_read(file)` derives it: `base_content = git show <base>:<rel>` (empty for add/add), `ours_content = git show <ours>:<rel>`, `theirs_content = git show <theirs>:<rel>`.

### `ConflictHunk` (in-memory, returned by `conflict_read`; not persisted)

| Field | Type | Notes |
|---|---|---|
| `path` | `string` | |
| `startLine` / `endLine` | `number` | hunk bounds in the unified view |
| `base`? | `string` | base side (undefined for add/add) |
| `ours` | `string` | |
| `theirs` | `string` | |
| `marker` | `"add/add" \| "modify/modify" \| "modify/delete" \| "delete/modify" \| "delete/delete"` | conflict kind |
| `agentSuggestion`? | `string` | the agent's proposed resolution (powers "accept agent's suggestion") |
| `status` | `"pending" \| "resolved"` | resolved by the user's decision or the agent's autonomous write |

### `ConflictDecision`

| Field | Type | Notes |
|---|---|---|
| `id` | `string` | |
| `askedAt` | `number` | |
| `question` | `string` | the agent's question |
| `options` | `DecisionOption[]` | e.g. `{ id:"theirs", label:"Accept theirs" }`, `{ id:"ours" }`, `{ id:"keep-both" }`, `{ id:"manual" }` |
| `path`? / `hunk`? | | the hunk the decision is about (null for a whole-file decision) |
| `answer`? | `{ optionId, manualText?, answeredAt }` | set when the user answers (clears `pendingDecision`) |
| `autonomous`? | `boolean` | true when the agent resolved it without asking (no user answer recorded) |

### `SessionResult`

| Field | Type | Notes |
|---|---|---|
| `kind` | `"resolved" \| "failed" \| "timed-out" \| "abandoned"` | |
| `summary`? / `error`? / `reason`? | `string` | |

## State machine

```
        create
           │
           ▼
        working ──agent decision──▶ awaiting-user ──user answer──▶ working
           │                            │
           │ (agent done)               │ (agent done)
           ▼                            ▼
        resolved ◀───────────────── resolved
           │
   failure/timeout/abandon:
        working ─▶ failed | timed-out | abandoned
        awaiting-user ─▶ failed | abandoned
```

| State | Meaning | Run live? | Restart behavior (FR-024) |
|---|---|---|---|
| `working` | agent is resolving (autonomous or about to ask) | yes (`runId` set) | boot sweep **re-launches** the run (research R3/R5) |
| `awaiting-user` | parked on a typed decision request; waits **indefinitely** | no (`runId` null) | **restored** as-is; agent re-launches only on the user's answer |
| `resolved` | terminal — all hunks resolved, operation completed | no | none (terminal) |
| `failed` | terminal — agent errored / no repo access (FR-021 loud-fail) | no | none |
| `timed-out` | terminal — escalation exceeded its wait (25-min default) | no | none |
| `abandoned` | terminal — user rolled back via the rollback tag | no | none |

**Invariants**
- `main`/base is **never** left conflicted; the rollback tag always exists once any merge/rebase is attempted (FR-017).
- Exactly one of `pendingDecision` (non-null ⇔ `status = awaiting-user`).
- The `inFlightEscalations` guard (keyed by `repoPath`) treats a session in `working` **or** `awaiting-user` as "active" — a concurrent reconcile on the same repo re-points to the existing session rather than starting a parallel pipeline; it does **not** re-escalate an `awaiting-user` session (design S3).

## API surface (server authority — constitution II)

`/api/gitops/sessions`
- `GET` → list non-terminal sessions; `GET ?id=` → one session
- `POST` → create (called by the pipeline escalation)
- `PATCH ?id=` → status transition / record a decision answer (the BS pane's "accept theirs/ours/…" posts here)

All read/write the session store server-side; the client only ever sees the serialized session (no `repoPath` is a *secret* here — it is the user's own data dir — but the agent's file access stays server-side and is gated to the configured agent, not exposed as a client capability).
