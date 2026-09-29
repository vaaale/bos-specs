# Feature Specification: Wire rejectContainerPath into the VFS CRUD file tools

**Feature Branch**: `bos/self-heal-0053`

**Created**: 2026-09-29

**Status**: Draft

**App Target**: `bos-core`

**Input**: Self-heal case 0053 — `file_read` (and its five CRUD siblings) accept a real on-disk container path (e.g. `/worktrees/bos/…`) and fail with a bare `ENOENT` that reads as a typo, so a capable agent retries the identical wrong call. The module already ships a guard — `rejectContainerPath` — whose docstring promises it is the "shared guard for every VFS file tool"; it is wired into `file_edit`, `file_patch`, `file_search`, `file_grep`, and `file_glob`, but not into `file_read`, `file_list`, `file_write`, `file_mkdir`, `file_delete`, or `file_rename`. This feature closes that gap so every tool in the module upholds the same contract.

## User Scenarios & Testing

### User Story 1 — An agent that points `file_read` at a real container path gets a self-repairing diagnostic (Priority: P1)

The agent is trying to read a file that lives on the real disk (a git worktree path, an `/app/data/…` path, an `/tmp/…` path) and hands that path to `file_read`. Today it gets a bare `ENOENT` that looks like a typo; it retries the identical call and fails identically, burning two turns in an unproductive loop (this is exactly what self-heal case 0053 shows). After this feature, the first call fails with a loud, named diagnostic that tells the agent the namespace is wrong and which tool CAN reach the target.

**Why this priority**: This is the case the guard was written for (EHS-0026) and the case that re-surfaced it (EHS-0053). It is the P1 because the user-visible symptom is precisely this, on precisely this tool.

**Independent Test**: Call `file_read` with `path` set to `/worktrees/bos/<branch>/…` (or any other path whose first segment is in `CONTAINER_ROOTS`). The call MUST reject with an error message equal to `containerPathDiagnostic(path)` — which includes the phrase "is a real on-disk container path, not a VFS path" — rather than an `ENOENT`-shaped `fs` error.

**Acceptance Scenarios**:

1. **Given** a conversation with no active feature branch and a `file_read` tool, **When** the agent calls `file_read({ path: "/worktrees/bos/052-spec-tree-delete/specs/user-specs/project.json" })`, **Then** the call rejects with the namespace diagnostic (the message from `containerPathDiagnostic`) and does NOT reach `vfs.readText`.
2. **Given** the same setup, **When** the agent calls `file_read({ path: "/Specs/user-specs/project.json" })` (a clean VFS path), **Then** the guard is a no-op and the tool behaves exactly as it did before (the call still reaches `vfs.readText` and the normal VFS outcome applies — read, or the existing ENOENT if the file is genuinely missing).

---

### User Story 2 — The other five CRUD tools (file_list, file_write, file_mkdir, file_delete, file_rename) uphold the same contract (Priority: P2)

The other CRUD tools have the identical gap: a real container path handed to any of them fails with a VFS-shaped error (a bare ENOENT, an empty listing, a no-op "Created folder …", etc.) rather than the namespace diagnostic. The fix that closes US1 on `file_read` must close the same gap on all five siblings in the same change, so the module's "every VFS file tool" comment is actually true.

**Why this priority**: P2 because `file_read` is the tool that triggered the case; but the contract is on the whole module, not on one tool, so the fix is wrong if it only patches `file_read`.

**Independent Test**: For each of `file_list`, `file_write`, `file_mkdir`, `file_delete`, `file_rename`, call the tool with a `path` (or `path`+`to`, for `file_rename`) whose first segment is a `CONTAINER_ROOTS` entry. The call MUST reject with `containerPathDiagnostic(path)` and must NOT reach the corresponding `vfs.*` call.

**Acceptance Scenarios**:

