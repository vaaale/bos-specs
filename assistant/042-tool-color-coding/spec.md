# Feature Specification: Tool State Color Coding in the Assistant Info Panel

**Feature Branch**: `bos/042-tool-color-coding`

**Created**: 2026-09-02

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "The Assistant app has a tab showing all the tools available. Add color coding to the tools. Tools granted to the selected agent should be green. Tools that are deferred and not discovered should be orange / dark yellow. Tools that are deferred and discovered should be blue. The rest should be as they are today."

## Context: what exists today

The Assistant's right-hand Info Panel (`src/components/agent/v2/InfoPanelV2.tsx`) has a
**Tools** tab (`ToolsTab`) that lists every capability in the live registry, grouped by
tool group. Today every tool row is rendered identically — a neutral wrench icon and
muted text — so the panel shows *what tools exist* but not *what state each tool is in
for the currently selected agent*.

Three of the four states the user wants to distinguish already have authoritative,
single-source-of-truth data; none of it is currently wired into the panel:

1. **Granted (allowlist)** — per-agent `tools: string[]`, returned by
   `/api/assistant/agent` and already fetched by the panel (it reads `skills`/`mcp`
   from the same response and ignores the tools/deferred fields). The panel renders
   the *full* registry for every agent, so it never reflects the selected agent's
   allowlist.
2. **Deferred** — per-agent `deferredTools: string[]`, also returned by the same
   route. A tool is "deferred" only insofar as it is *also granted*: a non-allowed
   tool is never visible regardless, so deferral is a sub-state of granted tools.
3. **Discovered (revealed)** — a deferred tool becomes *revealed* when a prior
   `find_tools` call in **this conversation** returned it. The canonical computation
   is `deriveRevealedIds(messages)` (`src/lib/assistant/messages.ts`), the exact set
   the run loop uses to decide which deferred tools are currently callable. This is
   conversation-scoped, not agent-scoped, so the panel — which is not
   conversation-aware today — needs it passed in.

The change is therefore: make the panel's Tools tab **agent-aware** (read
`tools`/`deferredTools` for the selected agent) and **conversation-aware** (receive
the revealed set), then color each row by the single matching state.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - See which tools the selected agent can actually use (Priority: P1)

A user switching between agents in the Assistant wants to glance at the Tools tab and
see, at a glance, which tools belong to the currently selected agent and which are
out of reach for it. Today the panel is identical for every agent, so nothing changes
when they switch — the allowlist is invisible.

**Why this priority**: This is the foundation the other states sit on. Without the
panel knowing the selected agent's allowlist it cannot classify *any* tool, and the
two deferred sub-states (which are sub-states of granted) have nothing to attach to.

**Independent Test**: Select two agents with different tool allowlists, open the Tools
tab for each, and confirm the granted tools render in the granted color and the
non-granted tools keep their existing neutral styling.

**Acceptance Scenarios**:

1. **Given** an agent granted a non-deferred tool, **When** the Tools tab renders,
   **Then** that tool's row is colored **green**.
2. **Given** a tool the selected agent is **not** granted, **When** the Tools tab
   renders, **Then** that row keeps its existing neutral styling (the "rest").
3. **Given** the user switches the selected agent, **When** the Tools tab re-renders,
   **Then** the color coding reflects the *newly* selected agent's allowlist, not the
   previous one's.

---

### User Story 2 - Distinguish hidden vs. revealed deferred tools (Priority: P1)

A tool the agent has but that is deferred is invisible to the model until the agent
discovers it via `find_tools`. A user reviewing a conversation wants to see which
deferred tools are still hidden (the agent cannot call them yet) versus which have
already been revealed (now callable). Today all deferred tools look exactly like
everything else.

**Why this priority**: This is the half of the request that carries real information —
the green/non-granted split only separates *who has it*, while this split separates
*can the agent use it right now*. It also introduces the conversation-awareness the
panel lacks.

**Independent Test**: Configure an agent with a deferred tool, open its Tools tab in a
fresh conversation (tool shows **orange / dark yellow**), then run a turn that calls
`find_tools` and reveals it, and confirm the same row is now **blue**.

**Acceptance Scenarios**:

1. **Given** a deferred tool the agent is granted and that has **not** yet been
   revealed in the current conversation, **When** the Tools tab renders, **Then** the
   row is colored **orange / dark yellow**.
2. **Given** the same deferred tool, **When** a `find_tools` result in the current
   conversation has revealed it, **Then** the row is colored **blue**.
3. **Given** a deferred tool the agent is **not** granted, **When** the Tools tab
   renders, **Then** it is treated as "the rest" (neutral) — deferral has no
   visible effect on a non-granted tool.
4. **Given** the user opens a *different* conversation where the tool was not
   revealed, **When** the Tools tab renders, **Then** the tool is orange / dark
   yellow again — the revealed state is per-conversation, not global.

---

### Edge Cases

- **Empty allowlist.** An agent whose `tools` list is empty has no granted registry
  tools; every registry tool row is "the rest" (neutral). (Per the run's gate, an
  empty allowlist means zero granted registry tools — the panel must mirror that,
  not treat empty as "all".)
- **Discovery tools.** `find_tools` and `find_agent` bypass the allowlist and are
  always available; they have no capability-registry entry. They MUST render with
  the existing neutral styling, not green, so as not to mislead about allowlisting.
