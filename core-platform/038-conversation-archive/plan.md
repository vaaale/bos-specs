# Implementation Plan: Conversation Archive

**Branch**: `038-conversation-archive` | **Date**: 2026-09-11 | **Spec**: `spec.md` (this directory)

**Input**: Feature specification from `/Specs/user-specs/core-platform/038-conversation-archive/spec.md`

**Design**: `design.md` (this directory) — classification, context/container/component model, file/module plan, ADRs, and risks live there; this plan references rather than duplicates them.

## Summary

Add a persistent, reversible `archived` flag to the conversation data model so users can move long-standing conversations out of the way without deleting them. In the **Assistant app** (left `ConversationPanel`) the flag hides archived conversations from the default list and a **collapsible Archived section (collapsed by default)** lets the user restore them. In **Build Studio** (toolbar `ConversationSelector`), archived conversations are **filtered out of the dropdown entirely** and an archive button acts on the active conversation. Archived conversations are read-only until unarchived (enforced authoritatively on the server). When a feature branch is **promoted**, the conversation(s) carrying that branch are **auto-archived**. Because both apps are built-in windows sharing one `conversations.ts` store in a single JS context, cross-app propagation is free by construction. Full mechanism, ADRs, and risk analysis are in `design.md` — this plan is the task-level breakdown built on top of that design.

## Technical Context

**Language/Version**: TypeScript (BOS Next.js 15 App Router, existing toolchain)

**Primary Dependencies**: None new — reuses `lucide-react` icons, the existing `conversations.ts` store, `conversation-store.ts` metadata writer, and the `enqueuePerKey` write queue.

**Storage**: One JSON file per conversation at `/Documents/Chats/<id>.json` (user VFS). Additive `archived?: boolean` metadata field; **missing = `false`** → no migration.

**Testing**: Existing repo test setup (Vitest for unit, Playwright e2e). Authoritative read-only gate is a server route → e2e-verifiable via the send path.

**Target Platform**: BOS desktop shell (web-rendered, single JS context per instance)

**Project Type**: bos-core (modify existing data model + shared components + one API route; no new app/service/window ships)

**Performance Goals**: List filtering is a single boolean predicate over an in-memory store — no measurable cost.

**Constraints**:
- No new external dependency, no `package.json`/lockfile/build-config change (Constitution VII).
- The archive write MUST go through `patchConversationMeta` (queue-serialized) — a plain VFS write would re-introduce the documented `activeFeatureBranch` race (ADR-3).
- The read-only gate MUST be server-authoritative (Constitution II); the client banner is UX only.

**Scale/Scope**: ~5 modified files + 1 new route + 2 promote-surface hook sites (see design.md §4). Two app surfaces + a post-promote auto-archive seam, all on one shared store.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after design.*

Verified in `design.md` §2 and re-confirmed by the independent review (verdict: **Ready for plan**, no must-fix findings):

| Principle | Status |
|---|---|
| II. Server Authority & SSR Boundary | ✅ FR-009 gate on `POST /api/assistant/runs` (server); client banner is UX-only |
| IV. Minimize Blast Radius | ✅ Feature-branch edits; additive, backward-compatible flag; no new global state |
| V. The VFS Is Not the Source | ✅ Flag lives in `/Documents/Chats/<id>.json` (user data); source changes in `src/` on branch |
| VI. Specs & Docs Stay in Sync | ⚠️ `implement` must update `docs/dev/assistant/*` + a usage page in the same change (task) |
| VII. Respect Boundaries / Quality gates | ✅ No dependency/config change; `tsc`+lint gate unchanged |

No violations → no Complexity Tracking required.

## Project Structure

### Documentation (this feature)

```text
038-conversation-archive/
├── spec.md
├── design.md          # architect output — classification, ADRs, file plan, risks
├── mockup.html        # ui-designer output — Assistant surface, all required states
├── plan.md            # This file
└── tasks.md           # tasks step output (next)
```

### Source Code (repository root)

Concrete layout per `design.md` §4 — the only files this feature creates or modifies:

```text
src/
├── lib/
│   └── agent/
│       └── conversations.ts                          # MODIFY: archived flag, setConversationArchived, carry flag in saveConversationMessages
├── app/api/assistant/
│   ├── conversations/[conversationId]/archive/
│   │   └── route.ts                                  # NEW: PATCH → patchConversationMeta (queue-serialized)
│   └── runs/
│       └── route.ts                                  # MODIFY: FR-009 409 gate (+ import getConversationMeta)
└── components/
    ├── apps/assistant/
    │   ├── ConversationPanel.tsx                     # MODIFY: ConvRow kebab menu; collapsible per-agent Archived sub-sections (default collapsed)
    │   └── AgentSelector.tsx                         # MODIFY: ConversationSelector filters out archived + active-row archive button
    └── agent/v2/
        └── ChatInputV2.tsx                           # MODIFY: read-only banner / locked composer

# MODIFY (auto-archive on promote — design.md §4 items 7-8, ADR-7):
#   src/lib/agent/conversations.ts                    # + archiveConversationsForBranch(branch)
#   VersionControls (desktop topbar) + Settings→Versions  # call it at the existing notifySelfHealBranchSettled(branch,"promoted") site

# NOT modified (verified generic — design.md §4): src/lib/assistant/conversation-store.ts
# Docs to update in the same change (Constitution VI): docs/dev/assistant/* + docs/usage archive page
```

**Structure Decision**: This is a contained `bos-core` change — no new project, window, or service. The additive flag on the existing store plus per-surface split (design.md ADR-4) is the minimal structure that satisfies FR-002 (hide from default list), FR-008 (always-visible Archived section), and FR-009 (read-only until unarchived) simultaneously.

## Design Notes (folded from design.md)

- **Shared store, not shared component** (design.md §3.1): Assistant = left `ConversationPanel` (all-agents); Build Studio = toolbar `ConversationSelector` `<select>` (single agent). Both read the same `conversations.ts` `useSyncExternalStore` singleton → each surface filters/splits locally, and they differ by design: **Assistant shows a collapsible Archived section (default collapsed); Build Studio filters archived out of the dropdown entirely**.
- **Cross-app propagation is free** (§3.2): built-in non-iframe windows share one JS context; `setState→notify()` re-renders the other app (FR-005/SC-004).
- **Read-only gate is server-authoritative** (ADR-2) at the single send funnel `POST /api/assistant/runs`.
- **Persistence via queue** (ADR-3) avoids the documented `activeFeatureBranch` write-race.
- **Layout decision (user)**: all-agents panel uses **per-agent archived sub-sections**, not a pooled section (design.md R3, resolved).
- **Affordances** (ADR-5, amended): kebab row menu (Archive/Unarchive) in the Assistant panel; in Build Studio the dropdown lists only non-archived conversations + a single **Archive button** on the active conversation (restore happens in-place via the shared read-only banner or from the Assistant panel).
- **Self-archive** (ADR-6): the open conversation stays open and goes read-only; archiving does not change which conversation is "current."
- **Auto-archive on promote** (ADR-7): new `archiveConversationsForBranch(branch)` over the store's `activeFeatureBranch` link, called at the existing post-promote site (both promote surfaces, alongside `notifySelfHealBranchSettled(branch, "promoted")`). No new event/route; best-effort, `keepalive`-surviving the reload; fires on promote only.
- **R1 note (reviewer)**: the client-side `saveConversationMessages` *would* drop `archived`, but currently has **zero live callers** — the implement task is to *confirm* that dead-code status (and carry the flag defensively anyway), not to hunt for a caller.