1. **Given** `file_list({ path: "/app/data/foo" })`, **When** invoked, **Then** it rejects with the namespace diagnostic and does NOT call `vfs.list`.
2. **Given** `file_write({ path: "/worktrees/bos/…/x.txt", content: "hi" })`, **When** invoked, **Then** it rejects with the namespace diagnostic and does NOT call `vfs.writeText`.
3. **Given** `file_mkdir({ path: "/tmp/somewhere" })`, **When** invoked, **Then** it rejects with the namespace diagnostic and does NOT call `vfs.mkdir`.
4. **Given** `file_delete({ path: "/home/user/stuff" })`, **When** invoked, **Then** it rejects with the namespace diagnostic and does NOT call `vfs.remove`.
5. **Given** `file_rename({ path: "/var/log/a.log", to: "/var/log/b.log" })`, **When** invoked, **Then** it rejects with the namespace diagnostic on the FIRST container path it sees (in the order: `path` first, then `to`) and does NOT call `vfs.rename`.

### Edge Cases

- **Clean VFS path, file genuinely missing**: The guard MUST be a no-op. A `/Specs/…` or `/Documents/…` path that does not exist in the VFS still produces the existing per-tool ENOENT/empty/`no file` outcome — this feature only changes what a REAL container path produces. (Same boundary the search tools already hold.)
- **`file_rename` with `path` clean and `to` a container path**: The guard runs against both, in the order `path` first then `to`. If `path` is clean the tool reaches the `to` check and rejects there; if `path` is the container path the tool rejects on `path` before ever looking at `to`.
- **Path with empty leading segments** (e.g. `//worktrees/…`): `isRealContainerPath` already skips empty segments (its `find((seg) => seg.length > 0)`), so `//worktrees/…` is still classified as a container path and rejected. The guard does not re-derive this; it relies on the existing predicate.
- **Case-insensitivity**: `isRealContainerPath` already lowercases the first segment (`CONTAINER_ROOTS.has(first.toLowerCase())`), so `/WORKTREES/…` is rejected the same as `/worktrees/…`. No new behavior here.

## Requirements

### Functional Requirements

- **FR-001**: `file_read` MUST call `rejectContainerPath(path)` before calling `vfs.readText`, where `path` is the tool input's `path` field coerced to a string.
- **FR-002**: `file_list` MUST call `rejectContainerPath(path)` before calling `vfs.list`, where `path` is the tool input's `path` field coerced to a string (defaulting to `"/"` when missing, exactly as the existing defaulting logic does — the guard then runs against the defaulted value).
- **FR-003**: `file_write` MUST call `rejectContainerPath(path)` before calling `vfs.writeText`.
- **FR-004**: `file_mkdir` MUST call `rejectContainerPath(path)` before calling `vfs.mkdir`.
- **FR-005**: `file_delete` MUST call `rejectContainerPath(path)` before calling `vfs.remove`.
- **FR-006**: `file_rename` MUST call `rejectContainerPath` on `path` and, if that passes, on `to`, before calling `vfs.rename`. The first container path encountered in that order MUST produce the rejection; the `to` value is rejected with its own diagnostic message when it is the container path (the diagnostic names the path it was called on).
- **FR-007**: For each of the six tools above, the guard MUST run INSIDE the same feature-scope (`inScope` / `withFeatureScope`) wrapper the tool already uses, so a branch-coupled mount path that is ALSO a clean VFS path still resolves under the right feature scope exactly as before. The guard only rejects; it never changes scoping behavior.
- **FR-008**: For each of the six tools above, a path that `isRealContainerPath` classifies as NOT a container path MUST produce exactly the same observable behavior (return value OR error) as today — this feature changes only what a container path produces. In particular:
  - `file_read` on a missing clean VFS path still yields the `fs` ENOENT via `vfs.readText`.
  - `file_list` on a missing clean VFS path still yields whatever `vfs.list` yields today.
  - `file_write`, `file_mkdir`, `file_delete`, `file_rename` on a clean VFS path still reach `vfs.*` unchanged.
