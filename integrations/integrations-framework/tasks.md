# Integrations Framework — Phase 1 Task Breakdown

> **Status**: DRAFT — awaiting user approval. Do not begin work until
> [`plan.md`](./plan.md) is approved.
>
> **Legend**
> - **Complexity**: **S** ≤ ½ day · **M** ½–1 day · **L** > 1 day.
> - Every task must land with `npx tsc --noEmit` and `npm run lint` clean.
> - Dependency notation: `deps: 1.1.1, 1.2.3` means both must be complete first.
> - Total: **41 tasks** across 14 sections (1.0 – 1.13).

---

## 1.0 · Setup & Shared Types

### 1.0.1 Create integrations module skeleton + shared types
- **Description**: Create the `src/lib/integrations/` directory tree with empty
  index modules and the framework-free types described in `plan.md §4`.
- **Files to create**:
  - `src/lib/integrations/index.ts`
  - `src/lib/integrations/types.ts` — `IntegrationManifest`,
    `ServiceDefinition`, `OAuthConfig`, `OAuthTokens`, `IntegrationState`,
    `IntegrationEvent`, `JSONSchema` type alias.
  - `src/lib/integrations/errors.ts` — `IntegrationError` base +
    `IntegrationScopeError`, `IntegrationAuthError`,
    `IntegrationConfigError`.
- **Acceptance criteria**:
  - [ ] All types compile with `npx tsc --noEmit`.
  - [ ] No `import "server-only"` in `types.ts` / `errors.ts` (framework-free).
  - [ ] Barrel `index.ts` re-exports types + errors.
- **Complexity**: S
- **Dependencies**: —

### 1.0.2 Add integrations data-dir helpers
- **Description**: Add small helper to resolve `data/integrations/` paths
  consistently, matching the style of `src/os/data-dir.ts`.
- **Files to create/edit**:
  - `src/lib/integrations/paths.ts` — `integrationsRoot()`, `secretsFile()`,
    `keyfilePath()`, `stateFile(integrationId)`.
  - (edit) `.gitignore` — verify `data/` already covers it (no change if so).
- **Acceptance criteria**:
  - [ ] Paths derive from `dataDir()` (import from `src/os/data-dir.ts`) — no
        hard-coded `./data`.
  - [ ] Functions create missing parent directories on first call.
- **Complexity**: S
- **Dependencies**: 1.0.1

---

## 1.1 · SecretsStore

### 1.1.1 Keyfile + AES-256-GCM crypto helpers
- **Description**: Implement key-loading (create-if-missing, chmod 600) and
  AES-256-GCM encrypt/decrypt helpers using `node:crypto`.
- **Files to create**:
  - `src/lib/integrations/secrets/keyfile.ts` — `loadOrCreateKey(): Buffer`.
  - `src/lib/integrations/secrets/crypto.ts` — `encrypt(plaintext) →
    {iv, ciphertext, tag}`; `decrypt({iv, ciphertext, tag}) → plaintext`.
- **Acceptance criteria**:
  - [ ] Keyfile is 32 bytes, mode `0o600`.
  - [ ] Round-trip encrypt→decrypt of a known plaintext succeeds.
  - [ ] Wrong key ⇒ decrypt throws.
  - [ ] No use of deprecated `crypto.createCipher` (must use `createCipheriv`).
- **Complexity**: M
- **Dependencies**: 1.0.2

### 1.1.2 SecretsStore public API
- **Description**: Implement `SecretsStore` with `set / get / delete /
  listKeys`, serialised writes, atomic file replace.
- **Files to create**:
  - `src/lib/integrations/secrets/store.ts` — `SecretsStore` class + module
    singleton `getSecretsStore()`.
- **Acceptance criteria**:
  - [ ] `import "server-only"` at the top of `store.ts`.
  - [ ] Uses `src/os/atomic-write.ts` for the on-disk write.
  - [ ] Concurrent `set` calls for different keys do not corrupt the file
        (in-process serial queue).
  - [ ] `listKeys("gsuite")` returns only keys belonging to `gsuite`.
  - [ ] Manual scratch-test round-trips `set → get → delete → get(null)`.
- **Complexity**: M
- **Dependencies**: 1.1.1

