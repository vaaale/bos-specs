# Feature Specification: Fix Pane-Resize Reliability & Log Noise

**Feature Branch**: `033-fix-pane-resize`

**Created**: 2026-08-22

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Resizing the Build Studio window while previewing a UI mockup produces 'ResizeObserver loop completed with undelivered notifications' errors, the resize handle loses the pointer mid-drag, and after release the pane sticks to the cursor until clicked again. The chat pane also cannot be widened past an over-aggressive hard cap. Fix at the BOS system level so other apps benefit, update specs/docs so the issues are not re-introduced."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Reliable pane-resize dragging (Priority: P1)

A user drags a vertical pane separator (the shared `ResizeHandle` component) in Build Studio or any other app that uses it. Even when an embedded iframe (e.g. an HTML mockup preview) sits under the path the pointer crosses, the drag tracks the pointer smoothly and ends the instant the user releases — the pane never "sticks" to the cursor afterward, no stray pointer movement re-triggers resizing, and no drag session ever outlives the user's intent.

**Why this priority**: This is the core defect. The drag is the primary interaction with the resizable layout, and its failure (drop, stick, ghost-resume) makes the layout feel broken and untrustworthy.

**Independent Test**: With an HTML mockup open in the Build Studio center pane, drag the left separator across the iframe boundary and release. Then hover the pointer back over the separator. Neither drag should restart on its own. Repeat with rapid, jagged pointer motion.

**Acceptance Scenarios**:

1. **Given** a resizable pane with an iframe somewhere in the app window, **When** the user presses the separator and drags the pointer across the iframe region, **Then** the pane continues to track the pointer for the whole drag (the drag session is not interrupted by the pointer crossing the iframe).
2. **Given** an active drag on a pane separator, **When** the user releases the pointer (or the system cancels the pointer gesture, e.g. touch/pen loss or OS preemption), **Then** the drag session ends exactly once, all drag-state and window-level listeners are cleaned up, and the pane width is fixed at its final value.
3. **Given** a drag session that has ended (released or cancelled), **When** the user later moves the pointer — including back over the separator — **Then** no resizing occurs until the user explicitly presses the separator again.
4. **Given** an active drag, **When** a second pointer (e.g. a touch) contacts the surface, **Then** the original drag session is unaffected by that second pointer and only the initiating pointer can end or drive it.

---

### User Story 2 - Benign resize-loop errors no longer flood the log (Priority: P2)

A user resizes any app pane (with or without an iframe mounted) while the central browser logger is active. The known-benign browser notification "ResizeObserver loop completed with undelivered notifications" (and its sibling "…limit exceeded") never appears in the log at error level, while every other uncaught window error continues to be captured exactly as before.

**Why this priority**: The errors are pure log noise — they mask real errors in the central log and were the user's original entry point into this bug. Removing the noise system-wide is a prerequisite for trusting the log.

**Independent Test**: Open Build Studio with an HTML mockup, drag a separator several times while it crosses the iframe, then inspect the log for `window.onerror` records during the drag window.

**Acceptance Scenarios**:

1. **Given** the central browser logger is running in any app window, **When** the browser raises "ResizeObserver loop completed with undelivered notifications" or "ResizeObserver loop limit exceeded" as an uncaught window error, **Then** no log record is written for it.
2. **Given** the central browser logger is running, **When** any other uncaught window error or unhandled rejection occurs, **Then** it is logged at error level with its usual component, message, and source data, exactly as before this change.
3. **Given** an app that genuinely produces a runaway resize loop, **When** its loop error message differs from the two known-benign browser notifications, **Then** it is still logged (the filter does not hide novel or app-specific loop errors).

---

### User Story 3 - The chat pane can actually be widened (Priority: P2)

A user in Build Studio wants a wide chat pane on a large display. The right (chat) separator can now be dragged far past the previous 820px hard cap, up to a maximum that adapts to the current window size, while the center viewer is never allowed to collapse below a sensible minimum width.

**Why this priority**: The hard cap was an overly aggressive guard against collapsing the center viewer. The user explicitly wants it relaxed, but only in a way that keeps the center viewer usable — so the cap becomes viewport-aware rather than a bigger constant.

**Independent Test**: On a wide window, drag the right separator rightward until it stops; verify the stop point is far beyond 820px and that the center viewer retains a usable minimum width at the stop. Then shrink the OS window and repeat — the stop point adapts.

