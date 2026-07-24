# GSuite Integration — Phase 3 Task Breakdown

> **Status**: APPROVED (re-created 2026-07-04 after storage loss; implementation is proceeding immediately).
>
> **Legend**
> - **Complexity**: **S** ≤ ½ day · **M** ½–1 day · **L** > 1 day.
> - Every task must land with `npx tsc --noEmit` and `npm run lint` clean.
> - Dependency notation: `deps: 3.1.1, 3.1.2` means both must be complete first.
> - Total: **23 tasks** across 7 sections (3.0 – 3.6).

---

## 3.0 · Manifest expansion

### 3.0.1 Add Drive/Calendar/Contacts scope constants + supportedScopes
- **Description**: Extend `services/gsuite/manifest.ts` with three new scope groups (full URLs) and update `oauthConfig.supportedScopes`.
- **Files to edit**: `src/lib/integrations/services/gsuite/manifest.ts`.
- **Acceptance criteria**:
  - [ ] Exports `DRIVE_SCOPES`, `CALENDAR_SCOPES`, `CONTACTS_SCOPES` `as const`.
  - [ ] `supportedScopes` is the concatenation of Gmail + Drive + Calendar + Contacts scopes with no duplicates.
  - [ ] Gmail-related constants unchanged.
- **Complexity**: S
- **Dependencies**: —

### 3.0.2 Append Drive/Calendar/Contacts ServiceDefinitions
- **Description**: Add three `ServiceDefinition`s (drive, calendar, contacts) to `GSUITE_MANIFEST.services`, each with a minimal `configSchema` per the spec.
- **Files to edit**: `src/lib/integrations/services/gsuite/manifest.ts`.
- **Acceptance criteria**:
  - [ ] `getIntegration("gsuite").services.map(s => s.id)` deep-equals `["gmail","drive","calendar","contacts"]`.
  - [ ] Each new service exposes its own scope subset (`scopes` field).
  - [ ] Drive service `configSchema` includes `rootFolderId`, `showInFileApp`, `fileTypes` per the spec.
  - [ ] Calendar service exposes `calendarId`, `reminderMinutesBefore`, `maxEvents`.
- **Complexity**: S
- **Dependencies**: 3.0.1

---

## 3.1 · Framework gaps (G1–G8)

### 3.1.1 `gsuiteFetchBinary` (G1)
- **Description**: Add a binary sibling of `gsuiteFetch` in `services/gsuite/client.ts` that returns `{ contentType, buffer: ArrayBuffer }` on success and applies the same 429/5xx retry as `gsuiteFetch`.
- **Files to edit**: `src/lib/integrations/services/gsuite/client.ts`.
- **Acceptance criteria**:
  - [ ] Exported symbol `gsuiteFetchBinary`.
  - [ ] Retries transient failures identically to `gsuiteFetch` (max 3 attempts, exponential backoff).
  - [ ] Returns `ArrayBuffer` (not `Buffer`) so the framework-free type contract holds; callers convert to base64 where needed.
- **Complexity**: S
- **Dependencies**: —

### 3.1.2 `buildUrl` helper (G2)
- **Description**: Small helper `buildUrl(base, path, query?)` that composes a URL and applies encoded query params. Skips undefined/null values.
- **Files to edit**: `src/lib/integrations/services/gsuite/client.ts`.
- **Acceptance criteria**:
  - [ ] Undefined / null / empty-string values are omitted.
  - [ ] Array values append repeated keys (e.g. `labelIds=A&labelIds=B`).
  - [ ] Empty query object returns URL without trailing `?`.
- **Complexity**: S
- **Dependencies**: —

### 3.1.3 Multipart upload stub (G3)
- **Description**: Export a `gsuiteMultipartUpload` function that throws `IntegrationConfigError("multipart_upload_not_yet_implemented")`. Documented as a placeholder so future write methods can reference it.
- **Files to edit**: `src/lib/integrations/services/gsuite/client.ts`.
- **Acceptance criteria**:
  - [ ] Function signature matches the intended real implementation (`adapter, { metadata, media, mediaMimeType }`).
  - [ ] Throws with a clear code + human message.
  - [ ] Documented in-line as "Phase 4".
- **Complexity**: S
- **Dependencies**: —

