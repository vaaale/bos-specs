# Feature Specification: Browser Automation

**Feature Branch**: `004-browser-automation`

**Created**: 2026-06-28 (migrated from `spec/automation/browser-automation.md`)

**Revised**: 2026-09-22 (v2 — stateful sessions + first-class `browser_*` tools; branch `bos/browser-automation-v2`)

**Revised**: 2026-10-10 (storage state — a captured session loaded into the isolated profile; branch `bos/browser-storage-state`)

**Revised**: 2026-10-10 (user agent — a configurable UA, for sites that refuse the headless token; branch `bos/browser-user-agent`)

**Status**: Implemented

**Input**: "Make the assistant able to DRIVE a real, stateful browser — open a URL, click around, take screenshots — one live browser per conversation, held open across tool calls. Use cases: screenshot BOS for documentation, scrape content, perform actions by driving a browser instead of APIs."

> Distinct from `008-self-testing`: same Playwright substrate, but a different driver (assistant via managed MCP server vs dev agent via CLI) and risk profile.
>
> **v2 history.** v1 injected the managed Playwright MCP server through the legacy
> CopilotKit runtime path, which the server-owned assistant (v2 runs) never
> consults — so no current run ever saw a browser tool; and the only generic path
> (the MCP gateway) spawns a fresh stdio process per call, so no two browser
> tools ever saw the same page. v2 keeps `@playwright/mcp` as the engine and
> rebuilds the BOS side: a stateful session layer plus first-class registry
> tools.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - The assistant drives ONE stateful browser (Priority: P1)

The assistant (which has no shell) drives a real browser through first-class
`browser_*` server tools. All calls in a conversation operate on the same live
browser: what one call navigates to, the next can click. Element targeting uses
accessibility snapshots (yaml with `[ref=eN]` handles), not vision guessing.

**Acceptance Scenarios**:

1. **Given** automation is enabled, **When** the assistant calls
   `browser_navigate` and then `browser_click` with a ref from the first
   result's snapshot, **Then** the click acts on the page the navigate loaded —
   one session, not two processes.
2. **Given** a page-changing tool succeeds (navigate/click/type/…), **Then** its
   result ends with the resulting page's INLINE snapshot (refs included) — the
   model never has to chase a saved `.yml` file link.
3. **Given** a session is idle past its TTL (15 min) or the assistant calls
   `browser_close`, **Then** the browser process is closed and the next browser
   call starts fresh.
4. **Given** the browser process dies mid-session, **Then** the failing call
   reports it and drops the session; the next call reconnects instead of
   wedging forever.

### User Story 2 - Screenshots land where the user (and the docs) live (Priority: P1)

Screenshots are for using — in documentation, in files — not for dying in a
temp dir.

**Acceptance Scenarios**:

1. **Given** the assistant calls `browser_take_screenshot`, **Then** the image
   is saved under the VFS `/Screenshots` folder (visible in the Files app,
   addressable by `file_*` tools) AND returned to the model as a vision
   attachment so it can see the captured page.
2. **Given** a `filename` argument (e.g. `docs/shot.png`), **Then** the file is
   created at exactly `/Screenshots/docs/shot.png` (parents created); absolute
   or traversal filenames are refused — a filename can never escape the VFS.
3. **Given** any tool result referencing a saved file, **Then** the path shown
   to the model is the VFS path, never a host path.

### User Story 3 - Off and sandboxed until I enable and scope it (Priority: P1)

A Settings namespace governs enablement, host scope, headless, isolation,
consent, and limits; the capability is off by default.

**Acceptance Scenarios**:

1. **Given** the feature is off (default), **When** the assistant attempts a
   browser task, **Then** the tool answers with an in-band, actionable error
   naming Settings → Browser Automation — the failure is explainable, not
   silent.
2. **Given** the user enables it with an isolated profile, **When** automation
   runs, **Then** it uses a profile with no access to the user's real
   cookies/sessions.
3. **Given** the user has captured a storage state file and configured its path,
   **When** a session starts, **Then** the browser begins signed in to the
   accounts that file carries — the one deliberate, user-placed way past the
   isolated profile.
4. **Given** a configured storage state that CANNOT take effect — the file is
   absent, or the isolated profile is off (a persistent profile keeps its own
   cookies and the server ignores the flag) — **When** a session is requested,
   **Then** BOS refuses with a reason naming the fix and starts no browser; it
   MUST NOT browse signed out, because the agent would then report a
   logged-out page as the content.

5. **Given** a site that refuses automated browsers (an immediate 403 block
   page, not a solvable interstitial), **When** the user sets a User-Agent,
   **Then** the browser presents that string instead of the headless token —
   and the user has been told, in the docs, that this steps around the site's
   stated decision and to check its terms.

### User Story 4 - Page content cannot hijack the agent (Priority: P2)

Content read from automated pages is treated as untrusted data, never as
instructions.

### Edge Cases

- Host scope (allowed/blocked origins) is advisory only and does not affect
  redirects; true containment is the execution sandbox.
