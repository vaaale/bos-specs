# GSuite Integration — Phase 3 Implementation Plan

> **Status**: APPROVED (re-created after storage loss on 2026-07-04) — implementation is proceeding immediately per user direction.
> **Companion**: [`tasks.md`](./tasks.md) breaks this into ~23 executable tasks (sections 3.0–3.6).
> **Source spec**: [`spec.md`](./spec.md).
> **Predecessors**: Phase 1 shipped the framework + Gmail (see `specs/user-specs/integrations-framework/`); Phase 2 added polling + webhooks. Phase 3 extends the same framework to the remaining GSuite services.

---

## 1. Scope

### 1.1 In scope (Phase 3)
1. **Manifest expansion** — add Drive, Calendar, Contacts `ServiceDefinition`s to the existing `GSUITE_MANIFEST`, extend `supportedScopes`, and export short-alias constants (`DRIVE_SCOPES`, `CALENDAR_SCOPES`, `CONTACTS_SCOPES`).
2. **Framework gaps G1–G8** (mostly cross-service plumbing surfaced by Drive/Calendar work but useful for everyone):
   - **G1** — `gsuiteFetchBinary` sibling of `gsuiteFetch` that returns an `ArrayBuffer` (Drive file downloads).
   - **G2** — `buildUrl(base, path, query)` helper to replace ad-hoc `URLSearchParams` juggling in adapters.
   - **G3** — multipart upload stub in `client.ts` (returns `IntegrationConfigError("multipart_upload_not_yet_implemented")`) so future write methods can be declared; NOT wired to real requests in Phase 3.
   - **G4** — delta-scope OAuth: the `/api/integrations/oauth/start` route accepts `scopes=…` and the UI's `reconnectWithScopes(intId, scopes)` helper opens the popup with only the requested delta so users can grant a single new service without re-consenting to everything.
   - **G5** — Drive-specific UI hint (`DriveConfigSection`) rendered on the Drive service config page above the generic scope toggles, explaining `drive.readonly` vs `drive.file`.
   - **G6** — `registerAdapter()` API on the server-side adapter registry (`actions/adapter-registry.ts`) so a new service registers via `registerAdapter(integrationId, serviceId, entry)` at import time rather than editing a hard-coded map.
   - **G7** — scheduler stub for calendar reminders: `CalendarAdapter.pollUpcomingReminders()` skeleton (returns `[]` in Phase 3; scheduler wiring deferred to a follow-up).
   - **G8** — mock-fetch test harness under `src/lib/integrations/__tests__/` so adapters can be unit-tested without hitting Google.
3. **DriveAdapter** — seven read-only methods:
   - `listFiles`, `getFile`, `searchFiles`, `downloadFile`, `exportFile`, `listFolders`, `getAbout`.
4. **Wiring**:
   - Register Drive with `registerAdapter()`.
   - New descriptor list `DRIVE_METHOD_DESCRIPTORS` (framework-free) + `DRIVE_METHODS` (server-only).
   - `DriveMethodAction` component in `IntegrationActions.tsx` (mirrors `GmailMethodAction`).
   - Capability entries (`gsuite_drive_*`) in `capabilities-registry.ts`.
   - `ScopeGroup` presentational sub-component in the Settings integrations UI so the config view can visually group scopes by service.
5. **Verification** — mock-fetch harness (G8) + unit tests for at least the DriveAdapter read methods and the delta-scope OAuth URL builder.

### 1.2 Explicitly out of scope for Phase 3
- **Drive write operations** (upload, patch, delete, move) — Phase 4. The multipart stub exists but is not wired.
- **Calendar write operations** and full CalendarAdapter — Phase 4.
- **Contacts adapter** implementation — Phase 4. Manifest lists it so the UI shows the service; adapter throws `IntegrationConfigError("service_not_yet_implemented")` from a placeholder file.
- **Drive VFS mount** — a separate feature, out of scope indefinitely.
- **Scheduler wiring** for calendar reminders — the stub method exists; hooking it into the scheduler daemon is a follow-up.
- **Real Google API integration tests** — Phase 3 tests use the mock harness only.

---

## 2. Architecture — where the change lands

### 2.1 Manifest (`services/gsuite/manifest.ts`)
- Add `DRIVE_SCOPES`, `CALENDAR_SCOPES`, `CONTACTS_SCOPES` constants (full URLs).
- Extend `supportedScopes` union.
- Append three `ServiceDefinition`s (`drive`, `calendar`, `contacts`) with their own `configSchema`.
- Preserve existing Gmail service verbatim.

