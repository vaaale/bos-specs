# Integrations Framework — Phase 1 Implementation Plan

> **Status**: DRAFT — awaiting user approval before implementation begins.
> **Scope**: Phase 1 only (core framework + GSuite/Gmail + assistant tools + notifications).
> **Companion**: [`tasks.md`](./tasks.md) breaks this into 41 executable tasks.
> **Source spec**: [`spec.md`](./spec.md) — this plan resolves ambiguities and locks
> decisions but does not replace the spec.

---

## 1. Scope

### 1.1 In scope (Phase 1)
1. **SecretsStore** — encrypted at rest, AES-256-GCM, keyfile-only in Phase 1.
2. **OAuth Manager** — PKCE loopback flow, token refresh, per-integration client.
3. **Integration Registry** — static manifest registration at module load.
4. **Integration State Store** — per-integration `state.json` under `data/integrations/`.
5. **Service Adapter base** — TypeScript interface + shared helpers.
6. **API routes** — `/api/integrations`, `/api/integrations/oauth/*`.
7. **GSuite manifest + shared OAuth client** — provider config, token URLs, scope catalog.
8. **Gmail adapter** — full CRUD-ish surface: list, get, send, reply, modify labels, trash,
   search, plus a `client_secrets.json` upload path for BYO OAuth app credentials.
9. **Settings UI** — Integrations tab with master-detail-drilldown navigation
   (list → integration → service) matching `mockup-drilldown.html`.
10. **Assistant tool integration** — every adapter method exposed as a CopilotKit
    action, gated by the effective granted+enabled scopes.
11. **Notifications** — taskbar badge on the Assistant/Integrations icon plus a
    file-based inbox at `data/vfs/Inbox/` for adapter-emitted events.

### 1.2 Explicitly out of scope for Phase 1
- **Passphrase-based encryption** for SecretsStore (keyfile only in Phase 1).
- **Polling scheduler** (Phase 2) — Gmail polling wiring; polling *config* is
  captured but no scheduler job is registered.
- **Webhook Manager** (Phase 2) — UI shows the section as "coming soon"; no
  endpoint is registered.
- **Drive, Calendar, Contacts, Photos adapters** (Phase 3).
- **Drive VFS mount** — deferred; a Drive file browser is not the same as a VFS
  mount, and Phase 1 ships neither.
- **Additional integrations** (Telegram, Slack, …) — Phase 4+.
- **Docs**: developer / user guides beyond a stub `docs/dev/integrations.md`.

---

## 2. Architecture Analysis (seven core components)

### 2.1 SecretsStore  (`src/lib/integrations/secrets/`)
- **File**: `data/integrations/secrets.json` (encrypted blob, chmod 600).
- **Key material**: `data/.integrations-key` (32 random bytes, chmod 600,
  auto-created on first write). Rotating the key file invalidates all secrets.
- **Crypto**: `node:crypto` AES-256-GCM; per-secret random IV; auth tag stored
  alongside ciphertext. No third-party crypto dep.
- **Concurrency**: single-writer via `atomic-write` (reuses `src/os/atomic-write.ts`)
  — read-modify-write serialised through an in-process `p-queue`-style lock.
- **API** (server-only): `set / get / delete / listKeys` scoped by
  `integrationId`.
- **Not stored here**: `client_secrets.json` blobs — those are user-supplied OAuth
  *client* credentials and are stored as-is (encrypted) under a reserved key
  namespace `oauth_client:<integrationId>`.

### 2.2 OAuth Manager  (`src/lib/integrations/oauth/`)
- **Flow**: authorisation-code + PKCE (RFC 7636). Loopback redirect URI derived
  from `NEXT_PUBLIC_APP_ORIGIN` (fallback `http://localhost:3000`) plus
  `/api/integrations/oauth/callback`.
- **State**: short-lived, single-use `state` token stored in-memory (server) with
  a 10-min TTL, indexed by verifier hash; survives process life only.
- **Provider config**: pulled from the integration manifest (`oauthConfig`) *plus*
  the user-uploaded `client_secrets.json` (client_id, client_secret).
- **Token refresh**: `getValidToken(integrationId)` refreshes if
  `expires_at - now < 60s`; concurrent callers share a single refresh promise.