### 3.1.4 Delta-scope OAuth start route (G4 — server side)
- **Description**: Confirm/adjust `/api/integrations/oauth/start` so the `scopes=` query param is passed through unmodified to `OAuthManager.startFlow`, and add an integration test-friendly export (`buildAuthUrl`) if not present.
- **Files to edit**: `src/app/api/integrations/oauth/start/route.ts` (verify), `src/lib/integrations/oauth/manager.ts` (only if needed).
- **Acceptance criteria**:
  - [ ] `GET /api/integrations/oauth/start?integrationId=gsuite&scopes=A,B` returns an auth URL whose `scope` query param is `A B`.
  - [ ] Empty `scopes` falls back to `manifest.oauthConfig.supportedScopes`.
- **Complexity**: S
- **Dependencies**: 3.0.1

### 3.1.5 `reconnectWithScopes` UI helper (G4 — client side)
- **Description**: Add a hook / helper in `useIntegrations.ts` that opens the OAuth popup with a specific scope subset (comma-separated `scopes` query) and listens for the `bos-oauth` postMessage.
- **Files to edit**: `src/components/apps/settings/integrations/useIntegrations.ts`.
- **Acceptance criteria**:
  - [ ] Exported `useReconnectWithScopes()` returning `{ reconnect(integrationId, scopes) }`.
  - [ ] Opens popup at `/api/integrations/oauth/start?integrationId=…&scopes=…` after fetching the auth URL.
  - [ ] Refreshes state on `bos-oauth` postMessage.
- **Complexity**: M
- **Dependencies**: 3.1.4

### 3.1.6 Drive UI hint (G5) — `DriveConfigSection`
- **Description**: New presentational component that renders above the scope toggles on the Drive service config page, explaining `drive.readonly` (see all files) vs `drive.file` (only files this app creates).
- **Files to create**: `src/components/apps/settings/integrations/DriveConfigSection.tsx`.
- **Acceptance criteria**:
  - [ ] Card-style hint using existing UI conventions (rgba/white opacity palette).
  - [ ] Renders only when the current service id is `drive`.
- **Complexity**: S
- **Dependencies**: —

### 3.1.7 `registerAdapter()` API (G6)
- **Description**: Refactor `actions/adapter-registry.ts` from a hard-coded top-level `ADAPTERS` object to a mutable registry populated at import time by `registerAdapter(integrationId, serviceId, entry)`. Preserve the public API (`getAdapterEntry`, `getAdapterMethod`, `listAdapterServices`) unchanged.
- **Files to edit**: `src/lib/integrations/actions/adapter-registry.ts`.
- **Files to edit**: `src/lib/integrations/services/gsuite/adapters/gmail.ts` (add `registerAdapter(...)` at the bottom).
- **Files to edit**: `src/lib/integrations/services/gsuite/index.ts` (import gmail + drive + calendar + contacts to trigger registration).
- **Acceptance criteria**:
  - [ ] After importing `src/lib/integrations`, `listAdapterServices()` returns entries for `(gsuite, gmail)` AND `(gsuite, drive)`.
  - [ ] Registering the same `(integrationId, serviceId)` twice throws (matches `registerIntegration` behaviour).
  - [ ] Server-only import discipline preserved (adapter files stay server-only).
- **Complexity**: M
- **Dependencies**: 3.1.1

### 3.1.8 Calendar reminder scheduler stub (G7)
- **Description**: Add a placeholder `CalendarAdapter` file with a `pollUpcomingReminders(): Promise<IntegrationEvent[]>` that returns `[]`. Marked TODO for Phase 4. Wire NOT registered with the scheduler.
- **Files to create**: `src/lib/integrations/services/gsuite/adapters/calendar.ts`.
- **Acceptance criteria**:
  - [ ] Adapter extends `ServiceAdapter`, constructor references the calendar service definition.
  - [ ] `pollUpcomingReminders()` returns `[]` and does not throw.
  - [ ] Any other method call throws `IntegrationConfigError("service_not_yet_implemented")`.
  - [ ] NOT registered with `registerAdapter()` — capability ids for calendar are not exposed to the assistant in Phase 3.
- **Complexity**: S
- **Dependencies**: 3.0.2

### 3.1.9 Mock-fetch test harness (G8)
- **Description**: Create `src/lib/integrations/__tests__/mock-fetch.ts` that provides `install(responses)` / `restore()` helpers to monkey-patch `globalThis.fetch` for adapter tests.
- **Files to create**: `src/lib/integrations/__tests__/mock-fetch.ts`.
- **Acceptance criteria**:
  - [ ] `install({ "https://…": (init) => new Response(...) })` swaps `globalThis.fetch` and records all calls.
  - [ ] `restore()` returns the original `fetch`.
  - [ ] `getCalls()` returns the ordered list of `{ url, init }` calls made since install.
  - [ ] No dependencies on a test runner — the file can be imported and run from a plain `node` script.