### 2.2 Framework gaps (`services/gsuite/client.ts`, `oauth/manager.ts`, `actions/adapter-registry.ts`)
- **G1 / G2** — additions to `client.ts` (`gsuiteFetchBinary`, `buildUrl`). Same retry contract as `gsuiteFetch`.
- **G3** — `gsuiteMultipartUpload` stub throws `IntegrationConfigError` so callers can be written but never succeed until Phase 4.
- **G4** — no change to `OAuthManager.startFlow` (already accepts `scopes`) but a new client-side helper `reconnectWithScopes` in the settings UI encapsulates the popup flow with a specific scope subset.
- **G6** — `adapter-registry.ts` gains `registerAdapter(integrationId, serviceId, entry)` and the module-load pattern: each adapter file imports the registry and calls `registerAdapter` at the bottom (side-effect). The hard-coded top-level `ADAPTERS` map goes away.

### 2.3 DriveAdapter (`services/gsuite/adapters/drive.ts` + `drive-methods.ts`)
- Mirrors the Gmail pattern:
  - `drive.ts` (server-only): `DriveAdapter extends ServiceAdapter`, per-method `withScope`, uses `gsuiteFetch` / `gsuiteFetchBinary`.
  - `drive-methods.ts` (framework-free): `DRIVE_METHOD_DESCRIPTORS` + `DRIVE_INVOKERS` map (moved server-side).
- Methods (read-only, all guarded by `drive.readonly` or `drive.file`):

| Method | Endpoint | Scope | Notes |
|---|---|---|---|
| `listFiles`         | `GET /drive/v3/files`                 | `drive.readonly` | Supports `q`, `pageSize`, `pageToken`, `orderBy`, `fields`. |
| `getFile`           | `GET /drive/v3/files/{id}`            | `drive.readonly` | Returns full file metadata. `supportsAllDrives=true`. |
| `searchFiles`       | `GET /drive/v3/files?q=…`             | `drive.readonly` | Convenience over `listFiles` with `q` mandatory. |
| `downloadFile`      | `GET /drive/v3/files/{id}?alt=media`  | `drive.readonly` | Uses `gsuiteFetchBinary`; returns `{ contentType, base64 }`. |
| `exportFile`        | `GET /drive/v3/files/{id}/export?mimeType=…` | `drive.readonly` | Google-native docs (Docs/Sheets/Slides). Returns `{ contentType, base64 }`. |
| `listFolders`       | `GET /drive/v3/files?q=mimeType='application/vnd.google-apps.folder'` | `drive.readonly` | Convenience. |
| `getAbout`          | `GET /drive/v3/about?fields=user,storageQuota` | `drive.readonly` | Powers the auth card "Connected as …" line for Drive. |

Binary responses are base64-encoded in the JSON tool result — the LLM handles URLs / attachments via the notification inbox in a later phase; for now the assistant can hand the base64 blob to another action.

### 2.4 UI additions
- `DriveConfigSection.tsx` — informational card at the top of the Drive service config page (`drive.readonly` vs `drive.file` explanation).
- `ScopeGroup.tsx` — presentational grouping wrapper (title + description + children) used by `ServiceConfigView` so future services (Calendar, Contacts) can group scopes visually.
- No new route in `IntegrationsTab`; the existing drill-down already covers Drive because the manifest exposes it as a service.

### 2.5 Assistant tool wiring
- `IntegrationActions.tsx` gains a `DriveMethodAction` component and iterates `DRIVE_METHOD_DESCRIPTORS`. Same `available` / `scope_disabled` handling as Gmail.
- `capabilities-registry.ts` gains 7 `gsuite_drive_*` entries under the existing Integrations group.
- CORE_POLICY needs no change — the existing "scope_disabled" guidance is generic.

---

## 3. Technical decisions