- **Non-registry tools** (surface-registered or elicitation tools) have no capability
  entry and are not part of the `assistantToolsManifest()` list; the color coding
  MUST NOT invent rows for them — it only restyles rows that already render.
- **Revealed but not granted.** A tool the agent is not granted cannot meaningfully be
  "revealed" (it is never callable); if it somehow appears in the revealed set, it
  MUST be treated as "the rest", not blue.
- **Revealed and deferred and granted** is the blue case; **revealed and non-deferred**
  is the green case. A revealed non-deferred tool is simply always-visible and takes
  the green (granted) color, never blue — blue is reserved for the deferred/revealed
  pair.
- **Selected agent has no active conversation yet.** The revealed set is empty; all
  granted deferred tools are orange / dark yellow, all granted non-deferred are green.
- **Service tool lifecycle.** A marketplace service's tools enter/leave the registry
  as the service starts/stops (041-tool-groups). Color coding MUST work identically
  for dynamic service tools and built-ins; a row that disappears (service stopped) is
  simply no longer colored — no stale color persists.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The Tools tab MUST classify every rendered tool row into exactly one of
  four visual states: **granted** (green), **deferred-hidden** (orange / dark
  yellow), **deferred-revealed** (blue), or **neutral** (the existing default
  styling). No row MAY be unclassified.
- **FR-002**: A row MUST be colored **green** if and only if the tool is granted to
  the currently selected agent and is **not** in a deferred-revealed state (i.e. it is
  a non-deferred granted tool, or a deferred tool already revealed).
- **FR-003**: A row MUST be colored **blue** if and only if the tool is granted, is
  deferred, and has been revealed by a prior `find_tools` call in the current
  conversation.
- **FR-004**: A row MUST be colored **orange / dark yellow** if and only if the tool
  is granted, is deferred, and has **not** been revealed in the current
  conversation.
- **FR-005**: A row MUST keep its existing neutral styling (the "rest") if and only
  if the tool is not granted to the currently selected agent, or the tool is a
  discovery / non-registry tool (see Edge Cases). Neutral is the absence of a state,
  not a fifth color.
- **FR-006**: State determination MUST be derived from the same single sources the
  run uses — the selected agent's `tools` allowlist and `deferredTools` list, and the
  conversation's revealed set as computed by the canonical `deriveRevealedIds` — so
  the panel's colors can never diverge from what the agent can actually call.
- **FR-007**: The panel MUST become conversation-aware: it MUST receive the current
  conversation's revealed set (from the active conversation) and recompute states when
  the active conversation changes.
- **FR-008**: State MUST be recomputed when the selected agent changes, using the new
  agent's allowlist and deferred list.
- **FR-009**: The four colors MUST be visually distinguishable in the panel's existing
  dark theme, with the state encoded by a color cue on the row (icon and/or a state
  swatch) that does not alter the row's text content or layout.
- **FR-010**: A legend or inline affordance MUST make the meaning of each color
  discoverable, so a user who has not read the spec can read the panel (e.g. a small
  legend: green = granted, blue = revealed, orange = deferred/hidden, grey = not
  granted).
- **FR-011**: This change MUST apply to every surface that renders the Assistant Tools
  tab (the standalone Assistant and any embed that sets `showInfo`), because they all
  share `InfoPanelV2`/`ToolsTab`.
- **FR-012**: The existing per-group layout, the group headings, the per-tool name and
  description text, and the Skills / MCP tabs MUST be unchanged; this feature only adds
  a state color to tool rows and the legend.

### Key Entities

- **Tool state** — a four-value classification of a tool row for the (agent,
  conversation) pair: `granted`, `deferredHidden`, `deferredRevealed`, `neutral`.
- **Revealed set** — the set of deferred tool ids a prior `find_tools` result in the
  active conversation has exposed; identical to the run's per-step visible decision.
- **Agent allowlist / deferred list** — the selected agent's `tools` and
  `deferredTools`, as returned by `/api/assistant/agent`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For any agent and conversation, every tool row in the Tools tab matches
  the state the run's gate would compute for the same (agent, revealed) pair — 0
  mismatches, verified by a test that compares panel classification against
  `deriveRevealedIds` + the agent's allowlist/deferred lists.
- **SC-002**: Switching the selected agent updates 100% of the colored rows to the
  new agent's states; no row retains a previous agent's color.
- **SC-003**: A deferred tool transitions orange → blue the moment the active
  conversation gains a `find_tools` result that reveals it, and reverts to orange in a
  conversation without it.
- **SC-004**: A non-granted tool and every discovery tool render neutral (the
  existing default) — 0 instances of them appearing green/blue/orange.
- **SC-005**: A new user can correctly state what each of the four row colors means
  from the in-panel legend alone, without prior knowledge.

## Assumptions

- The color coding is informational only; it does **not** change which tools the agent
  can call, nor how deferral/revealing works. It restates existing run behavior in the
  panel.
- "Granted" for a registry tool means membership in the selected agent's `tools`
  allowlist, mirroring the run gate's `allow` set (empty allowlist ⇒ nothing granted).
- "Discovered" is synonymous with the run's "revealed" state and is
  conversation-scoped, matching `deriveRevealedIds`.
- The green/blue/orange palette values are chosen to be legible on the existing
  `#0f1117` dark theme; exact hex values are a design detail settled at `design`
  (a mockup will fix them), not a behavioral requirement.
- Surface tools and elicitation tools are out of scope for color coding (they have no
  registry entry and do not render as distinct rows in this tab today).
