# Feature Specification: file_grep — Single-File Content Search Tool

**Feature Branch**: `bos/self-heal-0002`

**Created**: 2026-09-08

**Status**: Draft

**App Target**: bos-core

**Input**: Self-heal case `EHS-0002` (scope class **e**, ownership **bos-core**). Filed feature request (conversation c-mtsumdoi9ax2): "I and other agents keep trying to use `file_search` to find content inside a file, which `file_search` can't do. Add a dedicated `file_grep` tool — file path + pattern, with optional flags for line numbers, context lines, and case-insensitivity."

## Context: what exists today

Four facts about the current implementation shape this spec. All are grounded in the self-heal diagnostic report (`/Documents/BOS Improvements/self-heal-0002.md`).

1. **`file_search` is a directory-subtree search, not a single-file grep.** It is declared in `src/lib/assistant/tools/server/files.ts` (`fileTools()`, `~:156`) as `parallel(serverTool(...))`. Its description says "Search file content for a literal string **under a VFS directory**," and its schema types the scope as `path: p.str("VFS directory to search under")` (`~:161`). There is no parameter for a single file.

2. **`file_search` cannot target one file — and fails silently.** The handler calls `walkFiles(root, 5000)` (`~:170`), a BFS that seeds `queue = [root]` and, each iteration, does `vfs.list(dir)` then pushes only `type === "dir"` entries back on (`:36-42`). `root` is expected to be a directory. Pass a file path and `vfs.list()` resolves to `fs.readdir(real)` (`src/os/vfs.ts:140-167`); `fs.readdir` on a regular file throws, which `walkFiles` swallows with `.catch(() => [])` (`:37`). The result loop never runs and `file_search` returns `[]` — **no error is surfaced**. An agent that wanted to grep one named file gets an empty array and no explanation.

3. **`file_search` is deliberately narrower than a general grep.** It only reads files whose extension is in a fixed whitelist `SEARCHABLE_EXT = { ".md", ".txt", ".json", ".yml", ".ts", ".tsx", ".js", ... }` (`files.ts:17-20`). A binary or otherwise-unlisted file is skipped regardless of how the call is shaped. A single-file grep must *not* inherit that restriction — the whole point is to search the one file the agent named.

4. **The `"grep"` alias is mis-pointed, and the wiring for a new tool is local.** `file_search` is the only grep-like capability, aliased `{ id: "file_search", group: "files", aliases: ["grep", "find text", "mention", "occurrence", "where is"] }` at `src/lib/agent/capabilities-registry.ts:117`. It is the literal string `"grep"` in that alias list that pulls agents toward `file_search` for single-file work, where it structurally cannot succeed. A new `file_grep` tool follows the existing two-file pattern: add the tool to `fileTools()` in `files.ts` (registered in the assistant registry at `src/lib/assistant/registry.ts:109` via `...fileTools()`) and add one capability row to the FILES block of `capabilities-registry.ts` (next to `file_search` at `:117` and `file_glob` at `:118`).

**Verdict (class e, not b).** There is no correct invocation of any existing tool that returns matching lines (with line numbers) from one named file: `file_search` cannot scope to a file, is whitelist-limited, and emits cross-directory results. Per the b-vs-e test ("if the tool *cannot* do the thing → e; the test is whether a correct invocation exists at all"), no such invocation exists. This is a genuine BOS-core source gap. `run_command` + shell `grep` is a workaround, not the fix — it is shell-bound, requires the agent to know the bind-mount layout, and is not a first-class FILES-group tool.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — An agent greps one named file and gets line-numbered matches (Priority: P1)

An agent (main chat or a delegated sub-agent) needs to find where a piece of text occurs *inside a specific file it already knows the name of*. Today it reaches for `file_search`, the only grep-like tool in the FILES group, and gets an empty array with no explanation — because `file_search` is a directory search and silently cannot target a file. With this feature the agent calls `file_grep` with the file path and the pattern, and gets back every matching line of that one file, each with its line number, in the same `path:line:text` convention the group already uses.

**Why this priority**: This is the whole point — the missing capability. Without it, single-file grep has no correct tool and the agent either gets a silent empty result or is forced onto a shell workaround.

**Independent Test**: Create a temp VFS file with several lines, some containing a known token. Call `file_grep` on that file for the token and confirm the response lists exactly the matching lines, each tagged with its correct 1-based line number, and that no other file's content appears (it is a single-file search).

