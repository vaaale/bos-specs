# Feature Specification: One branch scheme for user-apps — item specs join the feature-branch coupling, app-candidate retires

**Feature Branch**: `bos/unify-user-apps-branch-coupling`

**Created**: 2026-08-29

**Status**: Implemented.

**App Target**: bos-core

**Input**: "The agent made changes to an item's spec directly without using a feature branch. When the agent tries writing to a spec, a feature branch should have been elicited. Also, View History on that spec shows empty. I want the feature branch elicitation to function the same way as for BOS core from a user's perspective: if the agent tries writing to anything in a spec store or in user-apps, it should automatically trigger a branch elicitation."

> Supersedes `037-project-layer`'s **Edge Case** exclusion of item-owned stores and the "This requirement does not apply to item-owned stores" carve-outs in its **FR-001** and **FR-008**, and its **US4** acceptance scenario 3. Retires `007-gitfs` **FR-006** (the `app-candidate` branch) and the glossary entry for it; amends `009-installed-apps` **FR-004**'s `{draft:true}` routing; amends `035-spec-promote-conflict-escalation` **FR-015**/**D4** by removing one of its two named call sites. `020-branch-coupled-specs` is unchanged and is now the ONLY branch mechanism over `user-apps`.

## Why this exists (context)

`037-project-layer` excluded item-owned spec stores from the feature-branch rule on an explicit premise: *"An item store is a bare subdirectory of a repo shared with sibling items … plain git has no directory-scoped branch or worktree mechanism, so 'activate a feature branch for one item' would necessarily branch/expose every other item sharing that repo."*

The premise about git is correct, but the conclusion no longer follows, because `data/user-apps` was **already** a branch-coupled repo. `020-branch-coupled-specs` mounts every coupled repo — each spec store **plus `user-apps`** — as a worktree on the active `bos/*` feature branch, and promotes or discards them with the code as one logical operation. Branching "the whole repo" is exactly what that mechanism already did, and exactly what a feature branch is supposed to mean: one feature spans BOS's source, its specs, and its item content together.

The exclusion therefore bought nothing and cost correctness. Two failures followed from it:

1. **Item spec writes bypassed review entirely.** They went straight to the live default branch of `user-apps` — no branch, no draft, no promote step, no elicitation — while the same agent editing a BOS-core spec was required to have a feature branch. The user's own report: the agent revised a marketplace item's spec and the change was live immediately.

2. **A second, competing branch scheme grew over the same repo.** `app-candidate` (`007-gitfs` FR-006) checked `user-apps` out onto its own branch *in place*, so BASE served the draft. Two schemes over one repo required a `liveCheckoutOwners` registry in the Supervisor purely to stop a coupled promote from flipping branches on the directory BASE was live-serving from mid-request. Build Studio surfaced it as a per-item "Activate / inactive" badge that was purely advisory — writes succeeded whether or not the item was "activated", so the badge said `inactive` while edits went live.

Separately, treating an item store's root as a git repo root produced a silent data-visibility bug: **File history for an item's spec was always empty**, and historical versions were unreadable, because that root (`user-apps/items/<id>/spec`) has no `.git` of its own.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Writing an item's spec requires a feature branch (Priority: P1)

An agent or user editing a marketplace item's spec is subject to the same rule as a BOS-core spec: an active `bos/*` feature branch is required, and with none set the write is refused with a message that names the missing branch. The agent's response is the SAME elicitation it already uses for BOS-core work (`dev_branch_request`), so from the user's perspective there is one branch flow, not two.

**Why this priority**: This is the reported defect. Without it, an item's spec is the one writable surface in BOS with no review discipline at all.

**Independent Test**: With no active feature branch, call the item spec write/edit/patch tool; confirm it is refused, that the message names the branch requirement, and that the item's spec file and its repo's default branch are both unchanged.

**Acceptance Scenarios**:

1. **Given** no active feature branch, **When** an item's spec is written via the agent tool or Build Studio, **Then** the write is refused and the content on disk is unchanged.
2. **Given** an active feature branch, **When** the same write is made, **Then** it lands on that branch's coupled `user-apps` worktree and is committed there.
3. **Given** an active feature branch, **When** the branch is promoted, **Then** the item's spec change lands together with that branch's source code and BOS specs, as one operation.
4. **Given** an active feature branch, **When** it is discarded, **Then** the item's spec change is discarded with it.

### User Story 2 - The branch requirement never silently degrades (Priority: P1)

The requirement is unconditional. It does not depend on the Supervisor being present, on which version is being served, or on any per-item state. Only the *routing* of a write — which worktree it lands in — depends on the Supervisor; the *requirement* itself never does.

**Why this priority**: A gate that quietly disables itself when infrastructure is absent is not a gate. This mirrors the rule already true for `user-specs`.

**Independent Test**: With no Supervisor running at all, attempt an item spec write with no branch; confirm it is refused rather than falling through to the live checkout.

### User Story 3 - There is exactly one branch scheme over user-apps (Priority: P1)

`app-candidate` is retired: its Supervisor module, its `app-begin`/`app-promote`/`app-discard` control endpoints, its state field, the `liveCheckoutOwners` registry that existed only to reconcile it with the coupled scheme, Build Studio's per-item Activate/Promote/Discard row and badge, and the Topbar's "Promote app"/"Discard app" buttons all cease to exist. Item content — spec AND code — is promoted and discarded by the feature-branch controls, with the branch's source.

**Why this priority**: Two branch schemes over one repo is the defect class that produced the advisory badge and the collision registry. Removing the second one is what makes the first correct rather than merely better-enforced.

**Independent Test**: Confirm the Supervisor exposes no app-candidate endpoint and that no code path merges `user-apps` other than the coupled promote.

### User Story 4 - An item spec's full history is browsable and restorable (Priority: P2)

"View history" on any file in an item's spec lists every commit that touched it, each version's content is readable, and any version can be restored — subject to the same feature-branch rule as any other write.

**Why this priority**: The history was not merely unavailable, it was silently empty — indistinguishable from "this file has no history", which is actively misleading about whether prior versions exist to recover.

**Independent Test**: Make two commits touching an item's spec file, then list its history through the history API; confirm both appear and each version's content reads back correctly.

## Edge Cases

- **An item's spec folder is not a repo root.** It is a subdirectory of the shared `user-apps` repo. Any operation that addresses git *by repository* — listing history, or reading a blob at a ref (`git show <ref>:<path>` resolves its path against the repo root, not the working directory) — must use the owning repo and a repo-relative path. Using the store root yields silence, not an error: history lists nothing and the blob read fails as "path does not exist".
- **`user-apps` is a different repo from the spec-store mounts.** A coupled spec store mounts at `<codeWorktree>/specs/<storeId>`; `user-apps` mounts at `<previewDataDir>/user-apps`. Item stores therefore need their own path resolver. This is a path difference only — the branch policy is identical.
- **Drafting an install from BASE.** A preview process's own `user-apps` IS the branch-coupled worktree, so installs there ride the feature branch. BASE's own root is the live one, so an install belonging to a branch is redirected into that branch's data clone. `dataDir()` remains a per-process constant (base and each preview have fixed roots); the redirect is an explicit resolved root threaded through the install, never an ambient override — an ambient one would also relocate the VFS, memory and skills.
- **A branch install must not half-land.** Item files, the install symlink, seeded config and bundled assets all derive from the SAME resolved root. Splitting them — content on the branch, symlink in the live directory — would make base advertise an item it cannot serve.
- **Viewing one preview while another branch is active.** The pin and the active feature branch are independent, so this is a normal state, not an error — and resolving the data root from `BOS_VERSION_LABEL` alone silently commits the active branch's work onto the viewed branch. The root must always come from the branch itself.
- **Services on a branch install.** The manifest is validated (so a broken service still fails the install immediately) but NOT registered or started: the content belongs to a branch, and it is that branch's PREVIEW that must run it, which its own boot already does. Registering it in base would run a preview's service against base's registry and ports.
- **Sibling items share the repo.** A feature branch over `user-apps` covers every item in it, exactly as it covers every file in the BOS source repo. Per-item branches would require a worktree per item plus an item→worktree registry consulted by store discovery, the path resolver, and the install scanner, and would still collide on the repo-root `marketplace.json`; this was evaluated and rejected as disproportionate.
- **Item stores have no Projects.** The "write must target a file inside a Project" rule (`037` FR-001) continues not to apply to them; only the branch rule does.

## Requirements *(mandatory)*

- **FR-001**: EVERY writable spec store — directory-scanned and item-owned alike — MUST require an active `bos/*` feature branch before a write, rename, or delete. A read-only store MUST remain refused regardless of branch.
- **FR-002**: The branch requirement MUST be unconditional — independent of Supervisor presence, served version, and per-item state. Only the resolution of *where* a write lands may depend on the Supervisor.
- **FR-003**: A refused write MUST name the missing feature branch, and the agent MUST surface the SAME branch elicitation used for BOS-core source and spec edits. There MUST NOT be a second, item-specific activation concept.
- **FR-004**: An item-owned store's writes MUST resolve to the branch-coupled `user-apps` worktree for the active branch. The Supervisor's branch-provisioning response MUST expose the location of that mount.
- **FR-005**: Item content MUST promote and discard with the feature branch's code and specs as one operation. There MUST NOT be a separate promote/discard surface for app or item content.
- **FR-006**: The `app-candidate` branch mechanism MUST be removed in full: Supervisor module, control endpoints, published state, the live-checkout-owner registry it required, and every UI affordance for it.
- **FR-007**: A spec store MUST expose the git repository that versions its content, distinctly from its content root. All history listing and read-at-ref operations MUST use that repository together with a repository-relative path.
- **FR-008**: An item store's spec tree MUST include sub-directories and their contents, not only top-level files.
- **FR-009**: A draft install MUST require an active feature branch and MUST land its content on that branch, from BASE as well as from a preview. Item files, install symlink, seeded config and bundled assets MUST all resolve from one data root, and the branch MUST be resolved server-side from the conversation rather than accepted from the client.
- **FR-010**: A service facet installed for a feature branch MUST be validated but MUST NOT be registered or started in the installing process; the branch's preview starts it on boot.
- **FR-011**: A user MUST be able to remain on BASE for the whole of a piece of work — authoring specs and building apps — switching to the branch's preview only to test the built candidate.
- **FR-012**: Resolving a branch's data root MUST NOT be inferred from which version is running. Which preview is being VIEWED (the pin) and which feature branch is ACTIVE (the conversation) are independent, so a running preview asking for a different branch MUST get that branch's root, not its own.
- **FR-013**: An install that landed on a feature branch MUST report that fact, and the installing version MUST NOT register or launch the item: it has no install record for it and cannot serve it. The user MUST be told to build and preview that branch instead of being shown a window that cannot load.

## Success Criteria *(mandatory)*

- **SC-001**: With no active feature branch, no write to any writable spec store succeeds, with or without a Supervisor running.
- **SC-002**: An item's spec change made on a feature branch is absent from the default branch until that branch is promoted, and present after.
- **SC-003**: No Supervisor endpoint, published state field, or UI control refers to an app-only candidate.
- **SC-004**: File history for an item's spec lists every commit touching it, and each listed version's content is retrievable.
- **SC-005**: A user performing the same task on a BOS-core spec and on an item's spec encounters the same branch elicitation, with no additional per-item activation step.
- **SC-006**: Creating an app, building an app, and editing an item's spec are all possible without leaving BASE, and all land on the same feature branch.
- **SC-007**: No version ever shows a dock entry for an item installed into a different version's data root.

## Assumptions

- The user's existing "Active feature branch" selection (assistant chat dropdown, persisted per conversation) remains the single place a branch is chosen; this feature adds no new picker.
- `020-branch-coupled-specs`'s mount/promote/discard machinery is reused unchanged; this feature only removes the alternative to it and routes item stores into it.
