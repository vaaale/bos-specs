# Feature Specification: Browser Fill Form Schema Fix

**Feature Branch**: `bos/self-heal-0044`

**Created**: 2026-09-24

**Status**: Draft

**App Target**: bos-core

**Input**: Self-heal case EHS-0044 (scope class e / bos-core). The `browser_fill_form` tool's declared schema names the per-field element key `ref`, but the pinned `@playwright/mcp` server it proxies to validates `target`. The declaration and the implementation disagree, so an agent that follows BOS's own declared contract can never succeed. Fix the declared schema so it matches the proxied dependency's real contract.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Agent fills a multi-field form in one call (Priority: P1)

An agent drives a real browser and needs to populate several form fields (e.g. a username and password) in a single tool call. It inspects the latest `browser_snapshot`, reads the element refs (`e8`, `e9`, …), and calls `browser_fill_form` with one object per field. Each field object carries the human-readable field name, the field type, the element ref from the snapshot, and the value to type.

**Why this priority**: This is the only path the tool exists to support. As declared today, every `browser_fill_form` call that follows BOS's own schema is rejected by the proxied server before it reaches the page — the tool is effectively broken. Making this work is the entire point of the feature.

**Independent Test**: With Browser Automation enabled in Settings, open a login page, take a snapshot, then call `browser_fill_form` with two fields using the element refs from the snapshot (keyed as the declared schema requires). The call succeeds and both fields are populated on the page. This is testable with just the tool + a live page; no other feature is involved.

**Acceptance Scenarios**:

1. **Given** a live page with a username and password field, and a `browser_snapshot` exposing element refs for both, **When** the agent calls `browser_fill_form` with `fields: [{ name, type, target, value }, { name, type, target, value }]` (using the key the declared schema names for the element), **Then** the proxied `@playwright/mcp` server accepts the arguments, both fields are filled, and the call returns the resulting page snapshot rather than a validation error.
2. **Given** the declared schema for `browser_fill_form`, **When** an agent enumerates the required keys of each item in `fields`, **Then** the element key is `target` (not `ref`), and the required set is exactly `{ name, type, target, value }`.

---

### User Story 2 — Schema stays in lock-step with the rest of the browser tool family (Priority: P2)

The other element-targeting browser tools (`browser_click`, `browser_type`, `browser_hover`, `browser_select_option`, `browser_evaluate`, `browser_take_screenshot`) already name their element key `target` via a shared `el` object. An agent should not have to learn two different names for "the element this action acts on" within the same tool family.

**Why this priority**: Consistency is what prevents the next conflation. The defect here arose precisely because `browser_fill_form` was hand-written and deviated from the shared pattern. Keeping the family uniform makes this class of mismatch harder to reintroduce.

**Independent Test**: Enumerate the element-target key used by every element-targeting `browser_*` tool's declared schema. Each names it `target`. No tool in the family uses `ref` as a parameter name.

**Acceptance Scenarios**:

1. **Given** the full `browserTools()` registry, **When** each element-targeting tool's declared schema is inspected, **Then** the per-call element key is `target` for `browser_fill_form` just as it already is for `browser_click`, `browser_type`, and the other element tools.

---

### Edge Cases

- **Empty `fields` array**: `browser_fill_form` is called with `fields: []`. The declared schema must still validate (the array is present); behavior of zero fields is whatever the proxied server defines — the fix does not change how the executor handles it, only the key name the model is told to pass.
- **Element ref that is a selector string**: the `target` value may be an exact snapshot ref (`"e12"`) or a unique selector, matching the description the rest of the family already uses. The rename preserves the same accepted value shape.
- **Re-entrancy / retry**: an agent that previously retried a failed `browser_fill_form` call verbatim (the observed failure mode) must see a *different* outcome on the same call shape once the schema is corrected — the retry that failed on `target` missing now passes because the model is instructed to supply `target`.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The declared schema of `browser_fill_form` MUST name the per-field element key `target`, matching the element-key name used by every other element-targeting `browser_*` tool and by the proxied `@playwright/mcp` server's real contract.
- **FR-002**: The declared schema of `browser_fill_form` MUST list `target` in each `fields` item's required set, so that the required keys per field are exactly `{ name, type, target, value }`.
- **FR-003**: The declared schema of `browser_fill_form` MUST NOT require or reference a per-field key named `ref`; no field in the `fields` item schema is keyed `ref`.
- **FR-004**: The `target` field description MUST state that the value is the element reference from the latest `browser_snapshot` (e.g. `"e12"`) or a unique selector — consistent with the description the shared element tools already give `target`.
- **FR-005**: The tool's executor MUST continue to forward its input to the proxied `@playwright/mcp` server unchanged; the fix MUST be a declared-schema change only, with no rename/translation layer added in the executor.
- **FR-006**: The fix MUST be accompanied by a regression test that fails on the unfixed base (schema declares `ref`) and passes once the schema declares `target`.

### Non-Functional Requirements

- **NFR-001**: The change MUST not alter the runtime behavior of any other `browser_*` tool; the declared schemas and executors of all tools except `browser_fill_form` remain byte-identical in effect.
- **NFR-002**: The change MUST not modify the Supervisor or BOS build configuration; it is confined to the browser-tools declaration module and its test coverage.
- **NFR-003**: Type-check and the existing test suite MUST pass after the change.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An agent that reads the declared `browser_fill_form` schema and passes `target` per field succeeds on the first call against the proxied server (0 of N such calls rejected with an "expected string, received undefined at `fields[*].target`"-class validation error), where before the fix the identical call shape failed 100% of the time.
- **SC-002**: 100% of element-targeting `browser_*` tools name their element parameter `target`; the count of `browser_*` tools that name it `ref` is 0.
- **SC-003**: The regression test for the `browser_fill_form` schema fails on the pre-fix code and passes on the post-fix code, with no other test in the suite regressing.

## Assumptions

- **A-001**: The pinned `@playwright/mcp` dependency's real `browser_fill_form` contract requires per-field keys `{ name, target, type, value }` (the human-readable snapshot *displays* `ref=e5`, but the tool *schema* names the parameter `target`) — this is corroborated by the diagnostics report and the dependency's documented tool reference.
- **A-002**: The executor is a pure passthrough (`proxy("browser_fill_form", input, ctx)`) that forwards input to the MCP server unchanged, so correcting the declared key name is sufficient; no server-side or proxy-side rename is needed.
- **A-003**: The minimal, complete fix is the `ref` → `target` rename on the per-field property plus the `required` array. Reusing the shared `el` object is not required for correctness because a fill-form field's human-readable key is `name` (a field name), not `el.element` (an element description) — so the field object intentionally does not spread `el`. This keeps the change within the proposed surface.
- **A-004**: Browser Automation is gated in Settings and enabled for the acceptance test; the test exercises the declared schema (what the model is told to pass), not a live multi-field form fill against a real site.
