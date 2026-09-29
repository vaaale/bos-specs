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
