# Implementation Plan: file_grep — Single-File Content Search Tool

**Branch**: `bos/self-heal-0002` | **Date**: 2026-09-08 | **Spec**: `user-specs/assistant/043-file-grep/spec.md`

**Input**: Feature specification from `/Specs/user-specs/assistant/043-file-grep/spec.md` (self-heal case 0002, scope class **e**, ownership **bos-core**). Diagnostic report: `/Documents/BOS Improvements/self-heal-0002.md`.

## Summary

Add a new `file_grep` server tool to the FILES group that searches **one named VFS file** for a literal substring and returns the matching lines with 1-based line numbers, in the FILES group's existing `path:line:text` form. This closes a genuine gap: `file_search` is a directory-subtree search (`src/lib/assistant/tools/server/files.ts:156-190`) whose `path` argument is a directory — passing a file yields a silent `[]`, and no existing tool can grep a single named file. `file_grep` is read-only and parallel-safe (FR-004), fails loudly on a non-file path (FR-008, the core defect), is not extension-gated (FR-009), is bounded in output (FR-011), and is registered as a discoverable capability with corrected alias routing so single-file-grep intent lands on `file_grep` rather than mis-routing to `file_search` (FR-012/FR-013). The change is strictly **additive**: it adds one tool + one capability row + the FR-013 alias correction, and does not alter `file_search`, `file_glob`, or any other tool (FR-014).

## Technical Context

**Language/Version**: TypeScript (BOS monorepo, strict mode, Next.js 15 App Router). Server-only (`import "server-only"`).

**Primary Dependencies**: none new. Reuses the existing FILES tool utilities (`serverTool`, `parallel`, `schema`, `p` from `src/lib/assistant/tools/server/util.ts`) and the VFS read surface (`@/os/vfs` — `stat`, `readText`). No new packages; `package.json`/lockfiles are untouched (Constitution VII).

**Storage**: N/A (read-only; single `vfs.readText` of the named file per call). No DB, no new state.

**Testing**: Playwright unit suite (`npm run test:unit` — `NODE_OPTIONS='--conditions=react-server --require ./tests/_no-external-network.cjs' playwright test -c playwright.unit.config.ts`). Server-tool tests live under `tests/assistant/` (e.g. `fetch-to-path.test.ts`, `discovery-modes.test.ts`) and use `useTestDataDir(...)` + `import "../services/_stub-server-only"` to obtain a hermetic VFS; a tool handler is invoked directly via `fileTools().<tool>.execute!(input, ctx)`. Coverage gate is ≥95% line + branch on every modified file (NFR-004), measured by the repo's standard coverage run.

**Target Platform**: The BOS server process (Node 20+, the same host that runs `npm run dev` / the Supervisor).

**Project Type**: library — a server tool module plus its capability-registry entry, within the existing assistant tool subsystem.

**Performance Goals**: A single-file grep over up to ~5 MB (the existing FILES tool size budget) completes in well under one second of CPU (NFR-001). Implementation reads the file once and does **not** walk a directory tree (contrast `file_search`, which calls `walkFiles(root, 5000)`).

**Constraints**:
- Response size bounded by hard caps independent of match density (NFR-002 / FR-011): a max match count and a max per-line length; when either is hit the response states it was truncated and how many matches were withheld.
- Literal substring match only — no regex interpretation, so no ReDoS/escape surface (FR-003, resolved in spec Clarifications).
- Line numbers always on, 1-based, `path:line:text` form consistent with `file_search` (FR-005 / NFR-003).
- Non-file paths (directory, missing, unreadable) MUST return an explicit in-band error, never a silent `[]` (FR-008).
- No extension whitelist gating (FR-009); undecodable (binary) content reports explicitly.

**Scale/Scope**: Two source files touched (`files.ts`, `capabilities-registry.ts`), one new test file (`tests/assistant/file-grep.test.ts`), docs update (`docs/dev/assistant/actions-and-tools.md`). No new subsystem, no new group, no migration.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **I. Spec-Driven (NON-NEGOTIABLE)** — ✅ An agreed `spec.md` exists at `user-specs/assistant/043-file-grep/spec.md` (specify + clarify complete; all design decisions resolved in its Clarifications section). This plan references it; implementation follows FR-001…FR-014 / NFR-001…NFR-005.
- **II. Server Authority & SSR Boundary** — ✅ `file_grep` is a server tool in `src/lib/assistant/tools/server/files.ts` (`import "server-only"`), operating on the VFS. No secrets, no client-side Node/`fs`.
- **III. Always Delegate; Claude Codes** — ✅ Implementation is delegated to the Developer (Claude) sub-agent via `dev_delegate`; Build Studio authors only spec/plan/tasks.
- **IV. Minimize Blast Radius (NON-NEGOTIABLE)** — ✅ All changes land on the feature branch `bos/self-heal-0002` in an isolated worktree; the change is additive (a new tool + capability row), no existing behavior modified.
- **V. The VFS Is Not the Source** — ✅ BOS code is changed by editing `src/` via the Developer; the VFS is only the runtime data surface the tool reads. Spec artifacts live in the `user-specs` store, not `src/`.
- **VI. Specs & Docs Stay in Sync (NON-NEGOTIABLE)** — ✅ `docs/dev/assistant/actions-and-tools.md` is updated in the same change (NFR-005) to document `file_grep` and its distinction from `file_search`.
- **VII. Respect Boundaries** — ✅ No changes to secrets, `package.json`, lockfiles, or build config. Quality gates `npx tsc --noEmit` and `npm run lint` must pass on changed code; `npm run build` is not run while `next dev` is active.