---

## 1.2 · OAuth Manager

### 1.2.1 PKCE + pending-flow state
- **Description**: PKCE verifier/challenge helpers and an in-memory
  pending-flow map keyed by `state`, with 10-minute TTL.
- **Files to create**:
  - `src/lib/integrations/oauth/pkce.ts` — `newVerifier()`,
    `challengeFromVerifier(verifier)`.
  - `src/lib/integrations/oauth/state.ts` — `putPending({integrationId,
    verifier, scopes}) → state`, `takePending(state) → PendingFlow | null`.
- **Acceptance criteria**:
  - [ ] Challenge is `base64url(sha256(verifier))` (RFC 7636 method S256).
  - [ ] `takePending` deletes the entry (single-use).
  - [ ] Entries older than 10 min are pruned on read.
- **Complexity**: S
- **Dependencies**: 1.0.1

### 1.2.2 OAuthManager: startFlow + handleCallback
- **Description**: Build the authorisation URL from a manifest + user-uploaded
  client credentials; exchange code for tokens on callback; persist tokens to
  SecretsStore and metadata to the integration state.
- **Files to create**:
  - `src/lib/integrations/oauth/manager.ts` — `getOAuthManager()` singleton
    with `startFlow`, `handleCallback`.
- **Acceptance criteria**:
  - [ ] Authorisation URL contains `code_challenge`, `code_challenge_method=S256`,
        `state`, `access_type=offline`, `prompt=consent` for Google.
  - [ ] Callback exchanges code with a POST to `tokenUrl` using client
        credentials fetched via SecretsStore.
  - [ ] On success: `tokens:<integrationId>` written to SecretsStore; state
        updated with `granted_scopes` and `expires_at`.
  - [ ] On error: state marked with `lastError` and secrets untouched.
- **Complexity**: L
- **Dependencies**: 1.1.2, 1.2.1

### 1.2.3 OAuthManager: token refresh + getValidToken
- **Description**: Auto-refresh access tokens when within 60 s of expiry;
  share a single refresh promise across concurrent callers.
- **Files to edit**:
  - `src/lib/integrations/oauth/manager.ts`.
- **Acceptance criteria**:
  - [ ] `getValidToken("gsuite")` returns a non-expired access token.
  - [ ] Two parallel `getValidToken` calls near expiry share one network round-trip.
  - [ ] `refreshToken` updates SecretsStore + state atomically.
  - [ ] If refresh fails with `invalid_grant`, state is set to
        `connected: false` and the caller receives `IntegrationAuthError`.
- **Complexity**: M
- **Dependencies**: 1.2.2

---

## 1.3 · Integration Registry

### 1.3.1 Static registry + accessors
- **Description**: Central `IntegrationRegistry` that receives manifests at
  module load and exposes lookup helpers. Framework-free.
- **Files to create**:
  - `src/lib/integrations/registry.ts` — `registerIntegration(m)`,
    `listIntegrations()`, `getIntegration(id)`, `getService(intId, svcId)`.
- **Acceptance criteria**:
  - [ ] Duplicate `id` registration throws.
  - [ ] `listIntegrations()` returns manifests in stable order (insertion).
  - [ ] No `server-only` import — works in a client component too.
- **Complexity**: S
- **Dependencies**: 1.0.1

---

## 1.4 · State Store

### 1.4.1 Per-integration state store
- **Description**: Read / write `data/integrations/<id>/state.json` atomically;
  provide a `mutate(id, updater)` helper that reads, applies `updater`, writes.
- **Files to create**:
  - `src/lib/integrations/state/store.ts` — `readState(id)`, `writeState(id,
    state)`, `mutateState(id, (prev) => next)`.
- **Acceptance criteria**:
  - [ ] `readState` on a missing file returns a default `IntegrationState`.
  - [ ] `mutateState` is serialised per-integration (in-process lock).
  - [ ] Writes use `src/os/atomic-write.ts`.
  - [ ] `server-only` at top of file.
- **Complexity**: M
- **Dependencies**: 1.0.2

---

## 1.5 · Service Adapter Base