- If no Playwright browser is installed, the tools answer with the probe's
  install hint (`npx playwright install chromium`) — never a hard failure, and
  never a silent one.
- Two concurrent first calls for one session share a single in-flight browser
  start (no race to spawn two).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST expose browser automation as first-class `browser_*`
  server tools in the ONE tool registry (reaching main chat, sub-agents, and
  headless runs), each proxying the official Playwright MCP server
  (`@playwright/mcp`) — BOS MUST NOT reimplement the browser tools themselves.
- **FR-002**: A browser session MUST be scoped per (conversation, agent) and
  held open across tool calls: one persistent MCP client = one live browser per
  session, with in-flight connect dedup, an idle reaper (15-min TTL), explicit
  teardown (`browser_close`), and close-all on server shutdown. A
  transport-level failure MUST drop the session so the next call reconnects.
- **FR-003**: Page-changing tools MUST return the resulting page state inline:
  the proxy composes a follow-up accessibility snapshot into the tool result
  (the pinned `@playwright/mcp` answers actions with only a link to a snapshot
  file, which the model cannot read).
- **FR-004**: BOS MUST launch the browser via `--executable-path` pointing at
  the browser resolved by the shared capability probe (no extra download). The
  probe MUST recognize both Playwright install layouts (pre-CfT `chrome-linux/`
  and CfT `chrome-linux64/` etc.), prefer the newest build numerically, and —
  when the browsers dir holds no bundled build — fall back to a system Chrome
  channel install at the known OS locations (`/opt/google/chrome/chrome`, …;
  the Docker image installs via `npx playwright install chrome`, leaving
  `PLAYWRIGHT_BROWSERS_PATH` empty by design). Server arguments are structured
  command+args (no whitespace-splitting of values), with `--output-dir` pointed
  at the VFS screenshots folder and `--output-mode stdout`.
- **FR-005**: The capability MUST be OFF by default. The tools stay registered
  and gate at execute time: disabled ⇒ an in-band error naming
  Settings → Browser Automation; no browser installed ⇒ the probe's install
  hint. Never a silent no-op.
- **FR-006**: Real containment MUST be the execution sandbox — the Playwright
  browser makes its own network requests and bypasses the app-level
  `isBlockedHost()` proxy guard; host scope is advisory defense-in-depth, NOT a
  security boundary (the server default is allow-all).
- **FR-007**: An isolated browser profile MUST be the default (no access to real
  cookies/sessions/credentials); downloads MUST be disabled by default. The
  ONLY path past the isolated profile is the storage state of FR-013: a file
  the user captured and placed by hand. BOS MUST NOT read the user's real
  browser profile, and MUST NOT acquire a session itself (no credential entry,
  no login automation) — the user authenticates in their own browser and hands
  BOS the result.
- **FR-008**: The agent's tool guidance MUST state that extracted page content
  is untrusted input and must not be followed as instructions.
- **FR-009**: A configurable consent policy (`off` | `per-use` | `per-session`)
  MUST be exposed; the controls enforced today are the `enabled` gate, the
  origin filters, and the isolated profile; per-use/per-session elicitation is
  configurable but NOT yet enforced (a known follow-up, tracked in
  `discrepancies.md`).
- **FR-010**: A `browser-automation` config namespace MUST render a Settings tab
  (and auto-expose to the assistant) with at least: enabled (default off),
  allowed/blocked origins, headless (default true), isolated profile (default
  true), storage state path (default empty), user agent (default empty),
  consent policy, downloads (default false), and limits. Policy is resolved per call — changes apply to the next
  session with no restart.
- **FR-011**: Screenshots (and any file the browser saves) MUST land in the VFS
  (`/Screenshots`), MUST be reported by VFS path (host paths — absolute or
  cwd-relative — rewritten before the model sees them), and MUST return to the
  model as a vision attachment when the server provides image content (≤ 5 MB).
  A caller-supplied `filename` MUST be validated (relative, no traversal) and
  pinned inside the screenshots folder, parents created.