**Result**: PASS — no violations, no Complexity Tracking required. (No `design.md` was produced: this is an additive, single-subsystem feature with no daemon/network surface and no UI; the architecture is fully determined by the spec's Context section and the resolved Clarifications, so the `architect` step is skippable per the pipeline's "small, obviously-scoped feature" rule.)

## Project Structure

### Documentation (this feature)

```text
Spec store (external git-backed store, user-specs):
Specs/user-specs/assistant/043-file-grep/
├── spec.md              # specify + clarify output (complete — source of truth)
└── plan.md              # This file (plan output)
```

### Source Code (BOS repository root — created/modified by `implement`)

```text
src/lib/assistant/
└── tools/server/
    └── files.ts                 # MODIFY — add `file_grep` to `fileTools()` (read-only, parallel(serverTool(...)), withFeatureScope)
src/lib/agent/
└── capabilities-registry.ts     # MODIFY — add `file_grep` FILES capability row (next to file_search:117 / file_glob:118); FR-013 alias correction
tests/assistant/
└── file-grep.test.ts            # NEW — Playwright unit tests (TDD; fail on unfixed base); NFR-004 ≥95% line+branch
docs/dev/assistant/
└── actions-and-tools.md         # MODIFY — document `file_grep` and its distinction from `file_search` (NFR-005)
```

**Structure Decision**: This is the existing single-project (BOS monorepo) layout. `file_grep` is added to the **existing** `files` tool group — no new group is created (the `files` group already exists at `src/lib/agent/tool-groups.ts:53`, and Constitution VII / the "Adding a tool (recipe)" doc both say to prefer extending an existing group over inventing one). The tool is a sibling of `file_search`/`file_glob` in `files.ts`'s `fileTools()`, mirroring their `parallel(serverTool(...))` + `withFeatureScope` + `JSON.stringify` conventions.

## Design Notes

### The tool (FR-001…FR-011)

Add to `fileTools()` in `src/lib/assistant/tools/server/files.ts`, as a sibling of `file_search` (`:156`) and `file_glob` (`:195`):

```
file_grep: parallel(serverTool(
  "file_grep",
  "<description that says SINGLE named file, distinct from file_search's directory search; path:line:text; literal substring; line numbers on; optional ignoreCase + context>",
  schema({
    path:        p.str("The single VFS file to search (a file path, not a directory)"),
    pattern:     p.str("Literal substring to search for (matched exactly, no regex)"),
    ignoreCase:  p.bool("Match letter case-insensitively (default: case-sensitive)"),
    context:     p.num("Show up to n preceding and n following lines around each match (default: 0)"),
  }, ["path", "pattern"]),
  async (input, ctx) => withFeatureScope({ conversationId: ctx.conversationId }, async () => { ... }))
))
```