### 1.5.1 `ServiceAdapter` abstract + `withScope` decorator
- **Description**: Provide the base class every service adapter extends, plus a
  `withScope(scope, fn)` helper that throws `IntegrationScopeError` if the
  scope is not in the effective set (see `plan.md D6`).
- **Files to create**:
  - `src/lib/integrations/adapters/base.ts`.
- **Acceptance criteria**:
  - [ ] `getEffectiveScopes()` correctly applies user overrides to granted
        scopes (matches `spec.md §Scope Override UI Logic`).
  - [ ] `withScope("gmail.send", handler)` throws when user has disabled the
        scope; runs the handler otherwise.
  - [ ] `authedFetch(url, init)` attaches a bearer token from
        `OAuthManager.getValidToken` and retries once on `401`.
- **Complexity**: M
- **Dependencies**: 1.2.3, 1.4.1

---

## 1.6 · API Routes

### 1.6.1 `/api/integrations` list + `/api/integrations/[id]` detail
- **Description**: Two Next.js App-Router routes returning the registry state.
- **Files to create**:
  - `src/app/api/integrations/route.ts` — `GET` list.
  - `src/app/api/integrations/[id]/route.ts` — `GET` detail + `PATCH`
    (enable/disable services, scope overrides, per-service config).
- **Acceptance criteria**:
  - [ ] Routes are server-only (Node runtime).
  - [ ] `GET` returns `[{ manifest, state }]`.
  - [ ] `PATCH` validates that a scope override can only be `false` for
        granted scopes.
  - [ ] Unknown integration id returns 404 (JSON body).
- **Complexity**: M
- **Dependencies**: 1.3.1, 1.4.1

### 1.6.2 OAuth start + callback routes
- **Description**: Kick off the PKCE flow and consume the redirect.
- **Files to create**:
  - `src/app/api/integrations/oauth/start/route.ts` — `GET
    ?integrationId=…&scopes=…` returns `{ authUrl }` (client redirects).
  - `src/app/api/integrations/oauth/callback/route.ts` — `GET
    ?code=…&state=…` finalises and returns a small HTML page (`postMessage`
    to opener, then "you may close this window").
- **Acceptance criteria**:
  - [ ] `start` calls `OAuthManager.startFlow(...)` and returns the auth URL.
  - [ ] `callback` calls `handleCallback(...)`; on error renders a red
        error page with a machine-readable JSON block.
  - [ ] Both routes are Node runtime, not Edge.
- **Complexity**: M
- **Dependencies**: 1.2.2

### 1.6.3 Client-secret upload + disconnect routes
- **Description**: Accept `client_secrets.json` uploads and support explicit
  disconnect.
- **Files to create**:
  - `src/app/api/integrations/[id]/client-secret/route.ts` — `POST` JSON body
    (raw `client_secrets.json` contents).
  - `src/app/api/integrations/[id]/disconnect/route.ts` — `POST`; clears
    tokens, resets connected=false, preserves per-service config.
- **Acceptance criteria**:
  - [ ] Upload validates the payload has exactly one of `web` / `installed`
        and required fields (`client_id`, `client_secret`, `redirect_uris`,
        `auth_uri`, `token_uri`).
  - [ ] Rejects payloads > 32 KB with 413.
  - [ ] Disconnect deletes `tokens:<id>` from SecretsStore.
  - [ ] Neither route logs secret material.
- **Complexity**: M
- **Dependencies**: 1.1.2, 1.4.1

---

## 1.7 · GSuite Manifest + Client

### 1.7.1 GSuite manifest with Gmail service only
- **Description**: Declare the `gsuite` integration manifest and register it at
  module load. Include only the Gmail service in Phase 1 (Drive/Calendar/Photos
  present in `spec.md` are deferred).
- **Files to create/edit**:
  - `src/lib/integrations/services/gsuite/manifest.ts` — manifest export.
  - `src/lib/integrations/services/gsuite/index.ts` — imports and calls
    `registerIntegration(manifest)`.
  - (edit) `src/lib/integrations/index.ts` — import `services/gsuite` so
    registration runs.