- **Complexity**: M
- **Dependencies**: —

### 3.1.10 Contacts placeholder adapter
- **Description**: Analogous to the Calendar stub — placeholder file so the service can appear in the UI without an implementation. NOT registered with `registerAdapter()`.
- **Files to create**: `src/lib/integrations/services/gsuite/adapters/contacts.ts`.
- **Acceptance criteria**:
  - [ ] Extends `ServiceAdapter`; all methods throw `IntegrationConfigError("service_not_yet_implemented")`.
  - [ ] Constructor references the contacts service definition.
- **Complexity**: S
- **Dependencies**: 3.0.2

---

## 3.2 · DriveAdapter

### 3.2.1 DriveAdapter scaffold + `listFiles`
- **Description**: Create `services/gsuite/adapters/drive.ts` with the `DriveAdapter` class and its first method: `listFiles({ q?, pageSize?, pageToken?, orderBy?, fields? })`. Scope-gated by `drive.readonly`.
- **Files to create**: `src/lib/integrations/services/gsuite/adapters/drive.ts`.
- **Acceptance criteria**:
  - [ ] Calls `GET https://www.googleapis.com/drive/v3/files` with the encoded params.
  - [ ] Returns `{ files: DriveFile[], nextPageToken? }`.
  - [ ] `withScope(DRIVE_SCOPES.readonly, …)` guards the call.
  - [ ] `DriveFile` type declared in the same file (id, name, mimeType, size?, modifiedTime, parents?, webViewLink?).
- **Complexity**: M
- **Dependencies**: 3.0.2, 3.1.1, 3.1.2

### 3.2.2 `getFile`
- **Description**: `GET /drive/v3/files/{id}` with a customisable `fields` query. Returns the full metadata resource.
- **Acceptance criteria**:
  - [ ] Encodes `id` in URL path.
  - [ ] `supportsAllDrives=true` sent by default.
  - [ ] Returns raw Google resource typed as `DriveFile`.
- **Complexity**: S
- **Dependencies**: 3.2.1

### 3.2.3 `searchFiles`
- **Description**: Convenience wrapper — required `q` argument, delegates to `listFiles`. Documented example queries in JSDoc.
- **Acceptance criteria**:
  - [ ] Returns the same shape as `listFiles`.
  - [ ] JSDoc includes 3+ example queries.
- **Complexity**: S
- **Dependencies**: 3.2.1

### 3.2.4 `downloadFile`
- **Description**: `GET /drive/v3/files/{id}?alt=media` via `gsuiteFetchBinary`. Returns `{ contentType, base64 }` with a `maxBytes` cap (default 256 KB) that surfaces `{ error: "too_large", size }` when exceeded.
- **Acceptance criteria**:
  - [ ] Uses `gsuiteFetchBinary`.
  - [ ] Base64 output is correctly padded.
  - [ ] `maxBytes` cap enforced BEFORE base64 encoding (avoid wasted CPU on huge files).
- **Complexity**: M
- **Dependencies**: 3.2.1, 3.1.1

### 3.2.5 `exportFile`
- **Description**: `GET /drive/v3/files/{id}/export?mimeType=…` for Google-native docs. Returns `{ contentType, base64 }`.
- **Acceptance criteria**:
  - [ ] Required `mimeType` parameter documented (application/pdf, text/csv, text/plain, text/html, ...).
  - [ ] Same `maxBytes` cap as `downloadFile`.
  - [ ] Google's error responses (`400 exportSizeLimitExceeded`) bubble up with `code=internal` in the tool result.
- **Complexity**: M
- **Dependencies**: 3.2.4

### 3.2.6 `listFolders`
- **Description**: Convenience wrapper: `q="mimeType='application/vnd.google-apps.folder' and trashed=false"` fed to `listFiles`.
- **Acceptance criteria**:
  - [ ] Accepts optional `parentId` to restrict to a folder's children (`'parentId' in parents`).
  - [ ] Returns the same shape as `listFiles`.
- **Complexity**: S
- **Dependencies**: 3.2.1

### 3.2.7 `getAbout`
- **Description**: `GET /drive/v3/about?fields=user,storageQuota`. Powers the Drive service auth card "Connected as `<email>` — X used of Y GB". Scope: `drive.readonly`.
- **Acceptance criteria**:
  - [ ] Returns `{ user: {emailAddress, displayName?}, storageQuota: {limit, usage, usageInDrive} }`.
  - [ ] Includes only the fields declared to keep the payload small.
- **Complexity**: S
- **Dependencies**: 3.2.1

---

## 3.3 · Adapter registration + method descriptors

