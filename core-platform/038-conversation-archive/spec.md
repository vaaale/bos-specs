# Feature Specification: Conversation Archive

**Feature Branch**: `038-conversation-archive`

**Created**: 2026-09-11

**Status**: Implemented (pending promote)

**App Target**: bos-core

**Input**: User description: "The list of conversations in both the Assistant app and in Build Studio are getting very long. I would like to have a way to archive conversations so they don't show up in the said apps, but are not deleted."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Archive a conversation to declutter the list (Priority: P1)

A user has a long list of past conversations in the Assistant app (or Build Studio) and wants to tidy it up. They find a conversation they no longer need at hand, archive it, and it immediately disappears from the list — but it is still there, intact, if they ever want it back. Archiving is a lightweight "out of sight" action, not a destructive one.

**Why this priority**: This is the core value of the feature. Without the ability to archive and have it vanish from the list, nothing else works. It is the minimal slice that already solves the stated problem.

**Independent Test**: Open the Assistant app (or Build Studio), select any existing conversation, trigger Archive, and verify (a) it no longer appears in the default conversation list, and (b) it still exists and can be reopened directly. No other user story is required to observe this value.

**Acceptance Scenarios**:

1. **Given** a conversation is visible in the Assistant app list, **When** the user archives it, **Then** it no longer appears in that list, and its content is fully preserved.
2. **Given** a conversation is visible in the Build Studio list, **When** the user archives it, **Then** it no longer appears in that list, and its content is fully preserved.
3. **Given** a conversation has just been archived, **When** the user reopens it directly (e.g. from a link, deep reference, or the archived view), **Then** it opens and can be viewed normally, with all prior messages intact.
4. **Given** a conversation is archived, **When** the user attempts to send a new message to it, **Then** the system requires the conversation to be unarchived first — an archived conversation is read-only until restored, and a message is only accepted after it is unarchived.

---

### User Story 2 - Review and restore archived conversations (Priority: P2)

A user has archived several conversations and later realizes they need one back. They can browse their archived conversations, find the one they want, and unarchive it, returning it to the normal list. Archiving is fully reversible.

**Why this priority**: Reversibility is what makes archiving safe rather than destructive — the user's explicit requirement that conversations "are not deleted." It is the second-highest value after the archive action itself, but the feature delivers its core value with P1 alone.

**Independent Test**: Archive a conversation, open the place where archived conversations are viewable, unarchive it, and verify it reappears in the normal list with all content intact.

**Acceptance Scenarios**:

1. **Given** at least one conversation is archived, **When** the user opens the view of archived conversations, **Then** all and only archived conversations are listed there.
2. **Given** a conversation is shown in the archived view, **When** the user unarchives it, **Then** it reappears in the normal (default) conversation list and disappears from the archived view.
3. **Given** a conversation is archived, **When** the user inspects it from the archived view, **Then** its content is identical to what it was before archiving (nothing truncated or lost).

---

### User Story 3 - Archive from anywhere a conversation is surfaced (Priority: P3)

Whichever app surfaces a conversation list (Assistant today, Build Studio today, and any future app that lists conversations), the archive action is available there and behaves consistently. Users do not need to learn a per-app mechanism.

**Why this priority**: Consistency across surfaces is a natural extension once the mechanism exists, and it guards against per-app duplication. It adds polish and future-proofing but is not required for the core problem to be solved.

**Independent Test**: Archive the same conversation from the Assistant app and from Build Studio and confirm both lists reflect the change identically.

**Acceptance Scenarios**:

1. **Given** the same conversation appears in two app lists, **When** the user archives it in one, **Then** it is removed from both lists.
2. **Given** an app lists conversations, **When** the user archives one there, **Then** the same affordance (archive/unarchive) is available and the underlying state is shared across apps.

---

### User Story 4 - Auto-archive when its feature is promoted (Priority: P3)

A user drives a feature through Build Studio on a feature branch. When they promote that feature to base, the conversation that carried the work is done — it should tidy itself into the Archived section so it stops cluttering the list, without them having to archive it manually. It is still there, restorable, if they come back to it.

**Why this priority**: This extends the decluttering value to the natural "I'm done with this" moment (a promote). It is a convenience automation on top of the core archive mechanism and is not required for the primary problem to be solved.

**Independent Test**: Drive a feature branch to a promoted state and verify the conversation(s) associated with that branch are automatically moved to the Archived section, still restorable and content-intact.

**Acceptance Scenarios**:

