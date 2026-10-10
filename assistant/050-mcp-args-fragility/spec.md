# Feature Specification: `mcp_tool_call` args-string fragility — actionable error + split guidance

**Feature Branch**: `bos/self-heal-0060`

**Created**: 2026-10-07

**Status**: Implemented on preview (`bos/self-heal-0060`) — pending user Promote. Self-heal case EHS-0060.

**App Target**: bos-core

**Input**: Self-heal case EHS-0060 (scope class **e**, ownership **bos-core**). `mcp_tool_call` returned `Error: args is not valid JSON: {…}` — twice in a row (repeated-failure trigger, 2 consecutive within 300s) — while calling `comfy-mcp.set_workflow_slot`. The echoed `args` string looked like *complete, valid* JSON (9 overrides, including a ~1,500-character image prompt with embedded double-quotes). The model first mis-attributed the cause to unicode em-dashes, retried in pure ASCII (failed identically), and only succeeded on the third attempt by **splitting the payload** across two modest `mcp_tool_call` invocations. Its own conclusion, verbatim: *"it was the combined payload size that tripped the proxy — splitting it solved it."*

## Context & Root Cause

The failing layer is BOS's own tool code, not the MCP server. `mcp_tool_call` declares its `args` parameter as a **JSON *string*, not an object**, with the rationale stated in-line (`src/lib/assistant/tools/server/mcp.ts`, `args: p.str(…)`): an object-typed parameter with no declared sub-properties gets closed down to `{}` by some providers, so the model is required to hand-escape an arbitrary argument object into one big string. That string is parsed by `parseToolArgs`, which does `JSON.parse(raw)` in a `try` and, on a throw, returns a bare string (`src/lib/mcp/args.ts:20`):

```
return { error: `args is not valid JSON: ${raw}` };
```