- **FR-012**: The `browser_*` tool ids MUST be registered as capabilities
  (discoverable via `find_tools`, groupable, allowlistable), seeded into the
  assistant's allowlist, and granted once to pre-existing installs via a
  marker-guarded additive backfill (a user's later removal sticks).
- **FR-013**: BOS MUST be able to start the browser from a user-supplied
  Playwright **storage state** (cookies + localStorage captured from a session
  the user authenticated themselves), passed to the server as
  `--storage-state`, so the assistant can read what the user's own
  subscriptions and accounts entitle them to instead of the logged-out page.
  - A RELATIVE configured path MUST resolve against the **data dir**, not the
    process cwd, so one value holds across the checkout, a per-user container
    (`/app/data`), and a DataFS clone of the data dir.
  - BOS MUST refuse to start the browser (no server config, plus a reason
    naming the fix) when the setting cannot take effect: the file is missing,
    or the isolated profile is off. Degrading to a signed-out session is
    forbidden — it yields a plausible wrong answer (a paywall preview reported
    as the article), the failure class the constitution's no-workarounds rule
    exists to prevent.
  - The storage state applies to EVERY session — main chat, sub-agents and
    headless runs alike — so the accounts it carries are reachable by anything
    holding the `browser_*` tools. This MUST be documented to the user as a
    credential, together with the requirement that the file live OUTSIDE the
    VFS (inside it, the `file_*` tools could read the session back out and the
    agent could exfiltrate it).
  - BOS MUST NOT write, refresh, or transmit the file; it is read-only input.
- **FR-014**: The browser's User-Agent MUST be configurable (`--user-agent`),
  defaulting to empty = Chromium's own. Headless Chromium advertises itself
  (`HeadlessChrome/<v>`), and a site fronted by a WAF that refuses automated
  clients answers that token with a flat 403 before the origin sees the
  request — so a valid, signed-in session still yields a block page, which the
  agent would otherwise report as the content.
  - The value MUST be passed as a DISCRETE argv entry, never through the
    command-line splitter: every real UA contains spaces, and splitting one
    leaves the browser claiming to be `Mozilla/5.0` with the remaining
    fragments read as flags.
  - Blank/whitespace MUST be indistinguishable from unset (no empty flag).
  - It is independent of the profile mode — it applies to isolated and
    persistent sessions alike.
  - This setting changes only how the browser IDENTIFIES itself. It grants no
    access: content the user has no subscription to stays unavailable, and a
    storage state (FR-013) only ever carries accounts they already hold. The
    user-facing documentation MUST say that overriding the token steps around a
    site's stated decision not to serve automated clients, and that the user
    should check that site's terms. BOS MUST NOT go further than a UA string —
    no fingerprint spoofing, no challenge solving, no proxy rotation.

### Key Entities

- **Playwright MCP server** — managed stdio server providing the browser engine;
  derived per call from config + probe (`getBrowserAutomationStatus()`).
- **Browser session** — the stateful unit: one persistent MCP client per
  (conversation, agent), registered in a hot-reload-safe global registry
  (`browser-session.ts`), idle-reaped, crash-dropped.
- **`browser_*` registry tools** — the assistant-facing surface
  (`tools/server/browser.ts`), thin proxies + result shaping (inline snapshot
  composition, VFS path rewriting, vision attachments).
- **Capability probe** — shared "is a browser available?" check that also
  resolves the Chromium path across install layouts.
- **Storage state** — a user-captured Playwright session file (cookies +
  localStorage) loaded into the isolated profile; read-only input, held outside
  the VFS, treated as a credential.
- **`browser-automation` config namespace** — the policy surface.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: With the feature off, every browser tool call returns an
  actionable refusal naming the setting; nothing reaches a browser.
- **SC-002**: Automation reuses the single installed Chromium with no additional
  download — including on current Playwright versions (CfT install layout).
- **SC-003**: An isolated profile is used by default — no leakage of the user's
  real sessions.
- **SC-004**: A navigate → click → screenshot sequence across three separate
  tool calls acts on ONE page: the click targets a ref from the navigate
  result, and the screenshot shows the post-click page.
- **SC-005**: A screenshot taken for documentation is retrievable at a
  deterministic VFS path chosen by the caller and visible in the Files app.
- **SC-006**: With a valid storage state configured, a page that requires
  sign-in renders as the signed-in user; with a storage state that cannot take
  effect, no browser starts and the reason names what to fix — there is no
  configuration in which BOS silently returns the logged-out page.
- **SC-007**: On a site that 403s the headless token, setting the User-Agent is
  the only change needed to reach the page; the configured string arrives as a
  single argv value, and leaving it empty reproduces today's behaviour exactly.

## Notes

- Complements the proxy **Web Browser** app (user-facing, guarded by
  `isBlockedHost()`): different actor, trust tier, and containment.
- Shares the Playwright substrate with `008-self-testing` (MCP here is primary;
  there it is a fallback).
- The session registry deliberately mirrors `run_command`'s sandbox-container
  registry (globalThis maps, in-flight dedup, reaper, shutdown hooks) — one
  proven pattern for "expensive per-session process kept alive between tool
  calls".
- Engine choice (kept from v1): the raw Playwright library would require BOS to
  reimplement the accessibility-snapshot/ref interaction model and would put
  Chromium inside the Next.js process; the MCP server keeps it in a disposable
  subprocess.
- The storage state is scoped to a user reaching their OWN accounts — the same
  content they can already open in their own browser, on sites whose terms
  permit it. It is a key to one session, not a bulk-collection facility, and the
  user-facing docs say so where the capture recipe lives.
- Dev docs: `docs/dev/automation/browser-automation.md`. Tests:
  `tests/assistant/browser-session.test.ts`, `tests/assistant/browser-tools.test.ts`,
  `tests/assistant/playwright-probe.test.ts`, `tests/agent/browser-tools-backfill.test.ts`,
  `tests/assistant/browser-automation-storage-state.test.ts`,
  `tests/assistant/browser-automation-user-agent.test.ts`.
- v1 prose remains in git history; original migration source was
  `spec/automation/browser-automation.md`.
