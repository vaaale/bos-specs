- user-specs/alpha/001-foo: found a drift

## 052-spec-tree-delete (implementation pass, 2026-09-29)

- **Deviation from design.md §3.3** — the container context menu does **not** offer
  "View history" on a folder. The history dialog's detail pane and its one-click
  **Restore** are file-shaped (`git show <ref>:<path>` + `writeFile`), so on a directory
  the restore can only ever fail. Delete… (and Rename… on a Project) are offered as
  specified; history remains a file action. FR-005 does not require it.
- **Deviation from tasks.md T004** — the new unit tests live at
  `tests/specs/spec-tree-delete.test.ts` and `tests/build-studio/container-delete-menu.test.ts`,
  not co-located `src/lib/specs/*.test.ts`. BOS's unit runner (Playwright,
  `playwright.unit.config.ts`) only collects `tests/**`; a co-located file is never run.
- **Beyond the plan's three files** — `src/apps/build-studio/Dialogs.tsx` gained an
  optional `detail` prop on `ConfirmDialog` so the count renders in the mockup's own
  callout block rather than inside the body prose, and
  `src/apps/build-studio/tree-helpers.ts` gained `containerMenuKind` / `offBranchHint`
  so the menu decision is pure and unit-testable instead of buried in `renderNode`.
- **FR-001 gap found while implementing** — the persistent error banner was rendered
  *inside* the artifact viewer, so a refusal raised while the workflow canvas (or the
  conflict / self-heal pane) owned the centre column was set and never drawn. It now sits
  above the mode switch and is dismissible. Without this, SC-002's "100% of failed-delete
  cases" was not reachable.
- **T016 not run against a running instance** — this branch's changes were verified by
  the unit suite, `tsc --noEmit` and `npm run lint` only. The worktree has no hydrating
  dev server of its own and `:3000` serves a different branch, so the acceptance
  scenarios were not exercised end-to-end in a browser.

## 041-settings-cleanup (implementation pass, 2026-10-08)

- **Deviation from tasks.md "Tests: No automated test files are requested"** — added
  `tests/config/settings-module-order.test.ts` (5 tests). US3 is a real behaviour change
  (the sidebar comparator), and AGENTS.md §1.1 makes a test for a behaviour change
  non-negotiable regardless of what the spec requests. The test pins the *properties*
  (sorted-ness, `system-tools` absence, `run-command` survival, the namespace tie-break)
  rather than a pinned title list, so it survives modules being added — which is SC-005.
  Verified red: restoring the old comparator + registration fails 3 of the 5.
  `tests/config/` is a new directory; `tests/` already carries dirs beyond AGENTS.md §3's
  enumerated list (`components`, `os`, `integrations`), so this follows the convention.
- **FR-004 is vacuous in this worktree** — `data/config/system-tools.json` does not exist
  here (`data/` is gitignored and empty on a fresh checkout). Nothing was deleted and no
  migration/cleanup code was added, so ADR-3 holds for installs where the file *does*
  exist; it simply had nothing to leave in place locally.
- **Docs touched beyond plan.md's six source files** (T015 explicitly authorises this):
  `docs/dev/architecture-overview.md` listed `system-tools` as a live config namespace
  (now false); `docs/dev/configuration/configuration-system.md` documented `order` as the
  thing that positions a tab and told the "Adding a tab" recipe to pick one (now inert —
  both corrected, with the `order` column kept as historical reference);
  `docs/usage/settings/overview.md`'s tab table was in arbitrary order and is now A→Z to
  match the sidebar, with the "Assistant" row renamed **Agents** (the registry's actual
  title). `docs/dev/run-command/run-command.md`'s "Replaces the old unsandboxed `runBash`
  tool" and `docs/dev/assistant/actions-and-tools.md`'s "Knowledge bases" index-block
  mention were both left as-is: the first is accurate history, the second describes
  `buildKbIndexBlock`, which FR-009 preserves.
- **SC-001/SC-002 not verified in a browser** — no hydrating dev server on this branch
  (see the 052 note above). The sidebar order was instead verified by calling
  `listConfigSchemas()` through the unit runner, which is the exact array `GET /api/config`
  returns: 21 modules, A→Z, no System Tools. `POST /api/system/bash` → 404 follows from
  the route file's deletion.