**Acceptance Scenarios**:

1. **Given** a wide window (e.g. ≥ 1440px content width), **When** the user drags the chat (right) separator to its maximum, **Then** the chat pane can reach a width of at least 1100px, far beyond the previous 820px cap.
2. **Given** any window width, **When** the user drags either separator to its extreme, **Then** the center viewer's width never falls below a minimum floor, regardless of the other panes' widths.
3. **Given** a persisted chat-pane width that is now larger than the current maximum (e.g. the window shrank since it was saved), **When** the app loads, **Then** the persisted width is clamped into the current valid range instead of overflowing the layout.

---

### User Story 4 - The resize-loop trigger is defused at its source (Priority: P3)

While any app's shared V2 chat message list is resized, its stick-to-bottom behavior keeps the latest message pinned without performing a synchronous read-then-write of layout inside its ResizeObserver callback — the pattern that makes browsers raise the benign loop notification in the first place.

**Why this priority**: Story 2 hides the noise; this story removes the trigger so the benign error simply stops happening, which also makes drag-time reflow lighter. It touches a shared chat component used by several apps, so it is deliberately separable from P1–P2.

**Independent Test**: Resize the chat pane repeatedly with a long conversation open; verify the list stays pinned to the bottom and no resize-loop notification is raised.

**Acceptance Scenarios**:

1. **Given** a V2 chat message list that is scrolled to the bottom, **When** its container's size changes (pane resize, message growth), **Then** the list remains pinned to the bottom and the pinning adjustment does not perform a synchronous layout write inside the resize-observer callback.
2. **Given** a V2 chat message list that the user has scrolled up (not pinned), **When** its size changes, **Then** its scroll position is left undisturbed, as before.

---

### Edge Cases

- The pointer crosses an iframe (the Build Studio HTML mockup preview) mid-drag — the drag must not lose the pointer (Story 1, AC-1).
- The user releases the pointer while it is over an iframe or outside the app window — the drag must still end cleanly (Story 1, AC-2).
- The OS or browser fires a pointer-cancel instead of pointer-up (touch loss, gesture preemption) — cleanup must be identical to a normal release (Story 1, AC-2).
- A second pointer contacts the surface during an active drag — it must not corrupt or end the session (Story 1, AC-4).
- The window is smaller than the sum of the other panes' minimums plus the center floor — separator maxima must adapt (clamp) rather than allow overflow (Story 3, AC-2/AC-3).
- A persisted pane width is stale (saved when the window was larger) — it must be clamped on load (Story 3, AC-3).
- An app deliberately raises an error whose message merely contains the words "ResizeObserver" but is not one of the two known-benign notifications — it must still be logged (Story 2, AC-2/AC-3).

## Requirements *(mandatory)*

### Functional Requirements

**Reliable pane-resize drag (shared system-level component):**

- **FR-001**: The shared pane-resize handle MUST capture the initiating pointer for the duration of a drag, so that pointer events are delivered to the drag session even while the pointer is over embedded content (iframes) elsewhere in the window.
- **FR-002**: A drag session MUST end on both a pointer-release event and a pointer-cancel event, with identical cleanup (drag state cleared, window-level listeners removed, body text-selection suppression restored).
- **FR-003**: A drag session MUST ignore pointer events from any pointer other than the one that initiated it.
- **FR-004**: A drag session MUST be idempotent with respect to its end — after it ends, no residual state may cause a later pointer movement (including movement back over the handle) to resume resizing until a new press starts a fresh session.
- **FR-005**: During a continuous drag, the applied pane width MUST track the pointer within at most one display frame (width updates coalesced to no more than one per animation frame).

**Chat-pane maximum width (Build Studio):**

- **FR-006**: The Build Studio chat (right) pane's maximum draggable width MUST be computed from the current window content width instead of a fixed 820px constant, and MUST permit a width of at least 1100px whenever the window is wide enough.
- **FR-007**: For any window width, the sum of the left pane's minimum width, the center viewer's minimum floor width, and the chat pane's maximum width MUST NOT exceed the window content width, so the center viewer can never be reduced below its floor.
- **FR-008**: Persisted pane widths that fall outside the current valid range (after a window-size change) MUST be clamped into range when restored.