1. **Given** a conversation whose active feature branch is `bos/foo`, **When** `bos/foo` is promoted, **Then** that conversation is archived (hidden from the default list, present in the Archived section) without being deleted.
2. **Given** the promoted conversation was the active conversation in Build Studio, **When** it is auto-archived, **Then** it no longer appears in the Build Studio dropdown and its content is intact in the Archived section.
3. **Given** a conversation is auto-archived by a promote, **When** the user unarchives it from the Assistant app, **Then** it behaves exactly like any manually archived conversation.

---

### Edge Cases

- What happens when the currently-open conversation is archived from within its own view — does it stay open, and what does the list do next?
- How is a user who has navigated directly to a specific archived conversation's URL treated (still openable, or redirected)?
- What happens when an archived conversation is later deleted through the normal delete flow — does it disappear from the archived view too?
- Can a conversation be archived and unarchived rapidly/repeatedly without corrupting its state?
- If a conversation is the most recent/active one, does archiving it change which conversation is "current"?
- What happens to in-flight or pinned state on an archived conversation?
- What happens when multiple conversations share the same promoted branch — are they all archived?
- What happens if the auto-archive on promote fires but the promote's post-settle notification is lost (e.g. the page reloads) — is the conversation still eventually archived, or is this best-effort?
- If the currently-open conversation is auto-archived by a promote, does it stay open (read-only) or is the user redirected?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST support marking an existing conversation as "archived" and later unarchiving it, as a persistent, reversible state change that does not delete the conversation or its content.
- **FR-002**: Archived conversations MUST be excluded from the default conversation lists shown by the Assistant app and Build Studio.
- **FR-003**: The system MUST provide a view or mechanism through which a user can see their archived conversations and unarchive them.
- **FR-004**: Archiving a conversation MUST preserve all of its content (messages, metadata, and any associated context) exactly as it was before archiving.
- **FR-005**: The archive state MUST be shared across apps — archiving in one app MUST cause the conversation to be hidden in all apps that list conversations.
- **FR-006**: The system MUST expose an archive affordance in the conversation-list surface of the Assistant app.
- **FR-007**: The system MUST expose an archive affordance in the conversation-list surface of Build Studio.
- **FR-008**: The system MUST present a collapsible "Archived" section in the Assistant app's conversation panel that enumerates the user's archived conversations and is **collapsed by default** (so the tidy-up intent is the resting state). In Build Studio, archived conversations MUST NOT appear in the conversation dropdown at all — they are filtered out entirely. A Build Studio conversation is restored from the Assistant app's Archived section (the two surfaces share one archive state).
- **FR-009**: An archived conversation MUST be read-only: it MAY be opened and viewed, but sending a new message to it MUST first unarchive it.
- **FR-010**: When a feature branch is promoted, every conversation whose active feature branch is that branch MUST be archived automatically (not deleted, and still restorable from the Archived section).

### Key Entities *(include if feature involves data)*

- **Conversation**: An existing unit of work surfaced in app lists. Gains a persisted **archived** attribute (boolean, default false) that governs list visibility without affecting content or existence.
- **Archived view**: A place (list/section) that enumerates only conversations whose archived attribute is true.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: After archiving a conversation, the user's default list length in the originating app decreases by one, and the conversation is not present there.
- **SC-002**: 100% of a conversation's content is recoverable after an archive → unarchive cycle with no loss or alteration.
- **SC-003**: A user can archive a conversation and later restore it in under 10 seconds without any data loss.
- **SC-004**: Archiving a conversation in one app reflects in the other app's list with no manual refresh or re-login required.
- **SC-005**: No conversation is ever deleted as a result of an archive or unarchive action.
- **SC-006**: After promoting a feature branch, 100% of conversations associated with that branch are absent from the default list and present in the Archived section, with content intact.

## Assumptions

- Archiving is a property of the conversation itself, not of a particular app's list — hence one shared state, not two per-app lists.
- An archived conversation is read-only: it can be opened and viewed, but resuming it (sending a new message) requires unarchiving it first.
- "Not deleted" means content is fully retained and restorable; archiving is a visibility filter, not a purge.
- The existing conversation list surfaces in the Assistant app and Build Studio are the only surfaces required to honor the filter in v1; other surfaces that list conversations will honor it for free if they read from the same state.
- No retention period or auto-cleanup applies to archived conversations in v1 (they persist indefinitely until explicitly deleted or unarchived).
- Search behavior across archived vs. non-archived conversations is out of scope for v1 unless the existing search already reads the same list state, in which case it inherits the filter automatically.
