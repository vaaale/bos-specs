# Quickstart / Validation Scenarios: 035 Git Conflict Resolution System

These are the end-to-end scenarios the `e2e/035-*.spec.ts` tests must cover (self-tests before promotion, constitution IV/VII). Each names the call site, the conflict kind, and the observable success.

## S1 — Repro case: user-specs add/add on Supervisor feature-promote (FR-012a, the reported bug)

1. Two feature branches both add `core-platform/<x>/test-results.md` with different content.
2. Run the Supervisor feature-promote on one → the coupled-repo **pre-check** (`coupledConflicts`) detects the conflict.
3. **Assert**: `promote()` does **not** throw; instead it creates a `ConflictSession` (status `working`), emits `com.bos.gitops.conflict.escalated`, and the promote response carries `sessionId` (FR-018). `main` is **untouched** (FR-017).
4. **Assert**: Build Studio auto-launches (was closed) showing the conflict pane, amber `awaiting-user` (or `working`), repo label "user-specs store", branch, rollback tag.
5. **Assert**: the existing BS chat is the agent conversation; the agent's decision question appears there AND in the pane.
6. User answers "accept theirs" → agent writes the hunk, operation completes, session `resolved`, `main` receives the merged result.

## S2 — Source-repo promote escalation does not regress (FR-023)

1. A BOS-source feature branch conflicts with `main` on promote.
2. **Assert**: the pre-existing escalation path behaves identically (conversation created, `activeFeatureBranch` set for `dev_delegate`, rollback tag, 25-min timeout) **and** now also creates the session + emits the event + shows the pane. The default agent is `devops` (FR-025 default).

## S3 — Pull (git-remotes `fetch`) divergence (FR-013)

1. Local branch diverged from remote; `rebaseOntoRemote` conflicts.
2. **Assert**: the `case "fetch"` path routes through the pipeline (not the static `rebaseConflict: true` dead-end), creates a session with the **correct repo's** working context, returns `sessionId`.
3. `GitRemotesTab` shows the live session state + "Open resolution" (FR-019), not "resolve manually, or force-push."

## S4 — Push-recovery non-FF (FR-014)

1. A non-fast-forward push triggers the recovery rebase, which conflicts.
2. **Assert**: same as S3 for `case "push"` — session + escalation, correct working context.

## S5 — App-candidate promote (user-apps) (FR-015)

1. `appPromote`'s raw `git merge --no-edit` conflicts.
2. **Assert**: routes through the pipeline with the **user-apps** working context; the Supervisor reaches it via loopback HTTP (no BOS `@/` import).

## S6 — VFS-mounted repo (cross-repo generality, FR-003/004)

1. A VFS-mounted repo reconciles and conflicts.
2. **Assert**: the **same** session/agent/UI machinery works with the VFS mount's `WorkContext` — no per-repo special-casing of the access mechanism. `conflict_read`/`conflict_write` operate in the mount path.

## S7 — Autonomous-first (D2): agent resolves the unambiguous, asks only on ambiguity

1. A conflict with one clearly-mergeable hunk and one genuinely ambiguous hunk.
2. **Assert**: the agent resolves the unambiguous hunk **autonomously** (recorded in `decisions` with `autonomous: true`, no user prompt) and issues a typed `conflict_decision` **only** for the ambiguous one.

## S8 — Binary file loud-fail (FR-022)

1. A conflict involving a binary file.
2. **Assert**: the agent **does not** silently commit/merge the binary; it surfaces a clear error (session `failed` with a message naming the file and the missing capability), or explicitly asks the user.

## S9 — Restart recovery (FR-024, D3)

1. **Case A** (`working`): session created, agent running, then BOS restarts.
   - **Assert**: the boot sweep (`recoverSessions`) detects the dead run, re-emits the event (BS re-launches pane), and **re-launches the agent run** on the same conversation. The snapshot is re-derived from refs (research R1) — identical to pre-restart.
2. **Case B** (`awaiting-user`): session parked, then BOS restarts.
   - **Assert**: restored **as-is** (pane shows the pending decision); the agent does **not** re-launch until the user answers. Answering then re-launches the run.
3. **Browser refresh** (no restart): the pane re-queries the session store on load and restores state (no event re-emit needed).

## S10 — Rollback / abandon (FR — rollback affordance)

1. An active session; user clicks "Abandon & roll back."
2. **Assert**: the working tree is restored to the `rollbackTag`, the operation is not committed, and the session transitions to `abandoned` (terminal). `main` remains unconflicted.

## S11 — Configurable agent takes effect (FR-025)

1. Set `build-studio.conflictAgent` to a non-default agent (via Settings → Build Studio, second dropdown).
2. Trigger a conflict.
3. **Assert**: the escalation conversation is scoped to the **selected** agent (read at escalation time, no reload needed). If the selected agent lacks git/`conflict_*` access, the session is `failed` with a clear message naming the agent and the missing capability.

## S12 — Concurrent-op guard (FR-022 / design S3)

1. A session is `awaiting-user` on repo R.
2. A concurrent reconcile on repo R is attempted.
3. **Assert**: it re-points to the existing session (does **not** start a parallel pipeline, does **not** re-escalate the parked session).

---

**Unit-level (not e2e)**: `reconcile()` generalization (each step's outcome), `readFileAtRef` (incl. add/add base-side catch), the session store transitions + `writeFileAtomic`, the boot sweep's dead-run detection, and the `inFlightEscalations` `awaiting-user` guard.

**Quality gates before promotion**: `tsc --noEmit`, `lint`, unit tests, and the e2e suite S1–S12 all green (constitution VII).