- **Acceptance criteria**:
  - [ ] `listIntegrations()` returns exactly `["gsuite"]` after import.
  - [ ] Manifest exposes `gmail.readonly`, `gmail.modify`, `gmail.send` in
        `supportedScopes`.
  - [ ] `authorizationUrl` / `tokenUrl` point at
        `accounts.google.com/o/oauth2/v2/auth` and `oauth2.googleapis.com/token`.
- **Complexity**: S
- **Dependencies**: 1.3.1

### 1.7.2 GSuite shared HTTP client + `client_secrets.json` normaliser
- **Description**: A thin fetch wrapper that pulls client credentials from
  SecretsStore, uses `authedFetch`, and centralises Gmail base URL + retry.
- **Files to create**:
  - `src/lib/integrations/services/gsuite/client.ts` — `gsuiteFetch(url,
    init)`.
  - `src/lib/integrations/services/gsuite/client-secrets.ts` — normalise the
    two shapes (`{ web: {...} }` vs `{ installed: {...} }`) into a single
    `{ clientId, clientSecret, redirectUris, authUri, tokenUri }` record.
- **Acceptance criteria**:
  - [ ] Prefers `web` shape when both are present.
  - [ ] Rejects a JSON blob missing required fields with a
        structured `IntegrationConfigError`.
  - [ ] `gsuiteFetch` applies exponential backoff (max 3 attempts) on 429/5xx.
- **Complexity**: M
- **Dependencies**: 1.5.1, 1.6.3, 1.7.1

---

## 1.8 · Gmail Adapter

### 1.8.1 GmailAdapter class skeleton + `listMessages`
- **Description**: Create `GmailAdapter extends ServiceAdapter`. Implement
  `listMessages({ query?, labelIds?, maxResults?, pageToken? })` calling
  `GET /gmail/v1/users/me/messages`. Guard with `withScope("gmail.readonly")`.
- **Files to create**:
  - `src/lib/integrations/services/gsuite/adapters/gmail.ts`.
- **Acceptance criteria**:
  - [ ] Returns `{ messages: [{id, threadId}], nextPageToken? }`.
  - [ ] Missing scope ⇒ `IntegrationScopeError`.
  - [ ] Unit-smoke: with a real token, returns at least one message from a
        real Gmail account.
- **Complexity**: M
- **Dependencies**: 1.7.2

### 1.8.2 `getMessage(id, { format })`
- **Description**: Fetch a single message with `format ∈ { full, metadata,
  minimal }`. Scope: `gmail.readonly`.
- **Files to edit**: `.../gmail.ts`.
- **Acceptance criteria**:
  - [ ] Returns the full Gmail message resource.
  - [ ] Supports the `metadataHeaders` query param when format=metadata.
- **Complexity**: S
- **Dependencies**: 1.8.1

### 1.8.3 `sendMessage({ to, subject, body, cc?, bcc?, replyTo?, mimeType? })`
- **Description**: Compose an RFC 2822 message, base64url-encode, POST to
  `users/me/messages/send`. Scope: `gmail.send`.
- **Acceptance criteria**:
  - [ ] Handles UTF-8 headers (RFC 2047 encoded-word).
  - [ ] Defaults `mimeType` to `text/plain`, allows `text/html`.
  - [ ] Returns `{ id, threadId, labelIds }` from the API response.
- **Complexity**: M
- **Dependencies**: 1.8.1

### 1.8.4 `replyToMessage({ messageId, body, mimeType? })`
- **Description**: Build a reply with matching `Subject`, `In-Reply-To`,
  `References`; send via `sendMessage` internals. Scope: `gmail.send`.
- **Acceptance criteria**:
  - [ ] Threading works (response `threadId` == source message `threadId`).
  - [ ] `Re:` is prepended only if not already present.
- **Complexity**: M
- **Dependencies**: 1.8.2, 1.8.3

### 1.8.5 `modifyMessage({ id, addLabelIds?, removeLabelIds? })`
- **Description**: `POST users/me/messages/{id}/modify`. Scope: `gmail.modify`.
- **Acceptance criteria**:
  - [ ] Adding `STARRED` stars the message (visible in Gmail UI).
  - [ ] Removing `UNREAD` marks as read.
- **Complexity**: S
- **Dependencies**: 1.8.1