**Acceptance Scenarios**:
1. **Given** a VFS file whose lines 3 and 7 contain the token `foo`, **When** the agent calls `file_grep` with that file's path and pattern `foo`, **Then** the response contains exactly those two matching lines, each with its correct 1-based line number, in `path:line:text` form.
2. **Given** a VFS file that contains the token nowhere, **When** `file_grep` is called for it, **Then** the response reports zero matches in a way that is distinguishable from an error (an empty/zero result, not a silent blank, not a thrown failure).
3. **Given** a VFS file with a known token, **When** `file_grep` is called with case-insensitivity enabled and the pattern differs from the token only in case, **Then** the matching lines are still returned.
4. **Given** a VFS file with a known token, **When** `file_grep` is called with a context-lines flag, **Then** each matching line is returned together with the requested number of surrounding lines, with the matching line identifiable within its context block.

### User Story 2 — Passing a non-file path fails loudly instead of silently (Priority: P1)

The signature failure of the missing tool was that the only available path returned an empty array with no error, so the agent concluded "no match" and moved on. A correct single-file grep must be unambiguous about what it was given: if `path` is not a readable file (a directory, a missing path, or an unreadable file), the tool reports that explicitly rather than returning `[]`.

**Why this priority**: Equal to US1 in importance — the silent `[]` is the exact defect being fixed. A grep that fails as quietly as it used to is not a fix.

**Independent Test**: Call `file_grep` with (a) a directory path, (b) a non-existent file path, and (c) an existing regular file. Confirm (a) and (b) return an explicit error naming the problem, while (c) returns matches (or a clean zero-match) — never a silent empty array for a path that is not a file.

**Acceptance Scenarios**:
1. **Given** a VFS **directory** path, **When** `file_grep` is called with it, **Then** the response is an explicit error stating the path is not a file (and, where useful, that `file_search` is the directory-scoped tool) — not `[]`.
2. **Given** a **non-existent** file path, **When** `file_grep` is called with it, **Then** the response is an explicit error that no file exists at that path — not `[]`.
3. **Given** a path the agent is **not permitted to read**, **When** `file_grep` is called with it, **Then** the response is an explicit permission error — not a silent empty array.

### User Story 3 — `file_grep` is discoverable as the single-file grep (Priority: P2)

Agents discover tools through the FILES group and through search/discovery aliases. Once `file_grep` exists, the group's discovery must route "grep one file"-style intent to it, and the tool's own description must make clear that it searches a **single named file** (as distinct from `file_search`, which searches a directory). Today the bare alias `"grep"` points at `file_search`, which is precisely what mis-routes single-file intent onto a tool that can't serve it.

**Why this priority**: A tool that exists but isn't found, or whose description doesn't disambiguate it from `file_search`, leaves the original failure in place. Lower than P1 because the tool's core value (US1/US2) is real once it is callable directly.

**Independent Test**: With `file_grep` registered, confirm (a) its capability row sits in the FILES group with a description that says "single file," and (b) a natural-language discovery for "grep this file / find text in one file" surfaces `file_grep`, while directory-subtree search still surfaces `file_search`.

