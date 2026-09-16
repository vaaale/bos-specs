# Implementation Plan: User repositories (050)

## Summary

Let a user register repositories from inside BOS — clone an existing one or
start a new one — and rebuild `Settings → Versions` as **Repositories** around
what a repository is and what state it is in.

The discovery mechanism already exists. This adds registration and a UI on top
of it, and the main risk is adding a second way to find a store.

## Technical context

| | |
|---|---|
| Depends on | `045` (descriptor, binding), `049` (kind, binding scope) |
| Independent of | `051` |
| Reuses | `BOS_SPECS_ROOT` scanning incl. symlinks; `SpecStore`'s existing `root` vs `repoRoot` split |
| Touches | `stores.ts`, `store-kind.ts`, the Settings registry, the Versions components |
| Mockup | `mockup.html` — reviewed and approved before the UI requirements were fixed |

## Constitution check

- **Server decides, client displays** — kind, dirty state and bindings are
  server-computed.
- **No workarounds** — duplicate and nested registration are refused, not
  warned about. Both put two jails over one worktree.
- **Docs track features** — FR-014, plus the page's own copy for the two things
  a user cannot infer.

## Test strategy

1. **Assert by diffing the repo, not by inspecting intent** (SC-003). "We only
   meant to write in `openspec/`" is precisely the claim that fails quietly.
2. **Drive the registration path the UI calls**, not the helpers under it —
   three defects in this feature family survived tests that called a helper the
   product does not call.
3. **Test the failure, not just the success**: a failed add must leave nothing,
   and that is only observable by attempting a retry afterwards.
4. **No pre-seeded repositories.** Every fixture repo is created in the test.

## Phase breakdown

| Phase | What | Gate |
|---|---|---|
| 1 | store root inside a repo; `kind` in the manifest | `user-specs` and item stores unchanged |
| 2 | register / deregister / detect | a failed add leaves nothing |
| 3 | the Repositories page | every old capability still works |
| 4 | many marketplaces (P2) | collisions reported |
| 5 | docs + a live run | spec writes land only in the method's subdir |

Phase 1 is small and everything depends on it. Phase 3 is the bulk of the
visible work. Phase 4 is separable and deferrable.

## Rollout

Additive. An absent manifest `kind` keeps today's behaviour exactly, so no
existing store changes meaning and there is no migration step.

The rename is the one user-visible disruption: the tab changes name and layout
in place. Every remote, branch and sync capability must survive it — SC-007
exists because a redesign is the easiest place to quietly drop a feature nobody
mentions until they need it.

## Risks

- **R1 — a second source of truth.** The strongest temptation here is a
  `repositories.json`. The scan already answers "which repositories exist", and
  a list that can disagree with it is the shape of every "my repo vanished"
  report. Registration writes what the scan reads; nothing else.
- **R2 — `listStores()` is on every hot path.** Walking up for `.git` adds a
  stat per store; resolve once per scan.
- **R3 — it is someone else's source tree.** The spec jail bounds the STORE, not
  BOS, and conflating the two produced a wrong first draft of FR-003. Both write
  paths ride a feature branch.
- **R4 — the redesign silently loses a capability.** The current tab carries
  remotes, auth, branches, promote and rollback. Enumerate them before moving
  code, not after.
- **R5 — multiple marketplaces is bigger than it looks.**
  `LOCAL_MARKETPLACE_ID` is threaded through `marketplace/client.ts`. Sequenced
  last on purpose; if it sprawls, ship Phases 1–3 and 5 without it.