| # | Decision | Rationale |
|---|----------|-----------|
| **D1** | **Read-only Drive in Phase 3.** | Write operations require multipart upload + destructive-action UX polish (confirm modals, undo). Kept out of scope so Drive read shipping isn't blocked. |
| **D2** | **Binary downloads returned as `{ contentType, base64 }`.** | Keeps the tool contract JSON-serialisable so the CopilotKit dispatcher doesn't need a special path. Trade-off: base64 inflates payload ~33%; adapter-side `maxBytes` cap keeps the 8 KB LLM truncation from being reached in practice for anything but tiny files (LLM will call `downloadFile` and then hand off the base64 to a save action). |
| **D3** | **Delta-scope OAuth via popup with `scopes=…` query param.** | Google supports incremental authorisation (`include_granted_scopes=true` already set in Phase 1). The UI just needs to build a popup URL that lists only the newly-requested scopes; Google merges them with previously granted ones. Avoids forcing the user through the whole consent screen again when adding one service. |
| **D4** | **`registerAdapter()` module-load pattern** (G6). | Matches the manifest registry (`registerIntegration`). New adapters add themselves at import time — a single hard-coded `ADAPTERS` map turns into a growth pain when more integrations arrive. |
| **D5** | **Contacts service declared but not implemented.** | Users see the service in the UI (with a disabled state + "Coming Phase 4") so the integration surface is discoverable, but implementation is deferred to keep Phase 3 tight. Adapter file throws `service_not_yet_implemented`. |
| **D6** | **Multipart upload is a stub only.** | Real multipart requests need `Content-Type: multipart/related` construction with base64-encoded bodies — non-trivial and only useful once a write method needs it. Stub returns a clear error; no dead code path in the call graph. |
| **D7** | **Mock-fetch harness under `__tests__/`, no test runner change.** | No test runner is currently wired in package.json. Tests are `*.mock.ts` scripts that live in `__tests__/` and can be executed via `node --experimental-vm-modules` ad hoc; the harness is designed so hooking up Vitest later is a one-liner. Avoids modifying `package.json` in Phase 3. |
| **D8** | **Scope short-aliases stay UI-only.** | Full-URL scope ids remain the wire format everywhere (state, tokens, effective-scopes). Short aliases (`drive.readonly`) appear only in labels via `scopeLabel()`. |

---

## 4. File & module layout

**New files**
```
src/lib/integrations/
├── services/gsuite/
│   ├── adapters/
│   │   ├── drive.ts                       # DriveAdapter (server-only)
│   │   ├── drive-methods.ts               # DRIVE_METHOD_DESCRIPTORS (fw-free)
│   │   ├── calendar.ts                    # placeholder (throws not_yet_implemented)
│   │   └── contacts.ts                    # placeholder (throws not_yet_implemented)
├── __tests__/
│   ├── mock-fetch.ts                      # fetch mock harness
│   ├── drive-adapter.test.ts              # DriveAdapter unit tests
│   └── oauth-delta-scope.test.ts          # buildAuthUrl delta test

src/components/apps/settings/integrations/
├── DriveConfigSection.tsx                 # G5 Drive scope explainer
├── ScopeGroup.tsx                         # presentational grouping
```

**Edited files**
```
src/lib/integrations/services/gsuite/manifest.ts   # add scopes + service defs
src/lib/integrations/services/gsuite/client.ts     # add gsuiteFetchBinary, buildUrl, multipart stub
src/lib/integrations/services/gsuite/index.ts      # side-effect import for drive/calendar/contacts registration
src/lib/integrations/actions/adapter-registry.ts   # registerAdapter() API
src/lib/integrations/oauth/manager.ts              # no change needed (already scope-parameterised)
src/lib/agent/capabilities-registry.ts             # +7 gsuite_drive_* entries
src/components/apps/settings/integrations/useIntegrations.ts   # reconnectWithScopes helper + scopeLabel additions
src/components/apps/settings/integrations/ServiceConfigView.tsx  # DriveConfigSection injection
src/components/agent/IntegrationActions.tsx        # DriveMethodAction registration
docs/dev/integrations.md                           # Drive smoke test walkthrough
```

---

## 5. Definition of Done — per sub-phase

**3.0 — Manifest expansion**
- `getIntegration("gsuite").services` contains `["gmail", "drive", "calendar", "contacts"]` in that order.
- `supportedScopes` includes at least: `drive.readonly`, `drive.file`, `calendar.readonly`, `calendar.events`, `contacts.readonly` (full URLs).
- No behavior change to Gmail service or its scope list.

**3.1 — Framework gaps G1–G8**
- `gsuiteFetchBinary` returns an `ArrayBuffer` on 200 and applies the same retry logic as `gsuiteFetch`.
- `buildUrl` returns a URL string with encoded query params and no trailing `?` when the query object is empty.
- `gsuiteMultipartUpload` throws `IntegrationConfigError("multipart_upload_not_yet_implemented")` — reachable from a unit test.
- `/api/integrations/oauth/start?scopes=a,b,c` produces an auth URL whose `scope` query param is exactly `a b c`.
- `reconnectWithScopes(intId, scopes)` opens a popup at that URL and refreshes state on the `bos-oauth` postMessage.
- `registerAdapter("gsuite", "drive", entry)` populates the registry so `getAdapterEntry("gsuite", "drive")` returns it.
- `CalendarAdapter.pollUpcomingReminders()` exists and returns `[]` (stub) — no scheduler wiring yet.
- Mock-fetch harness can be imported by a test file and used to feed synthetic Google API responses to `gsuiteFetch`/`gsuiteFetchBinary`.

