# Design: User repositories (050)

## 1. What is actually new

Less than it looks. Stores are already discovered by scanning `BOS_SPECS_ROOT`
for anything carrying a `spec-store.json` — **symlinks included**, which is how a
developer already points BOS at a store kept outside the data dir. This feature
adds *registration* and a *UI*, and must not introduce a second discovery path.

| | |
|---|---|
| `store-kind.ts` | gains `arbitrary`, read from the manifest rather than inferred |
| `stores.ts` | a store root may be a SUBDIRECTORY of its repo |
| registration | clone or `git init`, then symlink + manifest |
| Settings → Versions | becomes **Repositories**, reorganised around repos |

## 2. Constitution check

- **Server decides, client displays** — kind, dirty state and binding come from
  the server; the page renders them.
- **No workarounds** — two corruption paths (duplicate and nested registration)
  are REFUSED rather than warned about (§3.5).
- **Docs track features** — FR-014, and the page's own copy carries the two
  things a user cannot infer (where BOS writes; forget vs delete).

## 3. Decisions

### 3.1 Registration is a symlink, not a registry file

`specsRoot/<id>` → symlink → the store root inside the repo. The manifest inside
records the kind.

Rejected: a `repositories.json` registry. It would be a **second source of truth**
next to the existing scan — and the scan wins, because it is what
`listStores()` already reads. Two lists that can disagree about which
repositories exist is the shape of every "my repo vanished" report.

Registering is therefore: clone or init → ensure the store root exists → write
`spec-store.json` → symlink. Every step is inspectable on disk, and removing the
symlink is a complete deregistration.

### 3.2 `repoRoot` is the nearest ancestor with a `.git`

`SpecStore` already separates `root` (what spec-fs jails to) from `repoRoot`
(what git commands run in) — item stores rely on it, their root being
`user-apps/items/<id>/spec` inside the shared `user-apps` repo.

An arbitrary repo is the same shape one level out: root is `<repo>/openspec`,
repo is `<repo>`. So rather than a new manifest field, **walk up from `root` to
the nearest directory containing `.git`**. That resolves all three cases with one
rule — a store that is its own repo finds `.git` immediately.

This is the one real change to `listStores()`: today it requires the store
directory itself to be a git repo (`isGitRepo(dir)`) and sets `repoRoot: dir`, so
a subdirectory store root is skipped entirely.

### 3.3 The store root is method-declared, and BOS writes ONLY there

`openspec/` for OpenSpec, because that framework's own CLI must keep working on
the same checkout. The descriptor declares the offset; BOS does not invent one.

**This bounds the SPEC store, not BOS.** Driving a pipeline over a Python
application is pointless if BOS cannot write that application's source, so
ordinary development tools operate on the whole repo. Two different write paths,
and conflating them is how the first draft of FR-003 came out wrong: specs are
jailed so a framework's CLI can find them; code is not.

### 3.4 Everything rides a feature branch

Including arbitrary repos — especially them. `user-specs` (020) and `user-apps`
(038) already have this contract; a repo BOS did not create needs it more, since
BOS is not the only thing touching it and it may have protected branches of its
own. Work that lands unbidden on `main` is not reviewable.

### 3.5 Two refusals, because both are corruption

- **Already registered** (by URL or resolved path) — two stores over one
  worktree diverge silently, each committing over the other's branch state.
- **Nested inside a registered repo** — the same worktree with two jails.

Refused, not warned: a warning beside a completed registration is not a gate.

### 3.6 Kind lives in the manifest

Not inferred from where a directory sits. It survives a restart, travels with
the repository, and is visible to anything that reads the store — the same
reasoning as `045` FR-008's binding, and the reason `store-kind.ts` currently
has to guess `user-specs` for every directory-scanned store.

### 3.7 The page is organised by REPOSITORY, not filesystem

Today's `GitRemotesTab` groups by filesystem and nests remotes under it, so the
two things you most often need — *what kind is this* and *does it have unsaved
work* — require opening something. The redesign leads with those and demotes
remotes to a detail.

Grouped **BrowserOS / Marketplaces / Projects** because the kinds have different
rules, and a flat list hides that `bos-system-specs` can never be edited while a
project repo is wholly the user's.

See `mockup.html` (reviewed and approved). Two states in it are load-bearing
rather than decorative:

- **Add** detects an existing framework layout and offers that method instead of
  imposing the default (FR-004).
- **Remove** names *forget* and *delete* as different operations and quotes the
  unpushed work that would be lost. Defaulting to delete loses work; defaulting
  to forget leaks disk.

## 4. Risks

- **R1 — `listStores()` is on every hot path.** Walking up for `.git` adds a stat
  per store. Resolve once per scan, not per call.
- **R2 — a removed repo's symlink dangles.** Discovery already skips broken
  links; the page must show it as broken rather than hiding it, or "my repo
  vanished" has no explanation.
- **R3 — multiple writable marketplaces (P2).** `LOCAL_MARKETPLACE_ID` is
  singular throughout `marketplace/client.ts`. This is the largest part and the
  least urgent; it is sequenced last so the rest can ship.
- **R4 — detection (FR-004) can be wrong.** Offer the detected method as a
  preselected choice, never as a silent decision.

## 5. Files

```
src/lib/specs/stores.ts          root-vs-repo walk-up; kind from manifest
src/lib/specs/store-kind.ts      `arbitrary`; kind read, not inferred
src/lib/specs/repositories.ts    register / deregister / detect  (new)
src/app/api/repositories/route.ts                                (new)
src/components/apps/settings/repositories/                       (renamed + redesigned)
src/lib/config/registry.ts       "Versions" -> "Repositories"
```
