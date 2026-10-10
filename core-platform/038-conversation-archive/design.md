# Design: Conversation Archive

Spec: `038-conversation-archive` · App Target: `bos-core` · Companion mockup: `mockup.html` (same dir)

---

## 1. Classification

**`bos-core`** — agrees with `spec.md`'s `App Target`.

Reasoning (per the build-studio target taxonomy, all three read):
- The change is to BOS's own **conversation data model** (`src/lib/agent/conversations.ts`), the **core assistant components** both apps embed, and **API routes** under `src/app/api/assistant/`. That is exactly the `bos-core` scope ("Settings, desktop, API routes, server-only logic that isn't a self-contained app/service" — `target-bos-core.md`).
- It is **not** a `builtin-app`: nothing new ships as a `src/apps/<id>/` window. The two affected windows (`chat`, `build-studio`) already exist; we only modify shared code they consume.
- It is **not** a `marketplace-item`: there is no raw/non-standard protocol, no continuous background daemon, no own-port service, no self-contained installable unit. The `target-marketplace-item.md` "service facet sidesteps a Next.js limitation" guardrail is **not triggered** — the only server logic needed is one boolean gate on an existing route and one thin PATCH, both ordinary Next.js App Router work. No `data/user-apps/items/<id>/`, no `service.json`, no worker thread.

So the "check marketplace-item before concluding bos-core" pass was done and the answer stays `bos-core`.

---

## 2. Constitution check

| Principle | Compliance |
|---|---|
| II. Server Authority & SSR Boundary | ✅ The authoritative read-only gate lives in `src/app/api/assistant/runs/route.ts` (server). The client UI gate is UX sugar, not authority. |
| IV. Minimize Blast Radius | ✅ All edits are on a feature branch via `dev_delegate`; no new global state; the `archived` flag is additive and backward-compatible. |
| V. The VFS Is Not the Source | ✅ The flag is persisted on the existing conversation file in the user's VFS (`/Documents/Chats/<id>.json`) — user data, not BOS source. Source changes go through `src/` on a branch. |
| VI. Specs & Docs Stay in Sync | ⚠️ Action for `implement`: update `docs/dev/assistant/*` (conversation model) and a `docs/usage` page for the archive affordance in the same change. Recorded as a task, not a design decision. |
| VII. Respect Boundaries / Quality gates | ✅ No `package.json`/lockfile/build-config change; no new external dependency (see ADR-1). `tsc`+lint gate unchanged. |

No conflicts. The only thing flagged (VI) is the standard docs-sync obligation, not a design tension.

---

## 3. Architecture

### 3.1 How the two surfaces actually reach the conversation list (verified, not assumed)

This is the load-bearing fact, and it is **not** "they share one list component":

```
Assistant app (src/apps/chat/index.tsx)
  └─ <AssistantChatV2 allGroups showConversations showInfo />
        showLeftPanel && allGroups  →  <ConversationPanel currentAgentId onPickAgent />   (LEFT PANEL, all-agents, grouped by agent)
Build Studio (src/apps/build-studio/index.tsx)
  └─ <AssistantChatV2 agentId={buildStudioAgent} showConversations conversationsInToolbar />
        useToolbar = conversationsInToolbar && !allGroups = true  →  showLeftPanel = false
        showToolbar →  <ConversationSelector agentId={resolvedAgentId} />   (TOOLBAR <select> dropdown, single agent)
```

Both components live in the files named by the task:
- `src/components/apps/assistant/ConversationPanel.tsx` → `ConvRow` (the row) + the single-agent and all-agents views.
- `src/components/apps/assistant/AgentSelector.tsx` → `ConversationSelector` (the `<select>` Build Studio shows).

**They share the store, not the component.** Both read the same module-level `useSyncExternalStore` singleton in `src/lib/agent/conversations.ts`:
- `useConversations(agentId)` → `s.conversations.filter(c => c.agentId === agentId)`
- `useAllConversations()` → `s.conversations` (all agents)
- `useActiveConversation(agentId)` / `useActiveConversationId(agentId)`