- **Persistence**: tokens land in SecretsStore under
  `tokens:<integrationId>`; the non-sensitive metadata (expiry, granted scopes)
  is mirrored into the integration's `state.json`.

### 2.3 Integration Registry  (`src/lib/integrations/registry.ts`)
- Framework-free (no `react`, no `server-only`) so both client and server code
  can read the manifest list.
- Static registration: `manifest.ts` files export their manifest; `registry.ts`
  imports and registers them at module load. **No file-system scanning.**
- API: `listIntegrations()`, `getIntegration(id)`, `getService(intId, svcId)`.

### 2.4 State Store  (`src/lib/integrations/state/`)
- **Files**: `data/integrations/<integrationId>/state.json`, atomic writes.
- **Schema** (see `types.ts`): connection status, per-service `enabled` +
  `config` + `lastSync`, `scopeOverrides`, non-sensitive OAuth metadata.
- **Concurrency**: same read-modify-write lock discipline as SecretsStore.

### 2.5 Service Adapter Base  (`src/lib/integrations/adapters/base.ts`)
- Abstract class `ServiceAdapter` implementing the spec interface, plus helpers:
  - `requireScope(scope)` — throws `IntegrationScopeError` when the user has
    disabled or not been granted the scope.
  - `authedFetch(url, init)` — attaches a fresh access token via OAuth Manager,
    retries once on `401` after a refresh.
- Adapters extend this and implement service-specific methods.

### 2.6 API Routes  (`src/app/api/integrations/…`)
- `GET /api/integrations` — list manifests + state.
- `GET /api/integrations/[id]` — one integration + service configs + granted
  scopes.
- `PATCH /api/integrations/[id]` — update `enabled`, `scopeOverrides`,
  per-service `config`.
- `POST /api/integrations/[id]/client-secret` — accept `client_secrets.json`
  upload (multipart or JSON body).
- `POST /api/integrations/[id]/disconnect` — clear tokens + reset state.
- `GET /api/integrations/oauth/start?integrationId=…` — kick off PKCE flow.
- `GET /api/integrations/oauth/callback` — finalise flow, store tokens.

### 2.7 Settings UI  (`src/components/apps/settings/IntegrationsTab.tsx`)
- Registered as a **custom tab** via `src/lib/config/registry.ts`
  (`customComponent: "integrations"`).
- Wired in `src/apps/settings/index.tsx` `CUSTOM_TABS`.
- Three "pages" managed by local view-state (list → detail → config), mirroring
  `mockup-drilldown.html`. No new router — internal state only.

---

## 3. Technical Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| **D1** | **Keyfile-only encryption in Phase 1** (no user passphrase). | Ships without a UX for entering/unlocking a passphrase and without the risk of losing everything on a forgotten passphrase; passphrase can be added later without a data migration by wrapping the keyfile with a KDF-derived key. |
| **D2** | **BYO OAuth app via `client_secrets.json` upload** rather than a shared BOS OAuth client. | Google's OAuth policy forbids redistributing app credentials with an open-source project; each user creates their own Cloud Console project. Uploading the JSON blob mirrors the file Google's console produces, so users don't have to hand-copy fields. |
| **D3** | **Single upload field handles both `web` and `installed` blocks.** | Google emits one of two shapes depending on client type; the server normalises by preferring `web`, falling back to `installed`. The user just uploads the file. |
| **D4** | **One CopilotKit action per adapter method** (tool-per-method), not a single generic dispatcher. | Gives the LLM discoverable, well-named tools with per-method schemas — matches how existing BOS actions are shaped (see `capabilities-registry.ts`). Trade-off: tool-count grows with each service, mitigated by scope-gating. |
| **D5** | **Tool naming exception: `{integration}_{service}_{methodName}` with camelCase method segment.** | BOS convention is `subsystem_object_verb` snake_case, but the adapter API is object-oriented (`listMessages`, `sendMessage`). Preserving the method name verbatim in the tool id keeps the mapping obvious, avoids a `_` explosion in verbs (`list_messages_batch_get`), and is documented as an explicit deviation in `capabilities-registry.ts`. |
| **D6** | **Scope authorization check happens in the action wrapper**, not inside each adapter method. | One `withScope(scope, handler)` decorator per action means adapter methods stay pure and the same gating error path is used everywhere; also lets `available` in `useCopilotAction` reflect the effective scope list so the LLM won't see disabled tools. |
| **D7** | **Notifications ship in Phase 1** — taskbar badge + `data/vfs/Inbox/` files. | Users have no way to observe events without them; Phase 2 will add polling that produces events, so having a delivery surface ready avoids reshuffling later. Keeps the surface small: no toast system, no notification centre app yet. |
| **D8** | **Drive mount is out of scope** and Drive adapter is out of scope. | Drive-as-VFS is a substantially larger design (mount points, caching, write semantics) that belongs in a separate spec; keeping it out of Phase 1 avoids scope creep. |
| **D9** | **Static manifest registration** (no dynamic discovery). | Manifests are typechecked TS modules — safer and simpler than a JSON registry, and integrations are compiled into BOS rather than user-installable in Phase 1. |
| **D10** | **`data/integrations/` is server-side only**, never exposed via VFS. | Prevents accidental exfiltration of encrypted secrets via VFS listing / read APIs. |

