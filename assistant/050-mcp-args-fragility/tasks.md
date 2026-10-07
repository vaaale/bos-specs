# Tasks: `mcp_tool_call` args-string fragility — actionable error + split guidance

**Branch**: `bos/self-heal-0060` | **Spec**: `spec.md` | **Plan**: `plan.md`

**Test-driven development is MANDATORY**: for each behavior change, write the failing test FIRST, watch it fail for the right reason, then implement until it passes. A test written after the fix does not prove the fix.

## Setup (P1)

- [ ] T001: Confirm the pre-fix baseline. Run the existing test suite for `src/lib/mcp/` (there is no `args.test.ts` yet) and confirm `parseToolArgs` on invalid JSON returns exactly `args is not valid JSON: ${raw}` with no recovery hint. (Verifies the test will target the real defect.)

## User Story 1 — A failed `mcp_tool_call` tells the agent how to recover (P1)

Independent test: a unit test on `parseToolArgs`'s invalid-JSON path asserts the error contains BOTH the `not valid JSON` signal AND the split/re-call guidance, and that the raw echo is preserved.

- [ ] T101: **Write the failing test first.** Create `src/lib/mcp/args.test.ts` with a test that calls `parseToolArgs('<an invalid JSON string>')` (e.g. a truncated object string) and asserts: (a) the returned `error` contains `not valid JSON`; (b) it contains the recovery guidance to **split the argument set across multiple `mcp_tool_call` invocations** (or shorten a long value) and re-call; (c) the raw input is still echoed. Run it and confirm it FAILS on the current code (no hint present).
- [ ] T102: Implement the fix in `src/lib/mcp/args.ts` — append a short, actionable recovery hint to the invalid-JSON `return` (line 20), preserving the existing `args is not valid JSON: ${raw}` text and the raw echo. Do NOT change the return shape or any non-invalid-JSON path.
- [ ] T103: Re-run the test — confirm it now PASSES.

## User Story 2 — The `args` parameter warns the agent to keep payloads modest (P1)

Independent test: assert the `mcp_tool_call` tool's `args` parameter description contains size/complexity + split-across-calls guidance while preserving the original contract.

- [ ] T201: Extend the `mcp_tool_call` `args` parameter description in `src/lib/assistant/tools/server/mcp.ts` (the `args: p.str(…)` argument) to add size/complexity guidance: for large or complex argument objects (long strings, many nested/escaped values), keep each `args` payload modest by splitting the argument set across multiple `mcp_tool_call` invocations. Preserve the existing contract wording (JSON object string matching the input schema; `{}` when the tool takes none).
- [ ] T202: Add a test asserting the `mcp_tool_call` `args` description contains the guidance. (Acceptable via: reading the tool schema returned by `mcpTools().mcp_tool_call` and asserting on the `args` parameter description string; or a focused unit test of the description constant. Choose whichever keeps the test honest against the actual exposed schema.)

## User Story 3 — The developer-facing MCP doc describes the contract + remedy (P2)

Independent test: `docs/dev/mcp/mcp.md`'s Gateway section states the `args`-as-JSON-string contract and the split remedy.

- [ ] T301: Update `docs/dev/mcp/mcp.md` (Gateway section) to document that `mcp_tool_call.args` is a single hand-escaped JSON object string and that large or complex payloads should be split across multiple `mcp_tool_call` invocations.

## Coverage & Regression (P1)

- [ ] T401: Ensure ≥60% line AND branch coverage on every modified file. `args.test.ts` MUST cover all `parseToolArgs` branches: invalid JSON (hint), valid object string (parsed), empty/`""`/`null` (`{}`), array JSON string (`must be a JSON object`), and non-string/non-object (`must be a JSON object string`). Add targeted tests for any uncovered branch.
- [ ] T402: Run the full affected test suite and confirm green. Run typecheck. Confirm the modified files are a subset of the plan's file list (only `src/lib/mcp/args.ts`, `src/lib/assistant/tools/server/mcp.ts`, `src/lib/mcp/args.test.ts`, `docs/dev/mcp/mcp.md`).

## Notes

- **HARD scope constraint**: You may modify ONLY the four files in the plan's file list. If the fix requires modifying any file not in that list, STOP and report the justification for each additional file rather than modifying it.
- Do NOT modify the Supervisor (`tools/supervisor/**`), BOS build configuration, or any `src/lib/self-heal/**` file.
- Do NOT introduce a size/token cap on `args` — valid large payloads must still parse.
