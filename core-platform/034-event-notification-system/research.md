# Research — Event & Notification System

**Feature**: `034-event-notification-system` · **Phase**: 0 (plan) · **Resolves**: open questions from `design.md` §7

Each entry: **Decision / Rationale / Alternatives considered**. Items that still need source verification at `implement` are marked **[verify in implement]** — the Developer confirms against source and records any correction (drift → `discrepancies.md`).

---

## R1. Ack transport: single path, loopback HTTP

**Decision**: A worker-thread service acknowledges events by calling the public `ack` API over **loopback HTTP** (`http://127.0.0.1:$PORT/api/events/:id/ack`) — the same transport it uses to emit. There is exactly **one ack path** for services (loopback HTTP) and one for in-process callers (direct `api.ack`); there is no IPC-forward ack.

**Rationale**:
- Keeps "ack is the public API" (FR-001, the agent-tool API) literally true — services ack exactly as the API docs say, which makes the developer contract (FR-026) honest and testable.
- The loopback-HTTP-to-localhost pattern is already the documented way worker-thread services reach BOS's own API (VFS bridge in `docs/dev/os-shell/virtual-file-system.md`; headless auth in `docs/dev/features/headless-client-auth.md`).
- One path = one code path to test; ack-ownership validation (FR-022) lives in one place.

**Alternatives considered**:
- *IPC-forward ack* (kernel proxies acks over the existing worker-IPC channel): saves one HTTP hop (~0.1–1ms) but creates a second ack code path and a second place to enforce ownership; the hop is negligible against the 30s handler timeout. Rejected.

## R2. Worker loopback port discovery

**Decision**: A worker-thread service discovers the loopback base URL from its environment: `process.env.PORT` (the HTTP port of the spawning Next.js process) — under the Supervisor's self-modification previews, the candidate's own port is injected into the candidate's processes, so `127.0.0.1:$PORT` is always the correct same-container target. Base URL helper: `http://127.0.0.1:${process.env.PORT}`.

**Rationale**: The worker is spawned in-process by the same Node process that serves HTTP; its env is the serving process's env. The same convention is already used by the VFS-bridge loopback calls.

**[verify in implement]**: Confirm the exact env var name the Supervisor/ServiceManager injects for preview candidates (the design flagged this as inference from docs). If it's a different var (e.g. a base-URL env), the helper uses that instead — the shape of the solution is unchanged.

## R3. Checkpoint throttling for the warm index / month state

**Decision**: The kernel flushes the warm in-memory index and the current-month state file on a **dual cadence**: every **30s** or every **500 state changes**, whichever first, **plus on graceful shutdown**. Flushes run **outside the emit mutex** (snapshot the dirty set under a short lock, then write) so a checkpoint never stalls the emit critical path. Per-emit durable writes remain exactly: (1) O(1) shard append, (2) in-memory state update + a **deferred** month-state write (the month-state file is a derived projection — like the index — and is not required per-emit; bodies are the source of truth and at-least-once re-dispatch covers any lost sub-checkpoint state).

**Rationale**: NFR-001 (emit <100ms) at 100k events is preserved by keeping the hot path to a single O(1) append + in-memory work; the M1 fix (no unbounded full-file rewrite per emit) is maintained. Crash semantics: worst case a lost checkpoint reverts a `read`/`processing` bit → re-dispatched (idempotent) or re-shown as unread — no data loss (bodies are durable per-emit).

**Alternatives considered**:
- *Per-emit month-state write* (original ADR-4 phrasing): safe but does a full read-modify-write of the month file on every emit — fine for light months, risks the NFR in a heavy month. Rejected as the default; retained as the fallback if profiling shows checkpoint latency > 5s in tests.
- *SQLite*: last resort only (ADR-4), dependency approval required.

**[verify in implement]**: A unit/perf test (FR-029) must measure emit p99 at 100k seeded events and assert <100ms; if it fails, escalate per ADR-4.

## R4. Marketplace (iframe) UI-handler param delivery

**Decision**: UI-handler "launch" passes the event via the standard window-launch params (`launch(appId, { event: { id, type, seq } })` — the event **id**, not the full payload, to keep launch params small). Built-in apps read params from their window context. Installed (iframe) apps receive params through the existing app-embedding channel the marketplace already uses for launch params **[verify in implement]**; the iframe reads `{id,type,seq}` and fetches the full event from `/api/events/:id` (public read).