So "both surfaces honor the same filter" is achieved by putting the flag in the store and having **each** surface filter/split default-vs-archived from it — not by one shared list widget. That is small and local, and the two surfaces differ by design (user requirement): the **Assistant panel** shows a collapsible Archived section (collapsed by default), whereas the **Build Studio `<select>`** filters archived conversations out entirely (no archived optgroup) — unarchiving happens in-place via the shared read-only banner or from the Assistant panel.

### 3.2 Cross-app propagation is free — one JS context, one store

Verified in `src/components/desktop/Window.tsx` (line ~207): a built-in window renders via `createElement(AppComponent, …)` **directly in the desktop shell's React tree**; only `manifest.kind === "iframe"` windows get an `<IframeApp>`. `chat` and `build-studio` are built-in (in `src/apps/_components.generated.ts`), so **every built-in window in a given BOS instance runs in the same page, same JS context, same `conversations.ts` module instance**.

Therefore an archive done in the Assistant window calls `setState → notify()` in the shared module, and the Build Studio window's `ConversationSelector` re-renders off the same store on the next tick — **no event bus, no pub/sub, no API round-trip for propagation**. This satisfies FR-005 and SC-004 ("no manual refresh or re-login required") by construction. (The one persistence round-trip — the PATCH to the server — is for durability across restarts, not for in-session propagation.)

### 3.3 Data model

Conversations are one JSON file per chat at `/Documents/Chats/<id>.json`, holding **metadata + full message history** (`src/lib/agent/conversations.ts`, header comment). The metadata fields are parsed defensively field-by-field in `readConversationFile` (e.g. `activeFeatureBranch`), and the server's writer (`src/lib/assistant/conversation-store.ts`) preserves all top-level fields on message save.

The minimal change is **additive**: add an optional `archived?: boolean` to the metadata, defaulting to `false` when absent (which is every existing conversation). No schema version bump, no data migration — a missing field simply reads as `false`.

### 3.4 Context / container (matches BOS reality)

```mermaid
flowchart LR
  subgraph page["One BOS page (single JS context)"]
    A["Assistant window<br/>chat/index.tsx"] -->|render| CP["ConversationPanel<br/>(ConvRow rows)"]
    B["Build Studio window<br/>build-studio/index.tsx"] -->|render| CS["ConversationSelector<br/>(select dropdown)"]
    CP --> S["conversations.ts store<br/>(useSyncExternalStore singleton)"]
    CS --> S
    CI["ChatInputV2<br/>(composer)"] --> S
    S -->|setState+notify| CP
    S -->|setState+notify| CS
  end
  S -->|PATCH /api/assistant/conversations/<id>/archive| R1["archive route<br/>(server)"]
  R1 -->|patchConversationMeta (queued)| F[("/Documents/Chats/<id>.json<br/>archived:true")]
  CI -->|POST /api/assistant/runs| R2["runs route<br/>(server)"]
  R2 -->|getConversationMeta: archived?| F
  R2 -->|"409 if archived (FR-009)"| CI
```

Containers are the two real BOS ones: the Next.js app page (single SPA context holding every built-in window) and the VFS (`/Documents/Chats`). There is no separate service/container introduced.

---

## 4. Concrete file / module plan (only what this feature creates or modifies)

1. **`src/lib/agent/conversations.ts`** — *modify*
   - `Conversation` interface: add `archived?: boolean`.
   - `ConversationFile` interface: add `archived?: boolean`.
   - `readConversationFile`: parse `archived: typeof parsed.archived === "boolean" ? parsed.archived : false` (missing → `false`).
   - `loadFromVfs`: carry `archived` into each built `Conversation`.
   - `newConversation`: include `archived` in the file write (defaults `false`, so effectively a no-op but keeps the shape complete).
   - **`saveConversationMessages`**: carry `archived` over from `existing` when rebuilding the file (so a client message-save can never silently un-archive — mirrors the `activeFeatureBranch` preservation already there). See Risk R1.
   - **New `setConversationArchived(id, archived)`**: optimistic `setState` map (early-return if unchanged, like `setConversationAgent`), then `fetch` the dedicated endpoint. This is the client's single archive/unarchive entry point.