### 1.8.6 `trashMessage(id)` / `untrashMessage(id)`
- **Description**: Wraps `/messages/{id}/trash` and `/untrash`. Scope:
  `gmail.modify`.
- **Acceptance criteria**:
  - [ ] Trashed message appears in Gmail Trash.
  - [ ] Untrash restores to Inbox (or original labels).
- **Complexity**: S
- **Dependencies**: 1.8.5

### 1.8.7 `searchMessages({ query, maxResults? })`
- **Description**: Thin wrapper around `listMessages` with Gmail's search
  syntax pre-baked (e.g., `from:foo has:attachment newer_than:7d`). Scope:
  `gmail.readonly`.
- **Acceptance criteria**:
  - [ ] Documented example queries in the JSDoc.
  - [ ] Returns the same shape as `listMessages`.
- **Complexity**: S
- **Dependencies**: 1.8.1

### 1.8.8 `listLabels()` / `getLabel(id)`
- **Description**: `/users/me/labels` list + detail. Scope: `gmail.readonly`.
- **Acceptance criteria**:
  - [ ] Returns Gmail label resources including system labels (`INBOX`, etc.).
- **Complexity**: S
- **Dependencies**: 1.8.1

### 1.8.9 `getProfile()`
- **Description**: `/users/me/profile` → `{ emailAddress, messagesTotal,
  threadsTotal, historyId }`. Scope: `gmail.readonly`. Used by the Settings UI
  auth card to render the connected user's email.
- **Acceptance criteria**:
  - [ ] Called from the OAuth callback path to seed the "Connected as
        `user@example.com`" line in the auth card.
- **Complexity**: S
- **Dependencies**: 1.8.1

---

## 1.9 · Settings UI

### 1.9.1 Register Integrations custom tab
- **Description**: Add the `integrations` namespace to
  `src/lib/config/registry.ts` with `customComponent: "integrations"` and no
  form fields. Wire `IntegrationsTab` into `CUSTOM_TABS` in
  `src/apps/settings/index.tsx`.
- **Files to create/edit**:
  - (create) `src/components/apps/settings/IntegrationsTab.tsx` — empty
    scaffold rendering "Integrations".
  - (edit) `src/lib/config/registry.ts`.
  - (edit) `src/apps/settings/index.tsx`.
- **Acceptance criteria**:
  - [ ] The tab appears in Settings and renders the scaffold.
  - [ ] Ordering places it after `apps` and before `appearance`.
- **Complexity**: S
- **Dependencies**: —

### 1.9.2 List view (all integrations)
- **Description**: Render the master list per `mockup-drilldown.html §page-list`:
  status dot + name + service summary + drill-in arrow. Data via
  `useIntegrations()` hook hitting `/api/integrations`.
- **Files to create**:
  - `src/components/apps/settings/integrations/useIntegrations.ts`.
  - `src/components/apps/settings/integrations/IntegrationListView.tsx`
    (or export from `IntegrationsTab.tsx`).
- **Acceptance criteria**:
  - [ ] Colours & spacing match the mockup opacity scale.
  - [ ] Status logic: connected+all-services-enabled=green,
        connected+some-disabled/errored=amber, disconnected=grey.
  - [ ] Click drills into detail.
- **Complexity**: M
- **Dependencies**: 1.6.1, 1.9.1

### 1.9.3 Detail view (per integration)
- **Description**: Render the detail per `mockup-drilldown.html §page-detail`:
  auth card (avatar/email/status + Test/Reauthorize/Disconnect buttons),
  services list. Includes `ClientSecretUpload` when the integration has no
  client credentials stored yet.
- **Files to create**:
  - `src/components/apps/settings/integrations/IntegrationDetailView.tsx`.
  - `src/components/apps/settings/integrations/ClientSecretUpload.tsx`.
- **Acceptance criteria**:
  - [ ] Auth card shows the email from `getProfile()` when connected.
  - [ ] `Connect` button opens the OAuth start URL in a popup and refreshes
        state on `postMessage` from the callback.
  - [ ] `Disconnect` confirms then calls the disconnect route.
  - [ ] Upload accepts drag-drop and file picker; shows an inline error for
        malformed files.
- **Complexity**: L
- **Dependencies**: 1.6.2, 1.6.3, 1.8.9, 1.9.2