**3.2 — DriveAdapter**
- All seven methods implemented and scope-gated (`drive.readonly` sufficient for all).
- `downloadFile` returns `{ contentType: string, base64: string }` with correct base64 padding.
- `exportFile` supports the standard Google export MIME types (`application/pdf`, `text/csv`, `text/plain`, `text/html`, ...).
- `getAbout` returns `{ user, storageQuota }` (subset).
- `listFolders` filters by folder MIME type without needing the caller to know it.

**3.3–3.5 — Wiring (adapter registry + UI + assistant)**
- Drive service appears in the Settings drill-down under GSuite.
- `DriveConfigSection` renders above the scope toggles on the Drive service page.
- `ScopeGroup` used for at least the Drive page (visual polish; other services can adopt later).
- CopilotKit registers 7 `gsuite_drive_*` actions, each `available` gated by the effective scope set.
- Toggling `drive.readonly` off makes ALL Drive actions unavailable within one render cycle (verified via manual test note in docs).
- Capability registry includes the 7 new ids in the Integrations group.

**3.6 — Verification**
- `mock-fetch.ts` monkey-patches `fetch` in `beforeEach` and restores in `afterEach` (or, until a runner is wired, exposes `install`/`restore` helpers).
- At least 4 DriveAdapter unit tests using the harness:
  1. `listFiles` builds correct query string.
  2. `downloadFile` base64-encodes the binary payload.
  3. `exportFile` sends the `mimeType` query param.
  4. Scope-disabled call throws `IntegrationScopeError`.
- At least 1 OAuth test verifying `scopes` delta ends up in the auth URL.
- `npx tsc --noEmit` and `npm run lint` clean.
- `docs/dev/integrations.md` gains a "Drive smoke test" walkthrough at the bottom of the file.

---

## 6. Risks & mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **Binary payload size** — a 5 MB PDF base64-encodes to ~6.7 MB, blowing out any 8 KB LLM truncation. | Assistant tool result is unusable for real files. | `downloadFile` accepts `maxBytes` (default 256 KB); larger files return `{ error: "too_large", size }` so the LLM can suggest an alternative flow. |
| **Delta-scope popup flow** confuses users when Google's consent screen shows the delta scopes only. | Users unclear which permissions are being granted. | Popup title is set via `window.open("...", "bos-oauth-<intId>")` and the settings UI shows a pre-flight banner listing the exact scopes being requested. |
| **`registerAdapter()` refactor breaks Gmail** if a subtle import-order issue removes Gmail from the map. | Assistant loses all Gmail actions. | Keep the Gmail registration side-effect in `gmail.ts` (import at the bottom); the barrel `services/gsuite/index.ts` imports every adapter so registration always runs. Unit test asserts both `gmail` and `drive` are present after importing the barrel. |
| **Google's Drive `q` syntax** has escaping subtleties (single-quote strings, backslash escaping). | Search queries silently return zero results. | Adapter helper `escapeDriveQueryLiteral(str)` handles quoting; adapter tests cover the common escape paths. |
| **`exportFile` MIME types** are limited by the source file type (Docs export ≠ Sheets export). | Adapter returns 400 with unhelpful message. | JSDoc documents supported MIME per source type; adapter returns Google's error verbatim so the LLM can surface it. |
| **Contacts placeholder** invoked accidentally from the assistant. | 500 error surfaces to the LLM. | Placeholder returns `IntegrationConfigError("service_not_yet_implemented")` which the invoke route maps to a 400 with a clear message. No capability ids are registered for contacts in Phase 3 so the LLM never sees the actions. |
| **Mock-fetch harness leaking** between tests. | Flaky tests once a real runner ships. | Harness exposes explicit `install()`/`restore()` and captures the original `globalThis.fetch` on install. |

---

## 7. Integration points

- **`registerAdapter()`** replaces the hard-coded `ADAPTERS` map in `adapter-registry.ts`. Callers unchanged (still `getAdapterEntry(integrationId, serviceId)`).
- **`IntegrationActions.tsx`** adds a second child list (Drive) that renders `DriveMethodAction` per descriptor; Gmail stays in place.
- **Capabilities registry** — 7 new entries, under the existing group.
- **Settings UI** — no new tab or top-level component; only new sub-components inside `settings/integrations/`.
- **Docs** — one section appended to `docs/dev/integrations.md`.

---

## 8. Non-goals / anti-patterns to avoid

- Do **not** implement Drive writes.
- Do **not** add a new test-runner dep in `package.json`; keep tests runnable ad hoc.
- Do **not** VFS-mount Drive.
- Do **not** couple Drive metadata into `state.json` (adapters remain stateless).
- Do **not** rename or move Gmail files during the refactor — this phase is additive.
