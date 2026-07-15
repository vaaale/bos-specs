# Feature Specification: VFS Mount Points, SpecFS, Feature Context, Provider Registry, and Marketplace

**Feature Branch**: `027-vfs-specfs-marketplace`

**Created**: 2026-07-15

**Status**: Draft

**Input**: "Relocate user-specs into the user's VFS (Documents/Specs/) so any app or agent can write to them via standard VFS file operations — not just through the spec-write tool. The git repo must survive a full VFS wipe. BOS should provide an internal service (no GitHub required). Additionally: implement a general VFS mount-point abstraction, a SpecFS backend with auto-branching and LLM-generated commits, a Feature Context for correlating branches across repos, a Provider Registry for pluggable spec and app discovery, and a Marketplace where remote git repos deliver both apps and adoptable spec templates."

> This feature supersedes **018-external-spec-store FR-005/FR-007** (global branch and symlink mount) and fulfils **020-branch-coupled-specs** (one branch name spanning BOS source and all spec stores). It also extends **009-installed-apps** into a full three-source app model (builtin / local / marketplace).

## Why this exists (context)

Three independent problems converge into one architectural opportunity:

**1. Specs are siloed behind a single tool.** The only way to write to a spec is the `spec-write` server tool. Any app that wants to deposit an artifact into a spec — the UI Preview designer writing a mockup, an agent generating a schema, a future Image Editor saving an asset — must go through that one tool. There is no general write path. Real operating systems don't have this problem: `open()`, `read()`, `write()` work the same regardless of the underlying filesystem. BOS's VFS should work the same way.

**2. User-specs live in the wrong place.** `BOS_SPECS_ROOT` points into the source clone — not the per-user data volume. In a multi-user Docker deployment this means user-spec data is entangled with BOS source. A VFS wipe or container rebuild risks losing specs. The user-specs must live in a location that is: (a) per-user, (b) protected from VFS wipes, and (c) accessible through standard VFS operations.

**3. App discovery is static and closed.** The app list is a compiled-in manifest. There is no runtime path for user-developed apps or third-party apps. The only "installed apps" mechanism is GitFS-served iframes with no discovery story. BOS needs a pluggable provider pattern that scales from "just built-in apps" to a thriving marketplace.

The solution treats the VFS the way a real OS treats its filesystem: a uniform interface (`list`, `stat`, `readText`, `writeText`, …) backed by pluggable providers. Specs become a first-class VFS path. Git versioning is a backend concern, invisible to the caller.

## Clarifications

### Session 2026-07-15

- Q: Where should user-specs live? → A: **Inside the VFS** at `Documents/Specs/`, backed by `data/specs/user/` (outside `data/vfs/`, so a VFS wipe does not destroy them). A VFS mount point routes the path to SpecFS transparently.
- Q: Must user-specs survive a full wipe? → A: **Yes.** The backing git repo lives at `data/specs/user/` which is never touched by the "clear VFS" operation (which only clears `data/vfs/`). In the Docker deployment this path is inside the per-user `data/` bind mount — still surviving container recreation.
- Q: Should the VFS mount-point abstraction be general or just for SpecFS? → A: **General.** Define an `FSBackend` interface. `LocalFS` wraps current behaviour. `SpecFS` is one implementation. Future backends (`LocalAppFS`, `SharedFS`) plug in the same way.
- Q: When should a git commit happen? → A: **Debounced flush on the write side** (2 s window). Multiple rapid writes to the same feature branch (e.g. an agent generating several spec files) coalesce into one commit. Commits are async and never block the `writeText()` caller.
- Q: How is the commit message generated? → A: **LLM call with fallback.** SpecFS computes `git diff HEAD`, sends it to the configured AI provider with a short prompt, uses the response as the commit message. On any LLM error the fallback is a deterministic message (`"Update <filename> in <spec-id>"`).
- Q: What is the branch name for a feature that spans multiple specs? → A: **The Feature Context determines the branch.** A Feature Context is a named OS-level object with `id`, `branchName` (`bos/feat/<id>`), and lists of touched specs and source paths. All SpecFS writes use the active context's branch — not the spec folder name. If two specs are edited in the same feature context they land on the same branch.
- Q: Is a Feature Context required for every write? → A: **Always.** No implicit/anonymous context. If no context is active, SpecFS returns a clear error to the caller. This keeps the branch-coupling invariant: every spec write belongs to a named feature.
- Q: How does the BOS source branch relate? → A: **Same branch name.** When the Developer agent starts modifying BOS source files, it checks the active Feature Context and works on `bos/feat/<id>` in the BOS source repo. Branch name is the correlation key across all repos.
- Q: How should promotion work? → A: **Two paths based on `touchedSourcePaths`.** If empty (spec-only change): merge spec branch → main, no rebuild required, instant. If non-empty: merge spec branch AND leave source branch open for PR + rebuild. Build Studio shows which path will be taken before the user confirms.
- Q: Should BOS_SPECS_ROOT be kept? → A: **No — convention over configuration.** All spec paths are fixed relative to `dataDir()`. The env var is removed.
- Q: What is the unified marketplace format? → A: **A git repo** with a `marketplace.json` at the root. Each item can have an `app/` subtree (pre-built, iframe-served), a `spec/` subtree (adoptable spec template), or both. Registered marketplaces are listed in `data/config/marketplaces.json`.
- Q: What is the difference between installing an app and adopting a spec? → A: **Install** = run the pre-built iframe app directly from the cloned marketplace repo (or a cached copy). **Adopt** = fork the spec subtree into `data/specs/user/`, giving the user full ownership to modify and promote. The adopted spec has no ongoing link to the marketplace source.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Any app can write to a spec (Priority: P1)