**Central browser logger:**

- **FR-009**: The central browser logger MUST NOT record the window errors "ResizeObserver loop completed with undelivered notifications" or "ResizeObserver loop limit exceeded" at error level.
- **FR-010**: The filter in FR-009 MUST match only those two exact browser notification messages; all other uncaught errors and unhandled rejections MUST be logged unchanged at error level.

**Shared V2 chat message list:**

- **FR-011**: The V2 chat message list's stick-to-bottom adjustment triggered by its ResizeObserver callback MUST NOT perform a synchronous layout write inside the callback; the adjustment MUST be deferred to the next frame or skipped when it would not change the scroll position.
- **FR-012**: The stick-to-bottom behavior MUST remain functionally equivalent to today's: pinned lists follow new content and resizes; scrolled-up lists are never forced to the bottom.

**Regression protection:**

- **FR-013**: The shared handle component's in-code documentation MUST state the drag-session contract (capture, end-on-release-and-cancel, pointer-id scoping, frame-coalesced updates) so future edits to the component do not silently regress it.
- **FR-014**: The Build Studio agentic spec's persisted-pane success criterion MUST be updated so it also covers the drag-session contract of Story 1 and the viewport-aware maximum of Story 3, and the central-logging spec's documentation MUST record that the two known-benign ResizeObserver notifications are filtered (not lost) at error level.
- **FR-015**: The feature MUST ship with regression tests: automated coverage of the drag-session lifecycle (start, move, end-on-release, end-on-cancel, end-exactly-once, foreign-pointer ignored) and an end-to-end test of dragging a Build Studio separator with a mockup iframe open and verifying no resize-loop log records are produced.

### Non-Functional Requirements

- **NFR-001**: The drag-reliability fix MUST live in the shared system-level components (the pane-resize handle, the browser logger, the shared V2 chat message list), not in Build Studio's own code, so that every current and future app using them inherits the fix without app-specific changes.
- **NFR-002**: The fix MUST NOT change any app's visible layout, default pane widths, or persisted-width storage keys, apart from the chat-pane maximum being widened per FR-006/FR-007.
- **NFR-003**: The filter in FR-009 MUST be exact (the two known notification strings), never a broad substring, so that genuinely runaway resize loops in future code remain visible in the log.

### Key Entities

- **Drag session**: a transient state spanning one press-to-release (or cancel) on a pane separator — initiated pointer, baseline position/width, and the window-level listeners that drive and end it.
- **Pane width**: the persisted width of a resizable pane (Build Studio left/right), clamped to a min/max range that now adapts to the window.
- **Window error record**: a central-log entry produced from an uncaught window error; the two known-benign ResizeObserver notifications are excluded from error-level recording.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In 10/10 consecutive drag sessions across the mockup iframe, the drag tracks the pointer end-to-end and ends on release with zero residual stickiness (a subsequent hover over the separator does not resize it).
- **SC-002**: Zero "ResizeObserver loop …" records appear in the central log during a full drag sweep (left and right separators, iframe open), while a deliberately injected synthetic window error still appears at error level in the same session.
- **SC-003**: On a ≥1440px window, the chat pane can be widened to at least 1100px; at that width the center viewer still measures at least its documented minimum floor.
- **SC-004**: After shrinking the window and reopening Build Studio, every restored pane width is within its current valid range (no layout overflow).
- **SC-005**: All apps using the shared handle or the V2 chat message list show no behavioral regression (existing resize widths, persistence, and stick-to-bottom behavior unchanged apart from the intended changes).

## Assumptions

- The pointer-capture drag pattern already used by the desktop window's own move/resize logic (capture + end-on-release-and-cancel + pointer-id scoping + frame-coalesced application) is the correct reference for the shared handle, and generalizes safely to pane separators.
- The two known-benign ResizeObserver notifications are exact, stable browser strings (Chrome/Chromium wording) and safe to match verbatim at the logger's window-error capture point.
- A center-viewer minimum floor of roughly one-third of a typical working pane (a few hundred px) is the right protection level for FR-007's floor value; the exact constant is a design/plan decision, not a user-facing one.
- Build Studio is the only current consumer of the shared handle component, so behavioral changes to it do not ripple into other apps (the V2 chat message list, by contrast, is shared and is covered by its own story + regression test).