---

## 4. File & Module Layout

```
src/lib/integrations/
├── index.ts                     # Public entry: re-exports registry + types
├── types.ts                     # Framework-free types (Manifest, State, OAuthTokens, …)
├── errors.ts                    # IntegrationScopeError, IntegrationAuthError, …
├── registry.ts                  # Static integration registry
├── secrets/
│   ├── store.ts                 # SecretsStore impl (server-only)
│   ├── crypto.ts                # AES-256-GCM helpers
│   └── keyfile.ts               # keyfile load/create at data/.integrations-key
├── oauth/
│   ├── manager.ts               # OAuthManager impl (server-only)
│   ├── pkce.ts                  # PKCE helpers (verifier/challenge)
│   └── state.ts                 # in-memory state store for pending flows
├── state/
│   └── store.ts                 # per-integration state.json read/write
├── adapters/
│   └── base.ts                  # ServiceAdapter abstract + withScope helper
├── services/
│   └── gsuite/
│       ├── manifest.ts          # GSuite integration manifest + service defs
│       ├── client.ts            # authed googleapis fetch wrapper
│       ├── client-secrets.ts    # normalise uploaded client_secrets.json
│       └── adapters/
│           └── gmail.ts         # GmailAdapter (extends ServiceAdapter)
├── actions/
│   ├── dispatcher.ts            # registerAdapterActions() helper (D6)
│   └── gmail-actions.ts         # wraps GmailAdapter methods as actions
└── notifications/
    ├── emit.ts                  # emitNotification(): writes inbox file + bumps badge
    └── badge.ts                 # in-memory badge counter + subscribers

src/app/api/integrations/
├── route.ts                                  # GET list
├── [id]/route.ts                             # GET / PATCH / (DELETE)
├── [id]/client-secret/route.ts               # POST upload
├── [id]/disconnect/route.ts                  # POST
└── oauth/
    ├── start/route.ts                        # GET
    └── callback/route.ts                     # GET

src/components/apps/settings/
├── IntegrationsTab.tsx                       # top-level custom tab (list view)
└── integrations/
    ├── IntegrationDetailView.tsx             # services + auth panel
    ├── ServiceConfigView.tsx                 # scopes + polling + webhooks (webhooks disabled)
    ├── ScopeToggle.tsx                       # tri-state (granted/enabled/locked)
    ├── ClientSecretUpload.tsx                # drag/drop + validation
    └── useIntegrations.ts                    # fetch + mutation hook

src/components/agent/
└── IntegrationActions.tsx                    # mounts all integration actions (registers dispatcher)

src/os/notifications-dir.ts                   # resolves data/vfs/Inbox path (via VFS helpers)

data/                                         # gitignored
├── .integrations-key                         # 32 random bytes, chmod 600
├── integrations/
│   ├── secrets.json                          # encrypted blob
│   └── gsuite/
│       └── state.json
└── vfs/
    └── Inbox/                                # notification files
```

**Files touched (existing):**
- `src/lib/config/registry.ts` — register `integrations` custom-component tab.
- `src/apps/settings/index.tsx` — add `integrations: IntegrationsTab` to
  `CUSTOM_TABS`.