A running BOS app (e.g. UI Preview) calls `vfs.writeText('Documents/Specs/95-myapp/ui-mockup.json', content)` without knowing anything about git. The file lands in the user's spec git repo on the correct feature branch, and a commit is generated automatically.

**Why this priority**: This is the core motivating use-case; if this doesn't work the whole feature has no value.

**Independent Test**: From a server route, call `vfs.writeText('Documents/Specs/test-spec/file.json', '{}')` with an active feature context set. Confirm the file appears on `data/specs/user/` at the expected path, `git log` shows a commit on `bos/feat/<id>`, and no git operations are visible to the calling code.

**Acceptance Scenarios**:

1. **Given** an active feature context, **When** any code calls `vfs.writeText('Documents/Specs/…')`, **Then** the file is written and a debounced commit is scheduled — with no git API in the calling code.
2. **Given** no active feature context, **When** `vfs.writeText('Documents/Specs/…')` is called, **Then** a `SpecFSNoContextError` is thrown and the caller receives a clear error message.
3. **Given** five rapid writes within 2 s, **When** the debounce window closes, **Then** exactly one commit is generated containing all five changes.
4. **Given** a commit is generated, **When** the LLM call fails, **Then** the commit still happens with the fallback message (no write is lost).

### User Story 2 — A feature spans multiple specs (Priority: P1)

A developer starts a feature called "backend-with-ui" that touches spec `040-backend-service` and spec `041-ui`. Both specs land on branch `bos/feat/backend-with-ui` in the user-specs repo. When the Developer agent later implements the feature, it also works on `bos/feat/backend-with-ui` in the BOS source repo.

**Independent Test**: Set feature context `{ id: "backend-with-ui" }`. Write to `Documents/Specs/040-backend-service/spec.md` and then `Documents/Specs/041-ui/spec.md`. Confirm both commits are on branch `bos/feat/backend-with-ui` in `data/specs/user/`. Confirm `touchedSpecs` contains both paths in the persisted feature context.

**Acceptance Scenarios**:

1. **Given** an active feature context, **When** two different spec folders are written, **Then** both commits land on the same branch.
2. **Given** the same active feature context, **When** the Developer agent modifies a BOS source file, **Then** it creates/switches to `bos/feat/backend-with-ui` in the BOS source repo before making changes.
3. **Given** the feature context file, **When** the server restarts mid-feature, **Then** subsequent writes resume on the same branch (context persisted to disk).

### User Story 3 — Spec-only promotion is instant (Priority: P1)

A user edits specs, nothing else. Promoting the feature takes under 3 seconds and does not trigger a BOS rebuild.

**Independent Test**: Create feature context, write specs, promote. Measure time. Confirm `git log main` in `data/specs/user/` shows the new commits. Confirm no rebuild was requested.

**Acceptance Scenarios**:

1. **Given** `touchedSourcePaths` is empty, **When** promote is triggered, **Then** spec branch is merged to main with no rebuild.
2. **Given** `touchedSourcePaths` is non-empty, **When** promote is triggered, **Then** spec branch is merged AND the source branch name is surfaced for PR review; user is not surprised.
3. **Given** a successful promotion, **When** the feature context is cleared, **Then** subsequent writes to `Documents/Specs/` require a new feature context.

### User Story 4 — VFS wipe does not destroy specs (Priority: P1)

A user resets their VFS (clears `Documents`, `Pictures`, etc.). Their specs remain intact.

**Independent Test**: Write a spec file. Wipe `data/vfs/`. Confirm `data/specs/user/` still contains the spec file and its git history.

**Acceptance Scenarios**:

1. **Given** specs written to `Documents/Specs/`, **When** `data/vfs/` is cleared, **Then** `data/specs/user/` is unaffected.
2. **Given** a fresh VFS (post-wipe), **When** `vfs.list('Documents/Specs/')` is called, **Then** the spec files are visible again (mount point re-established).

### User Story 5 — Marketplace: browse, install, adopt (Priority: P2)

A user opens the Marketplace app, browses items from a registered marketplace repo, installs an app to run it immediately, and adopts a spec to customise it in Build Studio.

**Independent Test**: Register a local git repo as a marketplace (with a valid `marketplace.json`). Open the Marketplace app. Confirm items are listed. Install an app — confirm it appears in the app launcher. Adopt a spec — confirm it appears in `data/specs/user/` and Build Studio opens it.

**Acceptance Scenarios**:

1. **Given** a registered marketplace, **When** the Marketplace app loads, **Then** all items from `marketplace.json` are displayed with correct names, descriptions, and available actions (Run / Install / Adopt).
2. **Given** an item with an `app` entry, **When** "Run" is clicked, **Then** the app opens as an iframe in BOS.
3. **Given** an item with a `spec` entry, **When** "Adopt" is clicked, **Then** the spec is copied to `data/specs/user/`, a commit is made ("Adopt X from Y"), and Build Studio opens it.
4. **Given** an adopted spec, **When** the user modifies and promotes it, **Then** it behaves identically to a user-authored spec (no marketplace link remains).