The tool returns that error **before** any MCP round-trip (`mcp.ts`: `const parsed = parseToolArgs(input.args); if (parsed.error) return \`Error: ${parsed.error}\`;`), so `callServerTool` / gateway schema validation never runs — `comfy-mcp` was never contacted for the failing call. A large hand-escaped string with many nested quotes is exactly the shape that breaks at `JSON.parse` (the model's streamed `arguments` are assembled by concatenating deltas then parsed), while each shorter half parses cleanly.

The durable, fixable gap: `mcp_tool_call.args` exposes a hand-escaped single-string argument that is fragile for large/quote-heavy payloads, yet neither its parameter description nor its failure return tells the model that *size/escaping of one big `args` string* is the likely cause, nor that the remedy is to **split the argument set across multiple calls**. The model therefore had to guess the fix (unicode → length) and burn two retries — a repeatable failure mode the self-heal trigger confirms.

**Confirmed exclusions** (from the diagnostic): not a `comfy-mcp`/`set_workflow_slot` problem (error produced in BOS's tool layer before any round-trip); not a schema/validation error (validation runs later in the gateway, and the payload is schema-valid); not unicode (the ASCII-only retry failed identically); not token-budget truncation (default `max_tokens` is 65,535, far above a ~5 KB payload); not self-heal bucketing (`unhandled_exception` is just the deterministic fall-through).

## Clarifications

### Session 2026-10-07 (self-heal pre-authorized)

- Q: Add a hard size cap on `args`, or make the failure actionable? → A: **Actionable guidance, no cap.** Enforcing a byte/token limit would be a new behavior change beyond the diagnostic's finding and could reject *valid* large payloads; the durable fix is (a) an actionable error and (b) proactive description guidance, both pointing the model to the recovery step it otherwise discovers by guessing.
- Q: Does the fix change the `args`-as-JSON-string design? → A: **No.** The string design is retained (its rationale — providers strip no-sub-property object params to `{}`) is intact and correct. Only the `args` parameter *description text* and the invalid-JSON *error string* change, plus the developer doc.
- Q: Does the invalid-JSON error preserve the existing echo of the raw input? → A: **Yes.** The hint is *appended* to the existing `args is not valid JSON: ${raw}` text, not a replacement — the raw echo is diagnostic and must be preserved.
- Q: Are the other `parseToolArgs` error paths (empty, non-string, non-object/array) in scope? → A: **No.** Only the invalid-JSON path gains the hint. The other paths keep their exact current strings.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A failed `mcp_tool_call` tells the agent how to recover (Priority: P1)

An agent builds a large, quote-heavy `args` payload and `parseToolArgs` fails to `JSON.parse` it. Today the agent gets a bare `args is not valid JSON: {…}` that names no cause and offers no remedy, so it mis-attributes the fault (unicode) and burns retries. After the fix, the error names the likely cause (one big hand-escaped string) and gives the recovery step — split the argument set across multiple `mcp_tool_call` invocations (or shorten a long value) and re-call.

**Why this priority**: This is the core defect. The repeated-failure trigger exists precisely because the error was not actionable, forcing the model to guess the fix.

**Independent Test**: Call `parseToolArgs` with a string that is not valid JSON and assert the returned error (a) still contains the `not valid JSON` signal and (b) contains the split/re-call guidance. This test fails on the pre-fix base and passes on the fix.

**Acceptance Scenarios**:

1. **Given** an `args` value that throws under `JSON.parse` (e.g. a truncated or double-escaped large object string), **When** `mcp_tool_call` runs, **Then** the returned error contains `not valid JSON` **and** instructs the agent to split the argument set across multiple `mcp_tool_call` invocations (or shorten a long value) and re-call.
2. **Given** the same failing input, **When** the agent reads the error, **Then** the raw `args` string is still echoed (diagnostic preserved) — the hint is appended, not a replacement.

---

### User Story 2 - The `args` parameter warns the agent to keep payloads modest (Priority: P1)

Before the first failure, the `mcp_tool_call` tool's `args` parameter description should proactively tell the agent that a large or complex argument object (long strings, many nested/escaped values) is best kept modest by splitting across calls — reducing the chance the fragile single-string path is hit at all.

**Why this priority**: Proactive guidance prevents the failure in the first place; together with US1 it makes the tool self-explaining in both directions (up-front warning + in-band remedy).

**Independent Test**: Read the `mcp_tool_call` tool schema and assert the `args` parameter description contains size/complexity fragility + split-across-calls guidance.

**Acceptance Scenarios**:

1. **Given** the `mcp_tool_call` tool's declared schema, **When** the agent inspects the `args` parameter description, **Then** it notes that large or complex argument objects should be kept modest by splitting the argument set across multiple calls.
2. **Given** the same description, **When** the agent reads it, **Then** the existing contract ("Arguments as a JSON object string matching the tool's input schema… Use `{}` if the tool takes none") is preserved.

---

### User Story 3 - The developer-facing MCP doc describes the args-string contract and remedy (Priority: P2)

The MCP developer doc (`docs/dev/mcp/mcp.md`, Gateway section) describes the agent-facing tools but does not state that `mcp_tool_call.args` is a hand-escaped JSON object string or that large/quote-heavy payloads should be split. After the fix the doc reflects the contract and the remedy, so the behavior is documented rather than living only in the code string.

**Why this priority**: A fix that leaves the docs wrong is not done (Constitution VI). This is a doc-completeness guarantee, not a behavior change.

**Independent Test**: Read `docs/dev/mcp/mcp.md`'s Gateway section and assert it states the `args`-as-JSON-string contract and the split-remedy for large/complex payloads.

**Acceptance Scenarios**:

1. **Given** the updated MCP doc, **When** a developer reads the Gateway section, **Then** it states that `mcp_tool_call.args` is a single hand-escaped JSON object string and that large or complex payloads should be split across multiple calls.

---

### Edge Cases

- **Valid JSON object string**: `parseToolArgs` parses it to the object as before — unchanged.
- **Empty / `""` / `null`**: returns `{ args: {} }` — unchanged.
- **Array JSON string** (e.g. `"[1,2]"`): returns the existing `args must be a JSON object, got: …` error — unchanged.
- **Non-string, non-object** (number, boolean): returns the existing `args must be a JSON object string; got <type>.` error — unchanged.
- **Very large but *valid* JSON**: still parses fine — no size cap is introduced; the fix is guidance, not enforcement.
- **Hint length**: the appended recovery text is short (a sentence or two), not a paragraph — it must not bloat the model's context.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `parseToolArgs`'s invalid-JSON return MUST append a short, actionable recovery hint to the existing `args is not valid JSON: ${raw}` text. The hint MUST state that `args` is a single hand-escaped JSON object string and that, when it fails on a large or quote-heavy payload, the caller should **split the argument set across multiple `mcp_tool_call` invocations** (or shorten a long value) and re-call. The raw `args` echo MUST be preserved.
- **FR-002**: The `mcp_tool_call` tool's `args` parameter description MUST add size/complexity guidance: for large or complex argument objects (long strings, many nested/escaped values), keep each `args` payload modest by splitting the argument set across multiple `mcp_tool_call` invocations. The existing description contract (JSON object string matching the input schema; `{}` when the tool takes none) MUST be preserved.
- **FR-003**: The developer-facing MCP doc (`docs/dev/mcp/mcp.md`, Gateway section) MUST document that `mcp_tool_call.args` is a single hand-escaped JSON object string and that large or complex payloads should be split across multiple calls.

### Non-Functional Requirements

- **NFR-001**: The change MUST NOT alter `parseToolArgs`'s return shape (`{ args?, error? }`) or any non-invalid-JSON path — only the invalid-JSON error *string* gains the appended hint. `mcp.ts`'s `if (parsed.error) return …` handling is unchanged.
- **NFR-002**: The change MUST NOT introduce a size or token cap on `args`; valid large payloads MUST still parse.
- **NFR-003**: The `mcp_tool_call` tool's schema shape MUST be unchanged (same parameters: `server`, `tool`, `args`); only the `args` description text and the invalid-JSON error string change. No new tools, no signature changes, no change to the gateway's schema-validation path.
- **NFR-004**: The fix MUST NOT modify the Supervisor (`tools/supervisor/**`), BOS's build configuration, or any `src/lib/self-heal/**` file.

### Key Entities

- **`args` (string)** — the single hand-escaped JSON object string a model passes to `mcp_tool_call`; parsed by `parseToolArgs`. The contract is unchanged; its fragility and the recovery remedy are now documented (description + error + doc).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A regression test asserts that `parseToolArgs` on invalid JSON returns an error containing **both** the `not valid JSON` signal **and** the split/re-call guidance; the test fails on the pre-fix base and passes on the fix.
- **SC-002**: The `mcp_tool_call` `args` parameter description, as exposed to the agent, contains size/complexity fragility + split-across-calls guidance while preserving the original contract wording.
- **SC-003**: Every non-invalid-JSON `parseToolArgs` path is behaviorally unchanged: valid object string → parsed object; empty/`null` → `{}`; array string → `must be a JSON object` error; non-string/non-object → `must be a JSON object string` error.
- **SC-004**: `docs/dev/mcp/mcp.md`'s Gateway section describes the `args`-as-JSON-string contract and the split remedy.
- **SC-005**: The actually-modified files are a subset of the plan's file list (verified at converge): `src/lib/mcp/args.ts`, `src/lib/assistant/tools/server/mcp.ts`, `tests/mcp/args.test.ts` (new), `docs/dev/mcp/mcp.md`, and `tests/assistant/mcp-server-list-allowlist.test.ts` (one stale exact-equality assertion updated to a prefix + hint-containment assertion, as a necessary consequence of FR-001).

## Assumptions & Dependencies

- Builds on `014-mcp-tool-gateway` (the gateway and the agent-facing `mcp_tool_call`). This feature closes the non-actionable-error + missing-guidance gap the diagnostic (EHS-0060) found; it does not re-spec the gateway.
- The `args`-as-JSON-string design is retained per `014`'s stated rationale; only the description/error text and the developer doc change. No change to the gateway's schema-validation path (`src/lib/mcp/gateway.ts`) or to the MCP client.
- A unit test for `parseToolArgs` does not currently exist (it is pure and self-described as unit-testable); this feature adds `tests/mcp/args.test.ts` (colocated with the other MCP unit tests under the Playwright unit runner) to lock the new hint and the preserved paths.
- Scope boundaries: no Supervisor, build-config, or `src/lib/self-heal/**` change; no hard payload cap; the fix is prose in the error string + parameter description + docs only.