### 1.9.4 Service configuration view (Gmail)
- **Description**: Render the config page per `mockup-drilldown.html
  §page-config`: scope toggles, polling section (fields disabled + tagged
  "Phase 2"), webhook section (also disabled + tagged "Phase 2"). Save
  patches state via `/api/integrations/[id]`.
- **Files to create**:
  - `src/components/apps/settings/integrations/ServiceConfigView.tsx`.
  - `src/components/apps/settings/integrations/ScopeToggle.tsx`.
- **Acceptance criteria**:
  - [ ] `ScopeToggle` renders three states (granted-enabled, granted-disabled,
        not-granted-locked) per `spec.md §Scope Override UI Logic`.
  - [ ] Save triggers a `PATCH` and refetches state; unsaved changes prompt
        on Cancel.
  - [ ] Polling & webhook sections visibly disabled + labelled "Coming in
        Phase 2".
- **Complexity**: L
- **Dependencies**: 1.6.1, 1.9.3

### 1.9.5 Breadcrumb navigation
- **Description**: Add the breadcrumb component that switches between list /
  detail / config views, matching the mockup.
- **Files to create/edit**:
  - `src/components/apps/settings/integrations/IntegrationsBreadcrumb.tsx`.
  - (edit) `IntegrationsTab.tsx` to host the view-state + breadcrumb.
- **Acceptance criteria**:
  - [ ] Clicking `Integrations` from any depth returns to the list view.
  - [ ] Breadcrumb reflects the current integration/service names.
  - [ ] Keyboard navigation (Enter/Space) works on breadcrumb items.
- **Complexity**: S
- **Dependencies**: 1.9.2, 1.9.3, 1.9.4

---

## 1.10 · Tool Dispatcher Helper — **CRITICAL**

### 1.10.1 `registerAdapterActions()` helper
- **Description**: Central helper that walks an adapter's exposed method list
  and registers each as a CopilotKit `useCopilotAction`. Applies:
  - **Naming**: `integrations_{integrationId}_{serviceId}_{methodName}` —
    method segment stays camelCase (see `plan.md D5`).
  - **Gating**: `available` flag driven by the current effective scope set
    for the required scope of that method.
  - **Server-side scope check**: adapter method already gated by `withScope`;
    the action wrapper surfaces `IntegrationScopeError` as a structured
    tool result the LLM can reason about.
  - **Schema**: derived from a per-method Zod schema declared alongside the
    method (adapter authors declare `.schemas`).
- **Files to create**:
  - `src/lib/integrations/actions/dispatcher.ts`.
- **Acceptance criteria**:
  - [ ] Given `GmailAdapter`, one action is registered per method.
  - [ ] Toggling a scope in Settings flips the `available` flag on the
        relevant actions within one React render cycle.
  - [ ] `IntegrationScopeError` returned by an action becomes
        `{ ok: false, code: "scope_disabled", scope }` in the tool result.
  - [ ] Documented naming exception is linked from
        `src/lib/agent/capabilities-registry.ts` comment.
- **Complexity**: L
- **Dependencies**: 1.5.1, 1.8.1 – 1.8.9

---

## 1.11 · Assistant Tool Integration — **CRITICAL**

### 1.11.1 Add `Integrations` capability group + Gmail entries
- **Description**: Append Gmail action entries to
  `src/lib/agent/capabilities-registry.ts` under an `Integrations` group.
  Each entry uses `context: "action"` (client-only in Phase 1) and follows the
  documented naming exception.
- **Files to edit**:
  - `src/lib/agent/capabilities-registry.ts` — add group + 9 entries
    (`integrations_gsuite_gmail_listMessages`, `getMessage`, `sendMessage`,
    `replyToMessage`, `modifyMessage`, `trashMessage`, `untrashMessage`,
    `searchMessages`, `listLabels`, `getLabel`, `getProfile`).
  - Comment at the top of the section calling out the naming exception.
- **Acceptance criteria**:
  - [ ] `actionCapabilities()` includes the new entries.
  - [ ] `tool-manifest.ts` picks them up automatically (no code change).
  - [ ] Explanation comment links to `plan.md §Technical Decisions D5`.