**Acceptance Scenarios**:
1. **Given** `file_grep` is registered, **When** the capability registry is inspected, **Then** there is a `file_grep` capability in the `files` group whose description states it searches a **single file** (as distinct from `file_search`'s directory search).
2. **Given** the FILES group, **When** a discovery query expresses single-file grep intent (e.g. "grep this file"), **Then** `file_grep` is surfaced for that intent. The bare `"grep"` alias is **not** claimed by `file_search` alone for the single-file case.
3. **Given** the FILES group, **When** a discovery query expresses directory-subtree search intent, **Then** `file_search` is still surfaced — the two tools are distinguishable, not collapsed.

### Edge Cases

- **Empty file.** A regular file with zero bytes is a valid target; a search returns a clean zero-match (not an error, not `[]`-with-no-signal distinct from the non-file error in US2).
- **Match on every line / very many matches.** A common one-character pattern can match thousands of lines. The response MUST be bounded (see NFR) and MUST state when it was truncated and how many matches were withheld, so the agent is never silently given a partial answer.
- **Very long lines.** A single line far longer than the result budget (e.g. a minified file or a long log line) MUST NOT flood the response; each returned line is truncated to a bounded length with a marker, and the line number remains correct.
- **Binary / non-text file named explicitly.** Because the whole point is to search the one file the agent named, `file_grep` does NOT skip files by extension (unlike `file_search`'s `SEARCHABLE_EXT` whitelist). If the file cannot be decoded as text, the tool reports that explicitly rather than silently returning nothing.
- **Multiple matches on one line.** A pattern occurring more than once on a single line still yields that line once (line-level matches, not per-occurrence), matching `path:line:text` semantics; the agent can re-read the line if it needs per-occurrence detail.
- **Path that is a VFS mount point or a directory symlink.** Treated as a directory → explicit non-file error (US2), not a partial walk.
- **Trailing / leading whitespace in the pattern.** The pattern is matched as given (literal substring); the tool does not trim or otherwise mutate it, so the agent can search for an exact literal including surrounding spaces.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST expose a `file_grep` server tool in the FILES group, callable by agents (main chat and delegated sub-agents), that searches a **single named VFS file** for a pattern and returns the matching lines of that file only.
- **FR-002**: `file_grep` MUST accept a `path` argument identifying the single file to search (a VFS file path, not a directory) and a `pattern` argument holding the text to search for.
- **FR-003**: Matching MUST be a **literal substring** search — the pattern is matched exactly as given, with no regex interpretation — so a pattern of `a.b` matches the literal three characters, not a regular expression.
- **FR-004**: `file_grep` MUST be **read-only** (a single read of the named file; no writes, no side effects) and MUST be classified **parallel-safe**, consistent with the existing `file_search`/`file_glob` read-only tools in the FILES group.
- **FR-005**: `file_grep` MUST return **line-level** matches: each matching line is reported once, in **1-based line number** order, in the same `path:line:text` form the FILES group already uses for `file_search`. Line numbers MUST be on by default (the group's existing convention) and MUST be correct for every returned line.
- **FR-006**: `file_grep` MUST provide an optional **case-insensitivity** flag; when enabled, matching ignores letter case, and when omitted or disabled, matching is case-sensitive (the default).
- **FR-007**: `file_grep` MUST provide an optional **context-lines** flag (a non-negative integer `n`); when set, each matching line is returned together with up to `n` preceding and `n` following lines, with the matching line identifiable within its context block. When omitted, no extra context is returned.
- **FR-008**: When `path` is **not a readable regular file** (a directory, a non-existent path, or an unreadable file), `file_grep` MUST return an **explicit in-band error** naming the problem — it MUST NOT return a silent empty array. This is the behavioral fix for the exact defect that motivated the feature.
- **FR-009**: `file_grep` MUST search the one file the agent names and MUST NOT skip it based on file extension. It MUST NOT be gated on `file_search`'s `SEARCHABLE_EXT` whitelist. If the named file cannot be decoded as text, it MUST report that explicitly (not silently return nothing).
- **FR-010**: A zero-match result on a valid file MUST be distinguishable from an error: a valid file with no matches returns a clean zero-match result, while a non-file path returns the FR-008 error.
- **FR-011**: `file_grep` MUST be bounded in output: the total number of returned matches and the length of any single returned line MUST be capped, and when a cap is hit the response MUST state that it was truncated and how many matches were withheld (no silent partial answers).
- **FR-012**: `file_grep` MUST be registered as a capability in the FILES group in the capability registry, with a description that states it searches a **single file** (distinguishing it from `file_search`'s directory search).
- **FR-013**: Discovery routing for FILES-group intent MUST distinguish single-file grep from directory search: single-file grep intent (e.g. "grep this file") MUST be satisfiable by `file_grep`, and the bare `"grep"` alias MUST NOT remain claimed by `file_search` for the single-file case in a way that hides `file_grep`. `file_search` MUST remain the tool for directory-subtree search.
- **FR-014**: `file_grep` MUST NOT modify, replace, or change the behavior of `file_search`, `file_glob`, or any other existing tool; the change MUST be additive (a new tool plus its capability registration and the FR-013 alias correction).

### Non-Functional Requirements

- **NFR-001 (Performance)**: A single-file grep over a file of up to ~5 MB (roughly the existing FILES tool budget) MUST complete in well under one second of CPU on the target platform. The tool reads the file once; it MUST NOT walk a directory tree.
- **NFR-002 (Resource bounds)**: The response size MUST be bounded by the FR-011 caps independent of how many lines match, so a single huge or densely-matching file cannot exhaust the model's context.
- **NFR-003 (Consistency)**: Line numbering and the `path:line:text` output form MUST be consistent with `file_search` so an agent treats the two tools' outputs uniformly.
- **NFR-004 (Testing coverage)**: The implementation MUST be covered by tests at ≥95% line and branch coverage on every file it modifies, including the FR-008 non-file-error branches, the FR-011 truncation branches, the FR-006/FR-007 flag branches, and the binary/non-decodable-file branch (FR-009). Tests MUST be written before the implementation and MUST fail on the unfixed base.
- **NFR-005 (Docs)**: Because this adds a new agent-facing tool, the relevant developer and usage docs under `docs/` MUST be updated in the same change to document `file_grep` and how it differs from `file_search`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For a file whose matches are on known lines, `file_grep` returns exactly those lines with correct 1-based line numbers — 0 wrong line numbers, 0 omitted matches, 0 matches from any other file.
- **SC-002**: Calling `file_grep` with a directory path, a missing path, and an unreadable path each yields an explicit error; the number of these cases that return a silent empty array is 0.
- **SC-003**: A valid file with no matches returns a clean zero-match result, and the agent (or a test) can distinguish that from an error — 0 ambiguity cases.
- **SC-004**: With case-insensitivity enabled, a pattern differing from the file's text only in case produces the same matches as the exact-case pattern; with it disabled, it produces 0 matches. The flag changes behavior exactly as specified.
- **SC-005**: With context-lines `n`, each match is returned with up to `n` preceding and `n` following lines and the matching line is identifiable; with no flag, 0 extra lines are returned.
- **SC-006**: For a file engineered to exceed the FR-011 match cap, the response is bounded, states that it was truncated, and states the number of matches withheld — the agent is never given a silent partial answer.
- **SC-007**: A `file_grep` capability exists in the FILES group whose description says "single file," and a single-file-grep discovery query surfaces `file_grep` while a directory-search query still surfaces `file_search` — the two are distinguishable, not collapsed.
- **SC-008**: `file_search`, `file_glob`, and all other existing tools behave identically before and after this change (additive only) — 0 regressions in their existing tests.
- **SC-009**: Test coverage on every modified file is ≥95% line and ≥95% branch, and the new test suite fails on the unfixed base (proving it exercises the fixed behavior, not a pre-existing green).

## Assumptions

- `file_grep` operates over the same VFS surface the other FILES tools do (unmounted host paths resolve through the existing `vfs` read path, mounted stores through their bridge); it reuses the existing read mechanism rather than introducing a new one.
- "Literal substring" (FR-003) is the intended matching semantics: the filed request asks to "find content inside a file" and names no regex need; a literal matches the existing `file_search` "literal string" convention and avoids regex-escaping and ReDoS surface. (Confirmed decision — see Clarifications; this is the one genuine choice the diagnostic left open.)
- Line numbers are 1-based and always present, matching the FILES group's `path:line:text` convention; the request named line numbers as "default on," and a grep whose output omits line numbers would be less useful than `file_search`, not more. (Confirmed decision — see Clarifications.)
- The tool is a single read of one file; it performs no directory walk, so the NFR-001 sub-second budget holds up to the existing FILES tool size budget.
- The bounded-output caps in FR-011 (match count and per-line length) are chosen to be comfortably below the model-context budget; exact numeric values are a design detail settled at `plan`, not a behavioral requirement.

## Clarifications

### Session 2026-09-08

Resolved autonomously (self-heal case, pre-authorized to run the pipeline without stopping); each decision is committed into the FRs above.

- **Q: Literal substring or regex for `pattern`?** **A: Literal substring (FR-003).** Rationale: the filed request only needs to "find content inside a file"; a literal matches `file_search`'s existing "literal string" semantics (NFR-003 consistency), and avoids regex-escaping, invalid-regex, and ReDoS failure surfaces. Regex is out of scope — it would be a larger, riskier addition the request does not ask for.
- **Q: Are line numbers a flag or always on?** **A: Always on (FR-005), 1-based.** Rationale: the request names line numbers as "default on, matching `file_search`'s `path:line:text` convention"; a grep that can't show *where* a match is is less useful than `file_search`, so there is no sensible use case for hiding them.
- **Q: Which optional flags?** **A: `ignoreCase` and `context` (FR-006, FR-007)** — the two the request named beyond line numbers ("context lines, case-insensitivity").
- **Q: Extension whitelist?** **A: None (FR-009).** A single-file grep searches the file the agent named; the `SEARCHABLE_EXT` gate exists to keep `file_search`'s directory walk fast and is the wrong tool for one named file.
- **Q: Non-file path behavior?** **A: Explicit in-band error (FR-008), never a silent `[]`.** This is the core defect being fixed (US2).
- **Q: Alias routing?** **A: Single-file grep intent must be satisfiable by `file_grep`; the bare `"grep"` alias must not stay claimed by `file_search` in a way that hides `file_grep` (FR-013).** `file_search` remains the directory tool. Exact alias text is a `plan` detail.