- **FR-009**: A unit test MUST be added for the guard's behavior on each of the six CRUD tools, asserting (a) a container path produces an error message equal to `containerPathDiagnostic(path)` and (b) a clean VFS path does NOT produce that message. Tests MUST be written first and observed to fail against the unfixed code (the unfixed tools return a VFS-shaped outcome, not the diagnostic).
- **FR-010**: The fix MUST NOT modify `rejectContainerPath`, `isRealContainerPath`, or `containerPathDiagnostic` themselves — they are already correct (the search/edit tools that use them are the reference behavior). The fix is purely to wire the existing guard into the six CRUD handlers.
- **FR-011**: If any page under `docs/dev/` or `docs/usage/` describes the current (pre-fix) behavior of these tools — e.g. a page that lists which tools carry the container-path guard, or that documents `file_read`'s failure modes — that page MUST be updated in the same change to reflect the new behavior (Constitution VI). If no such page exists, this requirement is satisfied by adding nothing, but the developer MUST verify by searching `docs/` before concluding no update is needed.

### Key Entities

- **VFS CRUD file tool**: One of `file_read`, `file_list`, `file_write`, `file_mkdir`, `file_delete`, `file_rename` in `src/lib/assistant/tools/server/files.ts`. Each is a `serverTool` that binds the calling conversation's feature scope and dispatches to a `vfs.*` call.
- **Container path**: A path whose first non-empty segment, lowercased, is in `CONTAINER_ROOTS` (`home`, `users`, `root`, `app`, `tmp`, `var`, `worktrees`, `data-clones`, `private`). A path that names a location on the real container filesystem, not in the VFS.
- **Namespace diagnostic**: The error message produced by `containerPathDiagnostic(path)` — names the bad path, says it is a real on-disk container path, and names the tool that CAN reach the target (`run_command` for general files, `app_spec_read` for marketplace-item spec artifacts).

## Success Criteria

### Measurable Outcomes

- **SC-001**: A regression test exists that, against the pre-fix code, fails on the specific call from case 0053 (`file_read` with a `/worktrees/bos/…` path) and passes against the post-fix code.
- **SC-002**: Line and branch coverage on `src/lib/assistant/tools/server/files.ts` is ≥60% after the change, with explicit test coverage of both the reject branch and the no-op (clean-path) branch for each of the six CRUD tools.
- **SC-003**: A grep of `files.ts` shows `rejectContainerPath(` invoked inside every one of the twelve file tools in the module (the six CRUD + the five search/edit + … actually six CRUD + five search/edit = eleven; the module has eleven file tools and all eleven call the guard).
- **SC-004**: The module's docstring on `rejectContainerPath` ("Shared guard for every VFS file tool") is now true, not aspirational.

## Assumptions

- The existing `CONTAINER_ROOTS` set and `isRealContainerPath` predicate are correct and do not need to change for this case. (The diagnostic's own docstring and the search/edit tools' behavior are the reference; this feature extends their coverage to the CRUD half.)
- No user-facing UI surface is affected — these are server-side tool handlers, not rendered screens. No `ui-design` step.
- No daemon, network endpoint, or service facet is involved — `bos-core` server-file-tool change only. No `design.md` step is warranted (the feature touches one file, is obviously scoped, and reuses an existing in-module guard).
- The `inScope` wrapper is the correct place to add the guard: it already exists in every CRUD handler and is where the tool's feature-scope binding happens. The guard runs before the `vfs.*` call inside that wrapper.
- `file_rename`'s two inputs (`path`, `to`) are both checked, in order, before the `vfs.rename` call. The "first container path wins" rule means `path` is checked first and the tool rejects there if `path` is the bad one; if `path` is clean, `to` is checked next and the tool rejects there if `to` is the bad one. The diagnostic names whichever path triggered the rejection.

## Out of Scope

- Teaching the agent a different path (the agent's namespace mistake is the human-plausible behavior the diagnostic exists to catch — the fix is to make the tool fail loud, not to re-route the call).
- Modifying `CONTAINER_ROOTS`, `isRealContainerPath`, or `containerPathDiagnostic`.
- Adding new file tools, new mounts, or new VFS backends.
- Changing the `parallel` vs non-`parallel` classification of any tool.
- Modifying the Supervisor, build configuration, or any `src/lib/self-heal/**` file (the case's classification already excludes these, and the fix does not touch them).