2. **`src/app/api/assistant/conversations/[conversationId]/archive/route.ts`** — *new*
   - `PATCH { archived: boolean }` → `patchConversationMeta(conversationId, { archived })` (from `@/lib/assistant/conversation-store`), return the resulting metadata. 400 on bad body, 404 if the conversation file doesn't exist. Thin delegate, `force-dynamic`, matching the `feature-branches` route shape.

3. **`src/app/api/assistant/runs/route.ts`** — *modify*
   - In `POST`, after the existing `conversationId`/`agentId`/`message` validation and **before** `startAssistantRun`: `const meta = await getConversationMeta(conversationId); if (meta?.archived === true) return NextResponse.json({ error: "Conversation is archived and read-only — unarchive it to continue." }, { status: 409 });`
   - This is the **single authoritative FR-009 gate** (it covers textarea send, Enter-to-send, edit-resubmit, and voice — all funnel through `sendMessage` → this route).

4. **`src/components/apps/assistant/ConversationPanel.tsx`** — *modify*
   - `ConvRow`: replace the two inline hover glyphs (Pencil/Delete) with a single **kebab (`more-vertical`) row menu** (see ADR-5). Menu items are state-dependent: default rows → Rename / **Archive** / Delete; archived rows → Rename / **Unarchive** / Delete. Double-click-to-rename can stay or fold into the menu (impl detail).
   - Single-agent view: split `single.conversations` into a default block + a **collapsible "Archived" section** at the bottom of the scroll area, **collapsed by default** (chevron header + count badge; empty-state text when expanded and none). Collapsed-by-default is the resting state because the feature's whole point is tidying the list.
   - All-agents view: same split **per agent** — each agent bucket appends its own **collapsible "Archived" sub-section, collapsed by default** (see Risk R3, resolved: per-agent buckets, not pooled).
   - Collapse state is per-section local UI state (not persisted); it does not affect the `archived` flag.
   - Archive/Unarchive click → `void setConversationArchived(c.id, !c.archived)`.

5. **`src/components/apps/assistant/AgentSelector.tsx`** — *modify* (`ConversationSelector`)
   - The dropdown lists **only non-archived** conversations — archived ones are filtered out entirely (no archived `<optgroup>`). This is the user's requirement #2: in Build Studio, archived conversations should not show up in the dropdown at all.
   - Add a small **Archive button** beside the existing delete button, acting on the active conversation (same arm-then-confirm pattern the delete button uses) → `void setConversationArchived(activeId, true)`. Since the dropdown never surfaces archived rows, the active conversation in Build Studio is normally non-archived, so this button only needs the *archive* direction.
   - **Unarchiving a Build Studio conversation** is not done from the dropdown: it happens either (a) in-place via the shared `ChatInputV2` read-only banner (item 6), which is also rendered in Build Studio, or (b) from the Assistant app's collapsible Archived section. Both paths write the same shared flag, so the conversation returns to the Build Studio dropdown automatically (store re-render).

6. **`src/components/agent/v2/ChatInputV2.tsx`** — *modify*
   - Read the active conversation's `archived` (via `useActiveConversation` / pass a prop) to render the **client-side read-only state**: replace the composer with a locked placeholder + an amber "This conversation is archived and read-only … **Unarchive to continue**" banner (per the mockup). The banner button calls `void setConversationArchived(conversationId, false)`.
   - This is UX only; the server route (item 3) remains the real block.