- **Complexity**: S
- **Dependencies**: 1.10.1

### 1.11.2 Mount `IntegrationActions` in CopilotProvider
- **Description**: New client component that calls
  `registerAdapterActions(GmailAdapter)` and any other Phase 1 adapters. Mount
  in the aggregator that hosts `*Actions.tsx` (e.g., `CopilotProvider.tsx`
  or its children).
- **Files to create/edit**:
  - (create) `src/components/agent/IntegrationActions.tsx`.
  - (edit) `src/components/agent/CopilotProvider.tsx` (or the parent that
    mounts `OSActions`, `FsActions`, etc.).
- **Acceptance criteria**:
  - [ ] Tool list in the Assistant sidebar shows the new Gmail actions.
  - [ ] Component unmounts cleanly (no orphan action registrations on
        hot-reload).
- **Complexity**: M
- **Dependencies**: 1.10.1, 1.11.1

### 1.11.3 Scope-aware `available` wiring
- **Description**: Subscribe `IntegrationActions` to a hook (`useIntegration
  Effective Scopes(integrationId)`) that returns the effective scope set.
  Feed into each `useCopilotAction({ available })`.
- **Files to create/edit**:
  - `src/components/apps/settings/integrations/useIntegrations.ts` — add
    hook variant that also returns effective scopes for consumers.
  - `src/lib/integrations/actions/dispatcher.ts` — accept a scope-set
    source.
- **Acceptance criteria**:
  - [ ] Turning off `gmail.send` in Settings removes the send actions from
        the LLM's tool list live (verified via a manual test).
  - [ ] Turning it back on re-enables them without a page reload.
- **Complexity**: M
- **Dependencies**: 1.11.2, 1.9.4

### 1.11.4 Update assistant instructions (CORE_POLICY)
- **Description**: Append one bullet to `CORE_POLICY` in
  `src/lib/agent/config.ts` explaining how to react to
  `IntegrationScopeError` and pointing at Settings → Integrations.
- **Files to edit**: `src/lib/agent/config.ts`.
- **Acceptance criteria**:
  - [ ] Bullet is one sentence, uses the exact tool-name prefix
        (`integrations_*_*`).
  - [ ] Manual: ask the LLM to send an email with `gmail.send` disabled — it
        surfaces the scope name and the Settings path.
- **Complexity**: S
- **Dependencies**: 1.11.3

### 1.11.5 Integration Actions smoke test in Assistant chat
- **Description**: End-to-end validation of every Gmail action from a live
  Assistant chat: list messages, get message, send email, reply, star,
  trash, search, list labels, get profile.
- **Files to edit**:
  - (docs stub) `docs/dev/integrations.md` — record the smoke-test script.
- **Acceptance criteria**:
  - [ ] All nine actions invoked successfully from chat against a real Gmail
        account.
  - [ ] Screenshots or transcripts attached to the tracking issue.
- **Complexity**: M
- **Dependencies**: 1.11.4

---

## 1.12 · Notifications

### 1.12.1 Notifications module (emit + badge counter)
- **Description**: Small module that (a) writes a JSON file per event to
  `data/vfs/Inbox/`, using the VFS write helpers so it appears in the file
  browser, and (b) increments an in-memory badge counter emitted to
  subscribers.
- **Files to create**:
  - `src/lib/integrations/notifications/emit.ts` —
    `emitNotification(event: IntegrationEvent)`.
  - `src/lib/integrations/notifications/badge.ts` — badge counter +
    subscribe/unsubscribe.
- **Acceptance criteria**:
  - [ ] Emit writes exactly one file per event, filename
        `<integrationId>-<serviceId>-<timestamp>-<shortId>.json`.
  - [ ] Badge counter goes up on emit and can be decremented externally.
  - [ ] Server-only for emit; badge is a framework-free (client-safe) module
        that a React hook can subscribe to.
- **Complexity**: M
- **Dependencies**: 1.0.2

### 1.12.2 Dock badge integration
- **Description**: Render the badge dot / count on a Dock icon.
- **Files to edit**:
  - `src/components/desktop/Dock.tsx`.
  - (create) `src/components/desktop/InboxBadge.tsx` — small reusable badge
    component.