- `src/lib/agent/capabilities-registry.ts` — add `integrations` capability group
  and document the naming exception.
- `src/lib/agent/tool-manifest.ts` — no code change (derives from
  capabilities-registry) but naming exception documented next to it.
- `src/components/agent/CopilotProvider.tsx` (or equivalent aggregator) — mount
  `<IntegrationActions />`.
- `src/components/desktop/Dock.tsx` — read badge counter from
  `notifications/badge.ts` and render a dot / count on the Assistant (or
  Settings) icon.

---

## 5. Definition of Done — per sub-phase

Each sub-phase corresponds to a `tasks.md` section (1.a → 1.h). Sub-phase is
"done" when *all* of its tasks are checked AND the criteria below hold.

**1.a — Foundations (types, secrets, oauth, registry, state, adapter base)**
- `npx tsc --noEmit` clean, `npm run lint` clean.
- Unit-testable via a scratch script: SecretsStore round-trips a value; OAuth
  Manager builds a valid authorisation URL with PKCE; State Store writes and
  re-reads a state file.
- No routes yet, no UI yet — just server plumbing.

**1.b — API routes**
- `GET /api/integrations` returns the (still empty) registry list.
- OAuth start route returns a redirect URL; callback route accepts a code
  and stores tokens (verified against a mock provider or Google sandbox).
- All routes are `server-only` and return typed JSON.

**1.c — GSuite manifest + client**
- Manifest registers a `gsuite` integration with a `gmail` service (only).
- `client_secrets.json` upload endpoint validates the JSON shape and stores
  the client credentials via SecretsStore.

**1.d — Gmail adapter**
- Every method in `tasks.md §1.8` implemented, hitting real Gmail REST endpoints.
- Manual smoke test: connect a real Google account, list 10 messages, send a
  test email, star a message, trash it, empty trash.

**1.e — Settings UI**
- Master-detail-drilldown navigation matches the mockup visually (dark theme,
  violet accent, breadcrumb).
- Connect / Disconnect / Reauthorize buttons trigger the OAuth flow round-trip.
- `client_secrets.json` upload works via drag-drop or file picker.
- Scope toggles reflect and mutate `state.json`.

**1.f — Tool dispatcher + assistant integration**
- Every Gmail adapter method appears in the Assistant's tool list under an
  `integrations_gsuite_gmail_*` name.