7. **`src/lib/agent/conversations.ts`** — *modify* (auto-archive on promote, user requirement #3)
   - **New `archiveConversationsForBranch(branch)`**: **lead with `await ensureLoading()`** — this is load-bearing, not just style. The conversations store is a **lazy singleton** (`state` starts `null`, populated on first subscription via `loadFromVfs`), and NEITHER promote surface (`VersionControls`, `Settings → Versions`) subscribes to it. A user can promote from the topbar with no chat/build-studio window ever open, in which case a bare `get()` returns an empty snapshot and the helper would silently archive nothing. Awaiting `ensureLoading()` guarantees the store is populated before it is read; combined with the caller awaiting the whole helper before reloading (item 8), it also guarantees the archive PATCHes are dispatched before navigation.
   - After loading, iterate `s.conversations` and, for every conversation whose `activeFeatureBranch === branch` (and not already archived), dispatch a **keepalive** archive PATCH. The in-store `activeFeatureBranch` is the existing conversation↔branch link, so "the conversation for a promoted feature" is well-defined without any new persistence or lookup helper.
   - **Dispatch mechanism:** on this path the optimistic `setState` inside `setConversationArchived` is dead weight (the page reloads immediately after, so no UI ever renders the intermediate state) — so `archiveConversationsForBranch` dispatches the keepalive PATCHes directly (a dedicated thin helper) rather than routing each archive through `setConversationArchived`'s optimistic branch. The optimistic update stays and is still required for the MANUAL archive/unarchive UI paths (panel kebab, Build Studio button, read-only banner), which do call `setConversationArchived`. See ADR-7.

8. **Both promote surfaces — `VersionControls` (desktop topbar) and `Settings → Versions`** — *modify*
   - At the existing post-promote site where they call `notifySelfHealBranchSettled(branch, "promoted")` (`src/lib/self-heal/branch-settled-client.ts`), **await** the archive before the reload:
     ```ts
     notifySelfHealBranchSettled(selectedBranch, "promoted");
     await archiveConversationsForBranch(selectedBranch);   // ensureLoading() + keepalive PATCHes dispatched/landed
     … window.location.reload();
     ```
     Both handlers are already `async` (they `await promoteAndWait(...)`), so this is a single added line placed right after the settle call. This ordering — not the `keepalive` flag alone — is what makes it correct: (1) the helper's `await ensureLoading()` runs to completion, so the lazy store is populated even when no chat/build-studio window was ever open, and (2) the keepalive archive PATCHes are dispatched *and* landed before `window.location.reload()` preempts the call — strictly safer than the settle call, which relies on `keepalive` only because it is already dispatched synchronously before it returns. The design does **not** depend on the desktop promote path's optional blocking `window.alert` (`VersionControls.tsx` ~288–291, which fires only when the promote reports issues or a restart) — that delay is incidental and must not be relied on. For a *discarded* branch we do **not** auto-archive (only promote, per the user's request).

No changes are required to `src/lib/assistant/conversation-store.ts`: `patchConversationMeta(id, { archived })` and `getConversationMeta(id)` are already generic over arbitrary metadata keys, and `saveConversationMessages` already preserves top-level fields. (Optionally type `archived?: boolean` on its `ConversationFile` for clarity — cosmetic.)

---

## 5. Integration points (existing mechanisms this design calls into but does not create)

- **`useConversations` / `useAllConversations` / `useActiveConversation`** (`src/lib/agent/conversations.ts`) — the shared store both surfaces and the composer already read; the source of the `archived` flag.
- **`patchConversationMeta` / `getConversationMeta`** (`src/lib/assistant/conversation-store.ts`) — the queue-serialized metadata writer/reader. `patchConversationMeta` already serializes against the agent loop's transcript writes via `enqueuePerKey` — the exact race protection `setConversationActiveFeatureBranch`'s doc comment describes.
- **`enqueuePerKey` / write-queue** (`src/lib/agent/write-queue`) — the per-conversation critical section the archive write joins.
- **`sendMessage` → `POST /api/assistant/runs`** (`src/lib/assistant/client/run-client.ts` → `src/app/api/assistant/runs/route.ts`) — the send path the FR-009 gate sits on.
- **Desktop shell rendering of built-in apps in one page context** (`src/components/desktop/Window.tsx` + `src/components/apps/registry.tsx`) — the mechanism that makes cross-app propagation free (FR-005).
- **`AssistantChatV2`** (`src/components/agent/v2/AssistantChatV2.tsx`) — the shared embed that decides panel-vs-toolbar per host and owns the `activeId`/`ensureConversation` wiring the read-only state reads.
- **The `feature-branches` route** (`src/app/api/assistant/feature-branches/route.ts`) — the existing precedent for a dedicated thin PATCH that funnels a conversation-metadata change through the store queue; the archive route copies its shape.

---

## 6. ADRs

### ADR-1 — Classification: `bos-core`, no new dependency, no service facet
- **Context:** The feature is a data-model flag + UI on two existing windows + one API gate.
- **Options:** (a) `bos-core` (modify conversation store, shared components, one API route); (b) `marketplace-item` service facet; (c) `builtin-app`.
- **Decision:** (a).
- **Alternatives considered:** (b) was explicitly checked and rejected — no raw protocol, no daemon, no own port, no installable unit; there is nothing Next.js "can't do" that a worker thread would. (c) rejected — no new window ships. No external dependency is introduced (state lives in the existing JSON file; the menu/banner use existing `lucide-react` icons).
- **Consequences:** All change is additive to existing `src/` modules and one new thin route; no new runtime surface, no new lifecycle to manage.

### ADR-2 — Enforce FR-009 (read-only) at the server route, not just the client
- **Context:** "Sending a message requires unarchiving first" must hold for every send path.
- **Options:** (a) client-only gate (disable the composer); (b) server gate at `POST /api/assistant/runs` (+ client gate for UX); (c) gate inside `startAssistantRun`.
- **Decision:** (b).
- **Consequences:** Authoritative and bypass-resistant — textarea, Enter, edit-resubmit and voice all converge on `sendMessage` → this route, so one check covers them. The client banner (mockup) is a fast, friendly duplicate, not the source of truth. (c) was considered but is deeper in the run path and would couple the gate to run internals; the route boundary is the natural SSR authority.

### ADR-3 — Persist `archived` via a dedicated endpoint routed through the store queue (not a plain VFS write)
- **Context:** `setConversationActiveFeatureBranch`'s doc comment records a real bug: a plain, unserialized VFS metadata write raced the agent loop's transcript save and silently reverted the user's change (last writer won with a stale field snapshot).
- **Options:** (a) generic `/api/fs` write of the flag; (b) client-side `writeConversationFile` only; (c) dedicated endpoint calling `patchConversationMeta` (queue-serialized).
- **Decision:** (c).
- **Consequences:** The archive write joins the same `enqueuePerKey` critical section as message saves, so it can never be clobbered by a concurrent transcript write (and vice-versa). The optimistic `setState` in `setConversationArchived` gives instant UI feedback; the server write is durability. Avoids re-creating the documented race for a second field.

### ADR-4 — Keep the store "full" and split default/archived at each surface
- **Context:** FR-002 (hide archived from default lists) vs FR-008 (always-visible Archived section) vs FR-009 (the open archived conv must be readable + the unarchive affordance must reach its object).
- **Options:** (a) store returns only non-archived by default + a separate archived selector; (b) store returns the full list (each item carries `archived`), and each surface splits it.
- **Decision:** (b).
- **Consequences:** `useActiveConversation` still returns the archived conv object (needed to render the read-only banner and its Unarchive button), and the all-agents panel can group archived items without a second store query. Cost: each consumer must remember to filter — mitigated by co-locating the split in the one `ConvRow`/`ConversationSelector` each app uses. The filter is a single boolean predicate, so it is cheap and hard to get wrong.

### ADR-5 — Row affordance: kebab menu in the panel (mockup), archive-only button in Build Studio
- **Context:** The current `ConvRow` shows two inline hover glyphs (Pencil rename, Trash2 delete) plus double-click rename. The mockup replaces these with one kebab that opens a state-dependent menu.
- **Options:** (a) add a third inline glyph for Archive; (b) kebab menu (mockup).
- **Decision:** (b) for the panel.
- **Consequences:** A state-dependent **Archive ↔ Unarchive** toggle plus a stable Rename/Delete fits naturally in one menu; three inline hover glyphs would be cramped and can't express the toggle cleanly. It's a contained change to `ConvRow`, which both panel views already share. For Build Studio the surface is a `<select>`, which cannot host a per-row kebab — so, per requirement #2 / FR-008, the `ConversationSelector` **lists only non-archived conversations** (archived ones are filtered out entirely — no archived `<optgroup>`), and it gets a single **archive-only button** acting on the active conversation. Unarchive is not exposed from the dropdown: a Build Studio conversation is restored in-place via the shared `ChatInputV2` read-only banner (which Build Studio also renders) or from the Assistant app's collapsible Archived section — both write the same shared flag, so the conversation returns to the dropdown automatically. Consistent behavior (one shared store + `setConversationArchived`), per-surface affordance shaped by the widget it lives in.

### ADR-7 — Auto-archive on promote reuses the existing conversation↔branch link and the post-promote settle seam
- **Context:** User requirement #3: when a feature branch is promoted, the conversation(s) that carried it should be auto-archived. Two sub-questions: *how do we find the right conversation(s)* and *where do we hook the promotion event*.
- **Options:** (a) key on the conversation's existing `activeFeatureBranch` field, triggered from the client post-promote site; (b) a server-side listener on the `branch-settled` event that scans the VFS for matching conversations; (c) a dedicated new "feature promoted" event/handler.
- **Decision:** (a). Find the conversation(s) via the in-store `activeFeatureBranch === promotedBranch` predicate (already persisted per conversation, already in memory in the shared store). Trigger from the **existing** post-promote site — both promote surfaces already call `notifySelfHealBranchSettled(branch, "promoted")` immediately after a successful promote and right before they reload — and **`await archiveConversationsForBranch(branch)` there, before the `window.location.reload()`**. The await (not fire-and-forget) is the load-bearing part, for two independent reasons: (1) **the store is a lazy singleton** — `state` starts `null`, populated on first subscription via `loadFromVfs`, and NEITHER promote surface subscribes to it, so a topbar promote with no chat/build-studio window ever open would otherwise read an empty snapshot and archive nothing; the helper leads with `await ensureLoading()` (mirroring every sibling mutator in `conversations.ts`) and the caller awaits it, so the store is populated before it is read; (2) **`keepalive` alone is not enough on this path** — unlike `notifySelfHealBranchSettled`, a *synchronous* `fetch(keepalive)` whose request is already in flight before it returns, `archiveConversationsForBranch` is `async` and yields at its first `await`, so a fire-and-forget `void …; window.location.reload();` would let the reload preempt the helper before any PATCH is dispatched (keepalive only protects an *already-dispatched* request). Awaiting before the reload guarantees the PATCHes are dispatched *and* landed first — strictly safer than the settle call, not a weaker copy of it.
- **Why not (b)/(c):** (b) would require a server VFS scan + a new event subscription for what is fundamentally a client-session tidying action; the client already has the full conversation list in the shared store, so scanning the VFS server-side is strictly more moving parts for the same result. (c) invents a new event type where a hook at an existing, already-promote-aware call site is smaller and more discoverable. The settle call itself is the documented "feature branch was promoted to base" seam — so it is the natural owner for *when* — but the archive must be **awaited** (see Decision, reasons 1 and 2), not dispatched-and-forgotten, because its correctness depends on the lazy store being loaded and on its writes being in flight before the reload.
- **Consequences:** No new event, no new route, no VFS scan. The helper is a few lines over the existing store plus a dedicated keepalive-dispatch (it does **not** route through `setConversationArchived`'s optimistic `setState`, which is dead weight on a path that reloads immediately — the optimistic update stays for the manual UI paths, §4 item 7). Idempotent (skips already-archived). Only fires on *promote*, not *discard*. A promoted conversation that was also manually unarchived later stays unarchived (archive is a state, not a one-shot) — no conflict. The "multiple conversations share a branch" edge case is handled by the predicate (all matching conversations are archived). The client-side `activeFeatureBranch === branch` filter is consistent with an established server-side pattern — `src/lib/self-heal/intake.ts:1060` already filters cases by `c.activeFeatureBranch === target` when settling a branch — so the client predicate mirrors a pattern that is already in the codebase, not an invented one. The design does **not** rely on the desktop promote path's optional blocking `window.alert` (`VersionControls.tsx` ~288–291, issues/restart-only); that delay is incidental and is moot once the archive is awaited. The "settle notification lost on reload" edge case is accepted as best-effort for v1 (documented in spec edge cases); a missed auto-archive is recoverable by manual archive, not data loss, and it cannot block the promote (the archive is awaited but a throw from it is non-fatal to the promote).

### ADR-6 — Self-archive: the open conversation stays open and goes read-only
- **Context:** Edge case "what happens when the currently-open conversation is archived from within its own view?"
- **Options:** (a) auto-switch to another conversation; (b) stay open, render read-only.
- **Decision:** (b) — the mockup's behavior.
- **Consequences:** The `activeByAgent` pointer is independent of the `archived` flag, so archiving does **not** change which conversation is "current" (answers the "most recent/active" edge case: no change). Staying open + read-only is the least-surprising outcome and requires zero extra state: `selectConversation`/`activeId` are untouched; the next send simply hits the FR-009 gate and the banner offers Unarchive. Rapid archive↔unarchive is safe because `setConversationArchived` early-returns when the value is unchanged and each write is queue-serialized.

---

## 7. Risks / open questions

- **R1 — Client `saveConversationMessages` dropping the flag.** `src/lib/agent/conversations.ts` has a client-side `saveConversationMessages` that *rebuilds* the file from metadata. The authoritative server writer (`conversation-store.ts`) already preserves top-level fields, so the main loop is safe — but if the client version is still invoked by any live path, it must carry `archived` over from `existing` (planned in item 1; verify at `implement` that no caller reconstructs a file from a partial snapshot). This is the exact failure class as `activeFeatureBranch`.
- **R2 — Multi-tab propagation.** The store is an in-memory module singleton; it does **not** sync across separate browser tabs (a pre-existing property of the store, not introduced here). FR-005/SC-004 are about "the other **app**," which in a single BOS instance is the same tab/context → satisfied. Cross-tab staleness is out of scope for v1 (same as today's behavior for every other conversation change). If it ever matters, it's a separate store-persistence/refresh concern.
- **R3 — All-agents panel layout (RESOLVED: per-agent buckets).** *Decision (user):* the all-agents view uses **per-agent archived sub-sections** — each agent bucket appends its own "Archived" sub-section (divider-headed, count badge, empty-state) beneath that agent's default rows — rather than one pooled section across agents. This keeps the grouped layout internally consistent (an agent's archived chats stay under that agent). The mockup's pooled rendering is superseded for the all-agents case; the single-agent case is unaffected (one bucket = one archived sub-section either way).
- **R4 — Build Studio has no archived section; restore path is indirect.** Because archived conversations are filtered out of the Build Studio dropdown entirely (user requirement #2), a Build Studio conversation that has been auto-archived (or archived elsewhere) is not directly restorable *from Build Studio's own list*. The user restores it either (a) in-place via the shared `ChatInputV2` read-only banner (which Build Studio also renders when it re-opens that conversation), or (b) from the Assistant app's collapsible Archived section. This is a deliberate consequence of the requirement — the tidied dropdown is the goal — and is acceptable for v1; the read-only banner in Build Studio is the same component as in the Assistant, so there is no dead-end. Flag only if the user later wants a Build-Studio-local restore affordance.
- **R5 — Auto-archive trigger reliability (best-effort, but awaited before reload).** `archiveConversationsForBranch` is invoked from the client at the post-promote site, in the same window that then reloads — but it is **awaited before** `window.location.reload()` (see ADR-7), not fire-and-forget. That ordering is what makes it reliable: the helper's leading `await ensureLoading()` guarantees the lazy store is populated (the promote surfaces don't subscribe to it, so a topbar promote with no chat/build-studio window open would otherwise see an empty snapshot and archive nothing), and the awaited keepalive PATCHes are dispatched *and* landed before the reload can preempt them — strictly safer than `notifySelfHealBranchSettled`, which relies on `keepalive` alone because it is already dispatched synchronously before it returns. If a PATCH were nonetheless lost (e.g. the tab is hard-killed after the await returns but before the network layer completes), the conversation would simply not be auto-archived — recoverable by manual archive, never data loss, and it never blocks the promote itself (a throw from the archive helper is non-fatal to the promote; only the auto-archive is skipped). The design does **not** depend on the desktop promote path's optional blocking `window.alert` (`VersionControls.tsx` ~288–291, issues/restart-only) — that delay is incidental and is moot once the archive is awaited. Accepted for v1 per spec edge-case.
- **Edge cases (from spec), resolved by the design:**
  - *Open archived conv's URL / deep reference* → still opens (`useActiveConversation` returns it; `loadConversationMessages` reads it fine) and renders read-only (ADR-6). No redirect needed.
  - *Archived conv deleted via normal flow* → `deleteConversation` removes the file; it disappears from the Archived section too (same store). Nothing to special-case.
  - *Rapid archive/unarchive* → safe (ADR-6): idempotent + queue-serialized.
  - *Is archiving the "current" conv changed?* → No (ADR-6).

---

## 8. UI mockup reference

Path: `mockup.html` (same spec dir). It models the **Assistant** window only (left `ConversationPanel` + chat surface). Mapping onto §3/§4:

- **Left panel, single-agent bucket** → `ConversationPanel` single-agent view: default rows + always-visible **Archived** section with a count badge and empty-state text.
- **Kebab (`more-vertical`) row menu** → `ConvRow` change (ADR-5): default rows show *Rename / Archive / Delete*; archived rows show *Rename / Unarchive / Delete*.
- **Chat header "Read-only" pill** → `ChatInputV2`/`AssistantChatV2` read-only indicator for the active archived conv.
- **Amber "archived and read-only … Unarchive to continue" banner + locked composer** → `ChatInputV2` client gate (ADR-2); the banner button calls `setConversationArchived(id, false)`.
- **Archived row opens read-only; self-archive stays open** → ADR-6.
- **Not in the mockup (design fills the gap):** Build Studio's toolbar `<select>` form (now an archive-only button, no archived optgroup — see ADR-5 as amended and §4 item 5), the all-agents per-agent-Archived-sub-section layout (R3, resolved: per-agent buckets), and the auto-archive-on-promote behavior (ADR-7, no dedicated UI — it is a background tidying action). The mockup's kebab and self-archive-stays-open defaults are both adopted as the cleaner options per the component structure. **Amended for the user's second round of requirements:** the Assistant panel's Archived section is now **collapsible, collapsed by default** (mockup shows it expanded — treat the expanded state as the post-click state only), and the Build Studio mockup does not exist (the `<select>` form is design-only, §4 item 5).