The handler, in order:
1. **Validate the target is a file (FR-008/FR-010).** Call `await vfs.stat(path)` first. If it throws (no such path) → throw `Error("file_grep: no file at '<path>'")`. If `stat` returns `type === "dir"` → throw an error naming that it is a directory and pointing to `file_search` for directory-scoped search. If it is unreadable/permission-denied → throw a permission error. Because `serverTool` catches executor errors and returns `Error: file_grep: <message>` in-band (`util.ts:16-34`), these are explicit, never a silent `[]`. (The exact error-text wording is a Developer implementation detail; the requirement is the **behavior**: non-file → explicit in-band error, file → matches or clean zero-match.)
2. **Read the file once (NFR-001).** `const content = await vfs.readText(path)`. No `walkFiles` call — a single read of the one named file. If `readText` throws (undecodable/binary) → throw an explicit error (FR-009), no silent empty.
3. **Match (FR-003, FR-005, FR-006).** Split on `\n`; scan line-by-line with a literal substring test. If `ignoreCase` is set, case-fold both the pattern and each line (mirroring `file_search`, which `.toLowerCase()`s both sides). A line is reported **once** even if the pattern occurs multiple times on it (line-level matches). Line numbers are **1-based** (`i + 1`), matching `file_search`.
4. **Context (FR-007).** When `context = n > 0`, extend each match to include up to `n` preceding and `n` following lines, keeping the matching line identifiable (e.g. a `>` marker on the match line, as `file_search`-style output already distinguishes matched text — exact marker is a Developer detail). Context windows may overlap between adjacent matches; de-duplicate so each line appears once.
5. **Bound the output (FR-011/FR-012…NFR-002).** Cap the total number of returned matches (e.g. `MAX_GREP_RESULTS`) and truncate each returned line to a bounded length (mirroring `file_search`'s `.slice(0, 200)`). When a cap is hit, **append an explicit truncation note** stating how many further matches were withheld — never a silent partial answer.
6. **Format (FR-005/NFR-003).** Return `JSON.stringify` of the match rows in the FILES group's `path:line:text` convention so `file_grep` and `file_search` outputs are uniform. A valid file with zero matches returns a clean zero-match result (distinguishable from the FR-008 error per FR-010) — a non-empty, signal-bearing "no matches" form, not a bare blank and not an error.

Constants to add alongside `MAX_SEARCH_RESULTS`/`MAX_GLOB_RESULTS` (`files.ts:15-16`) — e.g. `MAX_GREP_RESULTS` (match count cap) and a per-line length cap reusing the existing `shorten`/`.slice(0, 200)` convention. Exact numeric values are an implementation detail settled here at plan: pick values comfortably below the FILES tool's existing ~5 MB / result budget so a single huge or densely-matching file cannot exhaust model context (NFR-002). Do **not** reference `SEARCHABLE_EXT` anywhere in the `file_grep` path (FR-009).

### The capability + alias correction (FR-012/FR-013)

In the FILES block of `src/lib/agent/capabilities-registry.ts` (next to `file_search` at `:117` and `file_glob` at `:118`), add:

```
{ id: "file_grep", group: "files", aliases: [ <single-file-grep vocabulary> ], context: "both", description: "Search a SINGLE named VFS file for a literal string; returns matching lines with line numbers (path:line:text)." },
```

- **`context: "both"`** — exposed on both surfaces (main chat + delegated sub-agent), matching the other FILES tools and FR-001 ("callable by agents (main chat and delegated sub-agents)").
- **Description must say "single file"** so it is distinguishable from `file_search`'s "across a VFS subtree" (FR-012, SC-007).
- **FR-013 alias correction**: today the bare alias `"grep"` lives only on `file_search` (`:117`) and is what mis-routes single-file intent onto a tool that cannot serve it. The correction is to make single-file-grep intent satisfiable by `file_grep` (give `file_grep` the single-file aliases — e.g. `"grep this file"`, `"search in one file"`, `"find text in a file"`, and the bare `"grep"` so it is no longer claimed by `file_search` alone for the single-file case) while keeping `file_search` the directory-subtree tool (give it a directory-flavored alias where needed so directory intent still lands there). `file_search` keeps its directory role; the two are distinguishable, not collapsed (SC-007). Exact alias strings are a Developer detail; the requirement is the routing behavior in SC-007 / FR-013, verified by a discovery test.

### Discovery routing is tested, not hand-rolled

`file_grep` becomes discoverable automatically because the discovery system (`src/lib/assistant/tools/server/discovery.ts`, ranked by `src/lib/agent/discovery-score.ts`) searches over id / description / curated aliases / group. There is **no** separate per-tool routing code to write — the FR-013 requirement is satisfied purely by the capability row's description + aliases. The FR-013/SC-007 assertion is therefore a **test**: extend/`tests/assistant/discovery-modes.test.ts`-style assertions that (a) a single-file-grep query surfaces `file_grep`, (b) a directory-search query still surfaces `file_search`, (c) `file_grep`'s capability description contains "single file" (the test already enumerates `FILES` via `listCapabilities()` at `discovery-modes.test.ts:16`).

### What is NOT changed (FR-014 — additive only)

- `file_search`, `file_glob`, `file_edit`, `file_patch`, and all other tools: no behavior change (SC-008). The only edits to `capabilities-registry.ts` are the new `file_grep` row and the FR-013 alias strings on the FILES rows; the only edits to `files.ts` are the new `file_grep` tool plus its two constants. `withFeatureScope`, `walkFiles`, `globToRegExp`, `relativeToRoot`, `SEARCHABLE_EXT`, `MAX_SEARCH_RESULTS`, `MAX_GLOB_RESULTS`, and the existing tools are left intact.

### TDD requirement (NFR-004)

Tests are written **first** and MUST fail on the unfixed base (SC-009). The natural "fail on base" property: on the unfixed base `file_grep` does not exist, so any assertion that `fileTools().file_grep` is defined / that a single-file grep returns line-numbered matches fails; and the FR-013 discovery assertion (single-file intent → `file_grep`) fails because the capability row is absent. The new suite must cover at least: correct 1-based line numbers + no cross-file content (US1/SC-001); case-insensitive vs sensitive (SC-004); context windows (SC-005); non-file path → explicit error, never `[]` (US2/SC-002: directory, missing, unreadable); zero-match distinguishable from error (SC-003); truncation + withheld-count on an over-cap file (SC-006); binary/undecodable explicit error (FR-009); capability description "single file" + FR-013 routing (SC-007); `file_search`/`file_glob` unchanged (SC-008). Hermetic via `useTestDataDir` + `_stub-server-only`, invoking `fileTools().file_grep.execute!(input, ctx)` directly (pattern from `fetch-to-path.test.ts` / `discovery-modes.test.ts`).