- Toggling `gmail.send` off in Settings immediately removes
  `integrations_gsuite_gmail_sendMessage` from the LLM's available tools
  (verified via `useCopilotAction`'s `available` flag).
- Calling a disabled tool returns `IntegrationScopeError` as a tool result.

**1.g — Notifications**
- Sending yourself an email and calling `pollOnce()` (a manual trigger for now)
  drops a file under `data/vfs/Inbox/gsuite-gmail-<msgId>.json` and increments
  the Dock badge.
- Badge decrements when the Inbox is opened (or on demand via a small "mark
  read" API).

**1.h — Verification**
- End-to-end walkthrough scripted in `docs/dev/integrations.md` (stub).
- All Phase 1 tasks checked in `tasks.md`.
- `npx tsc --noEmit` && `npm run lint` clean.

---

## 6. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **Tool explosion** — one CopilotKit action per method scales with each service. | LLM context bloat; slower tool selection. | Gate via `available` (scope-based); ship only Gmail in Phase 1; add tool grouping/filtering to the Assistant sidebar in Phase 2 if the count becomes unwieldy. |
| **Credential validation** — user uploads a malformed `client_secrets.json`. | Silent failure at OAuth time. | Validate the file server-side on upload: require exactly one of `web`/`installed`, required fields (`client_id`, `client_secret`, `redirect_uris`), and return a structured error. |
| **Concurrent writes** to `state.json` / `secrets.json`. | Lost writes if two requests race. | In-process lock (single Node process) around read-modify-write; use `atomic-write`; the lock is per-file. |
| **OAuth callback race with UI navigation.** | User closes the settings tab mid-flow. | State entries expire in 10 min; the callback endpoint renders a small "You can close this window" page and posts back to the opener via `postMessage`, but does not depend on it — reopening Settings shows the up-to-date state. |
| **Losing the keyfile** wipes all stored secrets. | User must re-authenticate every integration. | Document in `docs/dev/integrations.md`; back-up recommendation. In Phase 1 the failure mode is "reconnect", not "data loss" of anything else. |
| **Scope-disabled tool called by the LLM anyway** (client-side gate bypassed). | Confusing error surface. | Server-side scope check inside the action wrapper too (defence in depth); return a structured `IntegrationScopeError` the LLM can reason about. |
| **`client_secrets.json` accidentally committed** if the user drops it into the repo. | Credential leak. | Upload-only path is `/api/integrations/[id]/client-secret`; the file never touches the repo tree. `data/` is gitignored. |
| **Google's `installed` client redirect** requires `http://127.0.0.1` — not `localhost`. | OAuth fails with an obscure error. | Normaliser prefers `web` credentials; when only `installed` is available, redirect URI is derived from the file's own `redirect_uris` list, not fabricated. |
| **Notification inbox unbounded growth.** | VFS clutter over time. | Phase 1: no auto-cleanup, documented cap of ~1000 entries with oldest-first eviction if exceeded. |
| **Dock badge coupled to Assistant icon** conflates two UX signals. | Users think the assistant has messages. | Badge is attached to whichever icon we pick — likely the Settings icon (Integrations tab) or a new "Inbox" icon; final placement decided during 1.g. |

---

## 7. Integration Points

- **CopilotKit actions** — `IntegrationActions.tsx` mounts under
  `CopilotProvider` alongside the existing `*Actions.tsx` components; each
  Gmail method registers via `useCopilotAction` with an `available` flag driven
  by the current scope set. **Server-side sub-agent tools are NOT part of
  Phase 1** — Phase 1 exposes actions to the main chat only. When Phase 2
  wants to expose these to sub-agents, add matching `context: "both"` capabilities.

- **`capabilities-registry.ts`** — a new `Integrations` group. Each Gmail action
  gets one entry. This is where the *naming exception* (D5) is documented so
  future readers see it in the canonical registry.

- **VFS (`src/os/vfs.ts`)** — the notifications module writes files via VFS
  helpers so paths and atomic-write behaviour match user-facing files;
  `data/integrations/` bypasses VFS on purpose (server-only, never listed).

- **Config registry (`src/lib/config/registry.ts`)** — a new registration
  block declaring the `integrations` namespace with `customComponent:
  "integrations"`. `load`/`save` are no-ops (state lives in per-integration
  files, not the config store).

- **Dock (`src/components/desktop/Dock.tsx`)** — subscribes to
  `notifications/badge.ts` (a small event emitter + counter) and renders a
  badge dot / count.

- **Docs (`docs/dev/`)** — one new stub, `docs/dev/integrations.md`, covering:
  how to add a new integration, keyfile lifecycle, `client_secrets.json` setup
  for Google, tool naming exception.

- **Assistant instructions (`src/lib/agent/config.ts`, CORE_POLICY)** — one new
  bullet: "When using `integrations_*_*` tools, the user has already granted
  the scope; if you get an `IntegrationScopeError`, tell the user which scope
  is disabled and how to re-enable it in Settings → Integrations."

---

## 8. Non-goals / anti-patterns to avoid

- Do **not** introduce a generic "plugin sandbox" for integrations — they are
  compiled into BOS in Phase 1.
- Do **not** proxy adapter methods through a single "run tool" action —
  each method is its own action (D4).
- Do **not** couple SecretsStore to `data/vfs/` — it lives under
  `data/integrations/` on purpose (D10).
- Do **not** ship polling in Phase 1 — the polling config is captured in
  `state.json` but no scheduler job is registered.
- Do **not** implement webhooks in Phase 1 — the UI section exists but is
  disabled + labelled "Phase 2".

---

## 9. Approval checkpoint

Do not start implementation of any task in `tasks.md` until this plan is
explicitly approved. Reviewers, please confirm:

1. Scope split between Phase 1 and later phases is correct.
2. Ten technical decisions above are acceptable (esp. D1 keyfile-only, D2 BYO
   OAuth, D4 tool-per-method, D5 naming exception, D8 no Drive).
3. File layout doesn't collide with anything in flight.
4. Notifications surface (badge + inbox files) is the right minimum for Phase 1.