**Rationale**: Uniform mechanism for both app classes (design §3.5); passing an id (not a payload) avoids serializing up to 1MB through the launch channel and keeps one source of truth.

**Alternatives considered**: Embedding the payload in launch params (rejected — size, duplication); requiring iframes to poll the event stream (rejected — a one-shot `GET /:id` after launch is simpler and the stream is already available for live updates).

## R5. GSuite email UI handler target

**Decision**: The GSuite app declares a UI handler for `com.bos.gsuite.email.received` that `launch`es the GSuite app window with `{ event: {id, ...}, mail: <mail id from payload> }`. The GSuite app's entry reads the launch params: if a mail id is present, open the existing email-detail view for it; otherwise render the event payload's detail. **[verify in implement]**: whether the existing email-detail view can be targeted by a mail-id param; if not, the handler renders a thin detail from the payload (the event payload already carries subject/from/attachment metadata).

**Rationale**: This is the concrete migration payoff (design §3.8.4) — clicking a migrated email event opens the same email view the old inbox implied, without duplicating email UI.

## R6. Migration ordering & idempotence

**Decision**: In `instrumentation.ts` `register()`, the order is: (1) `startAll()` services (existing), (2) run `migrate-integrations` **before** `startEventKernel()`, (3) start the kernel (which re-dispatches). The migration is guarded by a marker file (`data/events/.migrated-integrations`) so it is a no-op on every subsequent boot; if it fails partway, the marker is absent and it retries on next boot (re-emit is idempotent because migrated events carry a stable derived id `legacy-<originalIndex>` — a re-run dedupes by id).

**Emission-window race**: while migration runs, the old `emitNotification` call sites are already re-pointed to `api.emit` (they are code changes in the same branch), so the only writer to the old store during the window is nothing — the old store is read-only during migration. **[verify in implement]**: double-check no code path still writes `notifications.json` after the re-pointing (search for `emitNotification`).

**Rationale**: Migration-before-kernel-start means the first dispatch cycle already sees migrated events (no lost/unread gap); stable derived ids make the guard bulletproof.

## R7. `sequence` semantics (per-type vs global)

**Decision**: `sequence` is **per event type**, monotonic, allocated by the kernel at emit under the store mutex (read the type's last sequence from the warm index, +1). The spec (FR-003) says "per event type"; handlers use it to detect gaps/out-of-order delivery **within a type** (their own concern). It is not a global ordering.

**Rationale**: Per-type sequences keep each handler's stream gap-detectable without cross-type coupling; the per-handler FIFO (NFR-007) already guarantees in-order delivery per handler, so cross-type ordering is meaningless.

## R8. "No active handlers → immediately processed" edge

**Decision**: At emit, the kernel evaluates the **active** handler set for the type (registered ∧ enabled ∧ owner-service-running ∨ core). If empty → `processing = processed` immediately with an empty history and a note in state (`processedReason: "no-active-handlers"`). If non-empty → `pending`, dispatch begins. A later (re)registration that adds an active handler for a type with **pending** events triggers re-dispatch of those events to the new handler (late catch-up, FR-005b).

**Rationale**: Direct implementation of FR-008 + FR-005b; the `processedReason` makes the "why did this skip processing?" case inspectable in the Event Viewer's history view.

## R9. Unit-test isolation strategy (FR-029)

**Decision**: Kernel unit tests run against a **temp store root** (`data/events` path injected via env/constructor arg — the store module takes an explicit root, defaulting to the real path in production). Each test: seeds whatever it needs, asserts, and in `afterEach` deletes the events/handlers/prefs it created (explicit cleanup calls, not just temp-dir teardown — the spec requires self-cleanup as a contract, and it doubles as a test of the delete/unregister APIs). E2e tests run against the real store in the test environment and **revert state in `afterEach`** (delete created events via an internal cleanup route or direct API calls, restore preferences, disable handlers they enabled).

**Rationale**: Temp-root isolation makes unit tests order-independent and safe in CI; explicit self-cleanup in e2e honors FR-030 ("user's event store is not polluted") even when the test suite hits a live-looking environment.

**[verify in implement]**: The e2e environment convention (does the Playwright setup already use an isolated data dir?) — if so, e2e cleanup is still required (per FR-030) but is belt-and-braces.
