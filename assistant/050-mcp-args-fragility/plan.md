# Implementation Plan: `mcp_tool_call` args-string fragility — actionable error + split guidance

**Branch**: `bos/self-heal-0060` | **Date**: 2026-10-07 | **Spec**: `/Specs/user-specs/assistant/050-mcp-args-fragility/spec.md`

**Input**: Feature specification from spec.md (self-heal case EHS-0060, scope class e / bos-core).

## Summary

The agent-facing `mcp_tool_call` tool passes its arguments as a single hand-escaped JSON object **string** (`args`), parsed by `parseToolArgs`. When that string is large or quote-heavy, `JSON.parse` throws and the tool returns a bare `args is not valid JSON: {…}` that names no cause and offers no remedy, forcing the model to guess the fix and burn retries (the EHS-0060 incident: a ~5 KB 9-override `comfy-mcp.set_workflow_slot` payload). The fix is prose-only, at three surfaces: (1) append an actionable recovery hint to `parseToolArgs`'s invalid-JSON error, (2) add size/complexity + split-across-calls guidance to the `mcp_tool_call` `args` parameter description, and (3) document the contract in `docs/dev/mcp/mcp.md`. No size cap, no signature change, no gateway-path change.

## Technical Context

**Language/Version**: TypeScript (BOS source tree)
**Primary Dependencies**: none new — edits text within existing `parseToolArgs` (`src/lib/mcp/args.ts`) and the `mcp_tool_call` tool schema (`src/lib/assistant/tools/server/mcp.ts`).
**Storage**: N/A
**Testing**: `vitest` — new `src/lib/mcp/args.test.ts` (unit, pure function; TDD: write failing test first).
**Target Platform**: BOS (Next.js) server runtime
**Project Type**: in-place modification of an existing feature (`014-mcp-tool-gateway`)
**Performance Goals**: N/A (text-only change)
**Constraints**: ≥60% line **and** branch coverage on every modified file; no change to `parseToolArgs` return shape or non-invalid-JSON paths; no Supervisor/build-config/`src/lib/self-heal/**` change; no payload cap.
**Scale/Scope**: 2 existing files edited, 1 new test file, 1 doc file edited.

## Constitution Check

*GATE: Must pass before implementation.*

- **Simplicity**: PASS. The change adds no abstraction, no new mechanism, no size enforcement — it makes an existing error actionable and the parameter self-explaining. A smaller alternative (hint only, no description/doc) exists but is rejected because NFR/SC-004 requires the doc and FR-002 requires the proactive description; all three are single-string/doc edits with no structural cost.
- **Testability**: PASS. `parseToolArgs` is pure and already unit-testable; a regression test on the invalid-JSON path fails on the pre-fix base and passes on the fix. Coverage target ≥60% line+branch applies to `args.ts` (all branches are covered by the new test) and `mcp.ts` (the description is a string literal; behavior is covered by the `args.ts` test plus the existing tool's unchanged flow).
- **Documentation (Constitution VI)**: PASS. The behavior change is reflected in `docs/dev/mcp/mcp.md` (Gateway section) in the same change.
- **No Supervisor / build-config change (FR-023 of 031)**: PASS. File list does not touch `tools/supervisor/**` or build config.

*Re-check after design*: no design step ran (small, single-subsystem change); re-checked — still PASS.

## Project Structure

### Documentation (this feature)

```text
/Specs/user-specs/assistant/050-mcp-args-fragility/
├── spec.md              # specify output
├── plan.md              # This file (plan output)
└── tasks.md             # tasks output
```

No `research.md` / `data-model.md` / `contracts/` — the change is a two-string + doc + test edit to a pure function and one tool schema; none of those artifacts is warranted.

### Source Code (repository root)

```text
src/
├── lib/
│   ├── mcp/
│   │   ├── args.ts              # MODIFY: append recovery hint to invalid-JSON return (line 20)
│   │   └── args.test.ts         # NEW: unit tests — invalid-JSON hint + preserved paths
│   └── assistant/tools/server/
│       └── mcp.ts               # MODIFY: extend mcp_tool_call `args` param description
docs/
└── dev/mcp/mcp.md               # MODIFY: document args-as-JSON-string contract + split remedy
```

**Structure Decision**: In-place modification of the existing `014-mcp-tool-gateway` code. The only files that change behavior are `args.ts` (the `parseToolArgs` pure function) and `mcp.ts` (the `mcp_tool_call` tool's `args` description string). The new `args.test.ts` locks the behavior; the `docs/dev/mcp/mcp.md` edit satisfies Constitution VI. No other file is touched.

### File list (HARD scope constraint for `implement`)

The Developer may modify ONLY these files:

1. `src/lib/mcp/args.ts`
2. `src/lib/assistant/tools/server/mcp.ts`
3. `src/lib/mcp/args.test.ts` (new)
4. `docs/dev/mcp/mcp.md`