- **Acceptance criteria**:
  - [ ] Dot shows when count > 0, count text shows when count > 1.
  - [ ] Placement decided in code review (Assistant vs Settings vs new Inbox
        icon) — document the choice.
- **Complexity**: S
- **Dependencies**: 1.12.1

### 1.12.3 `pollOnce()` manual trigger + wiring
- **Description**: Add a manual "poll now" button + API route that runs a
  single Gmail poll (using Gmail's history API when possible, else a
  last-24-hours query) and calls `emitNotification` for each new message.
  This is Phase 1's placeholder for the Phase 2 scheduler.
- **Files to create**:
  - `src/app/api/integrations/[id]/services/[serviceId]/poll/route.ts` —
    `POST` (server-only).
  - Adapter method: `GmailAdapter.pollOnce({ since?: number })`.
  - UI hook to the button in `ServiceConfigView`.
- **Acceptance criteria**:
  - [ ] Sending yourself an email → clicking "Poll now" → a file appears in
        `data/vfs/Inbox/` within a few seconds.
  - [ ] Badge counter increments by the number of new messages.
  - [ ] Duplicate polls do not double-emit (dedupe by Gmail message id).
- **Complexity**: L
- **Dependencies**: 1.8.1, 1.8.2, 1.12.1

### 1.12.4 "Mark inbox read" API
- **Description**: Small endpoint the client calls when the user opens the
  Inbox folder or clicks the badge, resetting the counter.
- **Files to create**:
  - `src/app/api/integrations/notifications/mark-read/route.ts` — `POST`.
  - UI: click on badge / opening `data/vfs/Inbox` in Files app calls it.
- **Acceptance criteria**:
  - [ ] Badge decrements to zero on call.
  - [ ] Idempotent when called twice.
- **Complexity**: S
- **Dependencies**: 1.12.1, 1.12.2

---

## 1.13 · Phase 1 Verification

### 1.13.1 End-to-end walkthrough script
- **Description**: Write a scripted walkthrough covering: upload
  `client_secrets.json` → connect → send an email via chat → disable
  `gmail.send` → verify tool disappears → re-enable → poll → verify
  notification.
- **Files to create/edit**:
  - `docs/dev/integrations.md` — add the walkthrough at the top.
- **Acceptance criteria**:
  - [ ] Followed step-by-step on a clean checkout by someone who did not
        write the code — no missing steps.
  - [ ] Every step has a screenshot or command block.
- **Complexity**: M
- **Dependencies**: 1.11.5, 1.12.4

### 1.13.2 Typecheck, lint, and manifest checklist
- **Description**: Final gate before Phase 1 sign-off.
- **Acceptance criteria**:
  - [ ] `npx tsc --noEmit` — clean.
  - [ ] `npm run lint` — clean.
  - [ ] `plan.md` DoD list re-reviewed and every sub-phase 1.a–1.h is done.
  - [ ] `data/integrations/` and `data/.integrations-key` are covered by
        `.gitignore` (verified with `git check-ignore`).
  - [ ] No `console.log` of tokens or secrets in the codebase (grep clean).
- **Complexity**: S
- **Dependencies**: all previous.

---

## Task summary

| Section | Tasks | Cumulative |
|---|---|---|
| 1.0 Setup & shared types           |  2 |  2 |
| 1.1 SecretsStore                   |  2 |  4 |
| 1.2 OAuth Manager                  |  3 |  7 |
| 1.3 Registry                       |  1 |  8 |
| 1.4 State store                    |  1 |  9 |
| 1.5 Adapter base                   |  1 | 10 |
| 1.6 API routes                     |  3 | 13 |
| 1.7 GSuite manifest + client       |  2 | 15 |
| 1.8 Gmail adapter                  |  9 | 24 |
| 1.9 Settings UI                    |  5 | 29 |
| 1.10 Tool dispatcher (CRITICAL)    |  1 | 30 |
| 1.11 Assistant tool wiring (CRIT.) |  5 | 35 |
| 1.12 Notifications                 |  4 | 39 |
| 1.13 Verification                  |  2 | **41** |

---

## Do not start any task until `plan.md` is explicitly approved by the user.