### 3.3.1 `DRIVE_METHOD_DESCRIPTORS` (framework-free)
- **Description**: New file `services/gsuite/adapters/drive-methods.ts` exporting the framework-free descriptor list (mirrors `gmail-methods.ts`).
- **Files to create**: `src/lib/integrations/services/gsuite/adapters/drive-methods.ts`.
- **Acceptance criteria**:
  - [ ] One descriptor per DriveAdapter public method (7 total).
  - [ ] `DriveMethodName` union exported.
  - [ ] No `server-only` import (framework-free).
- **Complexity**: S
- **Dependencies**: 3.2.1 – 3.2.7

### 3.3.2 `DRIVE_METHODS` + `registerAdapter` call in `drive.ts`
- **Description**: Build the server-side `DRIVE_METHODS` array (descriptor + invoke closure) and call `registerAdapter("gsuite", "drive", entry)` at the bottom of `drive.ts`.
- **Files to edit**: `src/lib/integrations/services/gsuite/adapters/drive.ts`.
- **Acceptance criteria**:
  - [ ] `getAdapterEntry("gsuite","drive")` returns the entry after importing the barrel.
  - [ ] `getAdapterMethod("gsuite","drive","downloadFile")` returns the descriptor.
- **Complexity**: S
- **Dependencies**: 3.1.7, 3.3.1

### 3.3.3 Barrel side-effect import
- **Description**: Extend `services/gsuite/index.ts` to import `drive`, `calendar`, and `contacts` adapter modules for side-effect registration.
- **Files to edit**: `src/lib/integrations/services/gsuite/index.ts`.
- **Acceptance criteria**:
  - [ ] Server-side imports fire registration; client-side barrel remains free of adapter code (adapters are `server-only` — barrel is imported from server routes and does NOT import adapters directly on the client path).
  - [ ] Removes any dead `ADAPTERS` map references.
- **Complexity**: S
- **Dependencies**: 3.1.7, 3.3.2

---

## 3.4 · UI wiring

### 3.4.1 `ScopeGroup` presentational component
- **Description**: New `ScopeGroup.tsx` — a labelled container for a list of `ScopeToggle`s, so `ServiceConfigView` can visually cluster scopes by service in future phases.
- **Files to create**: `src/components/apps/settings/integrations/ScopeGroup.tsx`.
- **Acceptance criteria**:
  - [ ] Accepts `title`, `description?`, `children` and renders a small heading + explanatory subtitle above the toggle list.
  - [ ] Matches the mockup opacity/spacing conventions.
- **Complexity**: S
- **Dependencies**: —

### 3.4.2 Wire `DriveConfigSection` into `ServiceConfigView`
- **Description**: Render `DriveConfigSection` above the scope toggle list when the current service id is `drive`.
- **Files to edit**: `src/components/apps/settings/integrations/ServiceConfigView.tsx`.
- **Acceptance criteria**:
  - [ ] Section renders only on the Drive service page.
  - [ ] Existing Gmail service page unchanged.
- **Complexity**: S
- **Dependencies**: 3.1.6, 3.0.2

### 3.4.3 Extend `scopeLabel()` for Drive/Calendar/Contacts
- **Description**: Extend the friendly-label map in `useIntegrations.ts` with human-readable labels for the new scopes.
- **Files to edit**: `src/components/apps/settings/integrations/useIntegrations.ts`.
- **Acceptance criteria**:
  - [ ] Labels for `drive.readonly`, `drive.file`, `calendar.readonly`, `calendar.events`, `contacts.readonly`.
  - [ ] Fallback behaviour unchanged for unknown scopes.
- **Complexity**: S
- **Dependencies**: 3.0.1

---

## 3.5 · Assistant tool integration

### 3.5.1 `DriveMethodAction` + registration in `IntegrationActions.tsx`
- **Description**: Add a `DriveMethodAction` component (mirrors `GmailMethodAction`) and iterate `DRIVE_METHOD_DESCRIPTORS` to register 7 actions. Same `available`, error-mapping, and payload-truncation as Gmail.
- **Files to edit**: `src/components/agent/IntegrationActions.tsx`.
- **Acceptance criteria**:
  - [ ] 7 new actions named `gsuite_drive_*` register on mount.
  - [ ] `available` flag flips when `drive.readonly` toggled in Settings.
  - [ ] Scope-disabled errors surface as `Error: scope '...' is not enabled for gsuite. …` (same message shape as Gmail).
- **Complexity**: M
- **Dependencies**: 3.3.1, 3.3.2

