# Feature Specification: Self-Heal "Pin & open preview" performs the Supervisor work

**Feature Branch**: `bos/032-self-heal-pin`

**Created**: 2026-09-29

**Status**: Approved (Gate 1 — approved via GitLab issue #52, project #41, note by @alex)

**App Target**: builtin-app

**Input**: GitLab issue #52 ("Bug: Self-Heal") — "The 'Pin & open preview' button does not do what the user would expect. Currently when clicked, all that happens is that it opens the Settings app. What I would expect to have happened is that it should trigger building the candidate through the supervisor so that the user then can press the 'Preview' button in the top toolbar to actually Preview the fix."

## User Scenarios & Testing *(mandatory)*

### User Story 1 — The button pins and opens the preview (Priority: P1)

A user who has reached a `preview-ready` class-`e` self-heal case opens its card in the Build Studio self-heal pane and clicks **Pin & open preview**. Instead of a Settings window opening, the click performs the Supervisor work the label promises: the candidate is ensured to be built (`activate`, idempotent — a no-op when already `ready`) and the window is pinned to the preview, so the top toolbar flips to `PREVIEW` on the case's branch and the fix is live in this window.

**Why this priority**: This is the entire reported defect. The control does not do what its label says; no other value in this card depends on it.

**Independent Test**: Get a class-`e` self-heal case to `preview-ready` (or seed one), open its card, click **Pin & open preview**, and observe the top toolbar change from `BASE` to `PREVIEW` on the case's `bos/self-heal-<id>` branch with no Settings window opening as the result.

**Acceptance Scenarios**:

1. **Given** a `preview-ready` self-heal case whose candidate is already `ready`, **When** the user clicks **Pin & open preview**, **Then** the preview is pinned and the window reloads, and the top toolbar shows `PREVIEW` on the case's branch — no Settings app opens.
2. **Given** a `preview-ready` self-heal case, **When** the user clicks **Pin & open preview**, **Then** the button shows a transient in-progress state ("Opening preview…") and is disabled while the Supervisor calls are in flight, then returns to its normal label.
3. **Given** the self-heal card, **When** the user reads the footnote beneath the button, **Then** it still states that Promote and Discard live in the top toolbar and that self-healing never promotes a fix (FR-024 of 031-self-healing) — this footnote is preserved by the fix.

---

### User Story 2 — A not-built / stopped preview is built through the Supervisor (Priority: P2)

After a Supervisor restart, a `preview-ready` case's preview can be restored as `not-built` or `stopped`. Clicking **Pin & open preview** on such a case triggers the build through the Supervisor (`activate`), which kicks off the candidate build and reports `building`. The handler does not attempt to pin a building preview (the Supervisor refuses), and the top toolbar then shows the build in progress; once it is `ready`, the normal preview/pin flow becomes available.

**Why this priority**: This is the "trigger building the candidate" half of the report's expectation, and it makes the button robust to the restored-preview state rather than only the already-ready state.

**Independent Test**: With a `preview-ready` case whose preview is `not-built`/`stopped` (fresh Supervisor restart), click **Pin & open preview** and observe the top toolbar show the build starting (`building…`), with the button not falsely reporting success until the build completes.

**Acceptance Scenarios**:

1. **Given** a `preview-ready` self-heal case whose preview is `not-built` or `stopped`, **When** the user clicks **Pin & open preview**, **Then** `activate` is called and the candidate build starts through the Supervisor; the handler does not pin (pinning a building preview is refused) and does not claim the preview is live.
2. **Given** the same case after the build completes and the preview is `ready`, **When** the user clicks **Pin & open preview** again, **Then** the preview is pinned and the window shows the fix (User Story 1, scenario 1).

---

### Edge Cases

- **No `activeFeatureBranch`.** The button is only rendered when a `preview-ready` case also has an `activeFeatureBranch`, so the handler must guard against a missing branch and do nothing (defensive; the render condition already prevents this).
- **`activate` returns not-`ready` (e.g. `building` or an error).** The handler must NOT proceed to `pin` and must not reload; the user is left at the in-progress/building state, consistent with the top toolbar's own display.
- **`activate`/`pin` reports a failure (`ok: false`).** The handler must not reload and must restore the button to its normal state so the user can retry; no error is silently swallowed.
- **Reverse-proxy timeout on the Supervisor call.** `supervisorPost` already returns a structured `ok:false` with an explanatory message in that case; the handler treats it as a non-success (no pin, no reload).
- **Reload scope.** Only `pin` (and `promote`) replace the build this window is served by, per `replacesServedBuild` in `src/lib/supervisor/client`. The handler reloads only after a successful `pin`, matching the existing top-toolbar behaviour — it never reloads for `activate` alone.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: When a `preview-ready` self-heal case card is shown, clicking **Pin & open preview** MUST ensure the candidate is built through the Supervisor by issuing an idempotent `activate` for the case's `activeFeatureBranch`, using the shared `supervisorPost` client (`src/lib/supervisor/client`), and MUST NOT open the Settings app.
- **FR-002**: After a successful `activate` that reports the preview is `ready`, clicking **Pin & open preview** MUST pin the preview for the case's branch (`pin` with `version: "preview"`) and reload the window so the top toolbar reflects `PREVIEW` on that branch.
- **FR-003**: The handler MUST NOT pin or reload when `activate` does not report the preview as `ready` (i.e. it is `building`, or the call failed); in that case the user is left in the in-progress state the top toolbar already renders.
- **FR-004**: While the Supervisor calls are in flight, the button MUST show a transient in-progress state and be disabled; it MUST return to its normal label when the call completes (success or failure).
- **FR-005**: The click MUST NOT add a second Promote or Discard path to the self-heal card. Promotion and discard remain exclusively in the top toolbar, per FR-024 of 031-self-healing; the card's existing footnote stating this MUST be preserved.
- **FR-006**: The handler MUST reuse the same Supervisor control surface (`supervisorPost`) that the existing top-toolbar (`VersionControls.tsx`) and Versions-tab (`VersionsTab.tsx`) controls already use; it MUST NOT introduce a new Supervisor endpoint or a second client implementation.

### Key Entities

- **`HealingCase`**: the self-heal case record rendered by `PreviewStatus`. Relevant attributes: `status` (drives the `preview-ready` render condition), `activeFeatureBranch` (the `bos/self-heal-<id>` branch the fix is built on), `scopeClass`. No new fields are added by this fix.
- **Supervisor preview**: the build/pin target identified by the case's branch; its state (`ready`, `building`, `not-built`, `stopped`) is the precondition the handler branches on.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Clicking **Pin & open preview** on a `preview-ready` case with an already-`ready` preview results in the top toolbar showing `PREVIEW` on the case's branch, with **zero** Settings windows opened, in the single click (no follow-up navigation required).
- **SC-002**: The same click on a `not-built`/`stopped` preview starts the candidate build through the Supervisor (toolbar shows the build in progress) rather than navigating away, and the preview becomes available to pin once it is `ready`.
- **SC-003**: The old failure mode — clicking the button opens Settings → "Repositories" (the `self-modification` tab) with no version controls — is no longer reachable from this button.
- **SC-004**: No new Supervisor endpoint, client, or Promote/Discard control is introduced; the change is confined to the self-heal card's button handler and its imports.

## Assumptions

- The intended target behaviour is that "the window ends up serving the candidate" — i.e. the fix is **pinned** as the preview for this window, which is the superset of the report's ask ("trigger the build so I can press Preview"). This matches the top toolbar's own pin semantics and `replacesServedBuild`'s reload rule. A narrower variant (build only, leave the user on base) is a one-line drop of the `pin` step and was offered as an alternative in the Gate-1 comment; the approved scope is the pin variant.
- By the time the button is visible (status `preview-ready`), `completeFix` in `src/lib/self-heal/intake.ts` has already verified the preview was `ready` during the pipeline run, so `activate` is normally a no-op; the `activate`-before-`pin` is retained for the robust case of a restored `not-built`/`stopped` preview after a Supervisor restart.
- A `preview-ready` case always carries an `activeFeatureBranch` (the render condition for the button); the handler's branch guard is defensive, not a normal-path requirement.
- The change is to a single built-in-app file — `src/apps/build-studio/selfheal/PreviewStatus.tsx` — its imports, and (optionally, per the implementation) a test. It does not touch the Supervisor process (`src/tools/supervisor/lib/`), the top toolbar, or the Versions tab.
- Testing follows the repo's Playwright unit runner (`npm run test:unit`); any new test lives at `tests/<area>/<name>.test.ts`, not a co-located `src/**/__tests__/` (see `user-specs/discrepancies.md` / the recorded Vitest-absence pattern).

## Out of Scope

- Adding Promote or Discard controls to the self-heal card (remains top-toolbar-only per FR-024 of 031-self-healing).
- Changing Supervisor `activate`/`pin` semantics, their endpoints, or `supervisorPost`.
- Modifying the class-`d-bis` app-rebuild branch of this card (it does not render this button).
- Any second promote path or an automatic retry of a failed build.