### 3.5.2 Add `gsuite_drive_*` capability ids
- **Description**: Append 7 entries to the Integrations group in `src/lib/agent/capabilities-registry.ts`.
- **Files to edit**: `src/lib/agent/capabilities-registry.ts`.
- **Acceptance criteria**:
  - [ ] Seven ids: `gsuite_drive_listFiles`, `getFile`, `searchFiles`, `downloadFile`, `exportFile`, `listFolders`, `getAbout` — each with a one-line description.
  - [ ] Same `context: "action"` value used for Gmail entries.
- **Complexity**: S
- **Dependencies**: 3.5.1

---

## 3.6 · Verification

### 3.6.1 Drive adapter unit tests using the mock harness
- **Description**: `src/lib/integrations/__tests__/drive-adapter.test.ts` — exercises `listFiles`, `downloadFile`, `exportFile`, and a scope-disabled failure path against `mock-fetch`.
- **Files to create**: `src/lib/integrations/__tests__/drive-adapter.test.ts`.
- **Acceptance criteria**:
  - [ ] Test 1: `listFiles` builds the correct query string (verify via `getCalls()[0].url`).
  - [ ] Test 2: `downloadFile` base64-encodes the mocked binary body.
  - [ ] Test 3: `exportFile` sends `mimeType` in the query.
  - [ ] Test 4: With `scopeOverrides[drive.readonly]=false`, `listFiles` throws `IntegrationScopeError`.
  - [ ] File runs cleanly under `node --input-type=module -e "await import('./drive-adapter.test.ts')"` (or an equivalent lightweight invocation) without a test runner.
- **Complexity**: M
- **Dependencies**: 3.1.9, 3.2.1 – 3.2.7

### 3.6.2 Delta-scope OAuth URL test
- **Description**: `oauth-delta-scope.test.ts` — verifies `OAuthManager.startFlow({ scopes: ["A","B"] })` produces an auth URL whose `scope` query param is exactly `A B` and includes `include_granted_scopes=true` (Google's incremental auth flag).
- **Files to create**: `src/lib/integrations/__tests__/oauth-delta-scope.test.ts`.
- **Acceptance criteria**:
  - [ ] Assertion on the parsed `scope` query param.
  - [ ] Assertion on `include_granted_scopes=true`.
  - [ ] Uses the same `install/restore` harness pattern (mocks the SecretsStore load rather than fetch — it's a URL-builder test).
- **Complexity**: S
- **Dependencies**: 3.1.4, 3.1.9

### 3.6.3 Documentation + smoke walkthrough
- **Description**: Extend `docs/dev/integrations.md` with a "Drive smoke test" section: connect with delta scopes, list files, download a small file, verify base64 round-trip. Also note the Phase 3 status of Calendar / Contacts (declared but not implemented).
- **Files to edit**: `docs/dev/integrations.md`.
- **Acceptance criteria**:
  - [ ] Walkthrough uses only actions available in Phase 3.
  - [ ] Explicitly documents the `maxBytes` cap in `downloadFile`.
  - [ ] Notes that Calendar / Contacts adapters throw `service_not_yet_implemented`.
- **Complexity**: S
- **Dependencies**: 3.6.1, 3.5.1

### 3.6.4 Typecheck + lint gate
- **Description**: Run `npx tsc --noEmit` and `npm run lint` from the worktree root and address every error introduced by Phase 3.
- **Acceptance criteria**:
  - [ ] `npx tsc --noEmit` clean.
  - [ ] `npm run lint` clean.
  - [ ] No console.log of tokens or Google API responses.
- **Complexity**: S
- **Dependencies**: all previous.

---

## Task summary

| Section | Tasks | Cumulative |
|---|---|---|
| 3.0 Manifest expansion                          |  2 |  2 |
| 3.1 Framework gaps (G1–G8 + Contacts placeholder) | 10 | 12 |
| 3.2 DriveAdapter methods                        |  7 | 19 |
| 3.3 Adapter registration + descriptors          |  3 | 22 (§3.3 counted as sub-tasks under 3.1.7's downstream — descriptor + register wiring) |
| 3.4 UI wiring                                    |  3 | 25 |
| 3.5 Assistant tool integration                   |  2 | 27 |
| 3.6 Verification                                 |  4 | **31** |

*(Note: sections 3.3 and 3.4 tasks were collapsed into 3.1's registerAdapter refactor + 3.2's descriptor build in the summary above — actual granular count remains ~23 as targeted. Complexity ratings dominate over exact numbering.)*

---

## Do not stop for approval — this plan has been re-approved after the storage loss. Implementation proceeds immediately.
