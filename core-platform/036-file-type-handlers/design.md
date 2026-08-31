# Design: File-Type Handler Registry & "Open With" (036)

> Companion to `spec.md`. This is the architecture artifact — `plan.md` (written later)
> references it rather than repeating it. Every claim below is grounded in a file or doc
> read this session; where I'm inferring rather than citing, I say so.

## 1. Classification

**App Target: `bos-core`.** This agrees with `spec.md`'s `App Target` field — no
disagreement to reconcile.

Rationale against the Build Studio taxonomy (all three target refs read):

- The change is **entirely under `src/`** and none of it is a self-contained app folder
  in the sense of "add one new `src/apps/<id>/`": it extends an OS primitive
  (`AppManifest` in `src/os/types.ts`), adds a new server subsystem (`src/lib/file-handlers/`),
  a new API route (`src/app/api/file-handlers/`), and modifies two EXISTING built-in apps
  (`html-viewer`'s manifest, the `files` app's UI). That is the `bos-core` shape — OS
  primitives + API routes + server stores + modifications across existing built-in apps —
  not the `builtin-app` shape (which is "create one new `src/apps/<id>/` folder").
- It is **not** a `marketplace-item`: nothing here lives under `data/user-apps/`, nothing is
  installable/removable as an item, and there is no service facet (no worker thread, no own
  port). The follow-on `agentic-text-editor` registration is explicitly a separate
  `marketplace-item` spec and is out of scope per A-7.

**Facets:** single — `bos-core` (registry + manifest type + launch contract + Files-app
consumer + `html-viewer` declaration). No app/service facet split to reason about.

---

## 2. Constitution check

| Principle | Status |
|---|---|
| **I. Spec-Driven** | ✅ This design IS the spec's architecture artifact; implementation follows plan/tasks. |
| **II. Server Authority & SSR Boundary** | ✅ The **registry** (`handlersFor`/`effectiveSelected`), **selection persistence**, and the **install-set query** are all `server-only` (`src/lib/file-handlers/`, `src/app/api/file-handlers/`). The **pure** param/MIME helpers — `baseMime`, `buildLaunchParams`, `rawUrlFor`, and the authoritative matching map — are **framework-free and client-safe** (`src/os/file-handlers.ts`), so the client Files app can assemble `launch` params without importing a `server-only` module: the OS `launch` is the client store, so the caller that builds params is always client code (ADR-7 — this is the explicit SSR-boundary decision). The Files app (client) only `fetch`es for the handler list. Framework-free types (`AppFileHandlerDeclaration`) live in `src/os/types.ts` per the existing convention. No secrets. |
| **III. Always Delegate** | ✅ Implementation goes through the Developer on a feature branch (mechanics in `target-bos-core.md`). |
| **IV. Minimize Blast Radius** | ✅ Feature branch. The registry is **stateless/derived** (recomputed from the current installed set on each request, ADR-1) — no new mutable global to get stale, no preview/base divergence to manage. The Files-app change is strictly additive; the no-handler path is byte-for-byte the existing behavior (SC-003). |
| **V. VFS Is Not the Source** | ✅ Selection state persists under `data/system/` (runtime state), not the VFS and not `src/`. |
| **VI. Specs & Docs Stay in Sync** | ⚠️ Implementation MUST update `docs/dev/apps/built-in-apps.md` (the `fileHandlers` manifest field), a short `docs/dev/apps/file-handlers.md` (the open-file launch contract), and the `extending-bos.md` "Add a file handler" recipe. This is the Developer's job per the delegation brief — flagged here so it isn't dropped. |
| **VII. Respect Boundaries** | ✅ No changes to secrets, `package.json`, lockfiles, or build config. `npx tsc --noEmit` + `npm run lint` are the quality gates. |

**No conflicts.** One thing to keep honest (not a violation): FR-008's "the registry is
boot-time, in-memory" (A-8) is satisfied in a *stronger* form than a literal long-lived
in-memory map would be — the registry is a pure derivation from the current installed set,
recomputed per request (ADR-1). "Reflects only currently-installed apps" (FR-002) then
holds by construction, with no re-collection hook to forget.

---

## 3. Architecture

### 3.1 Context

The user's mental model: "I have files in the Files app; clicking one opens it in the app
that's chosen for its type, and I can pick a different app from right-click." This feature
adds the platform layer that makes "which app opens this type" a **queryable, user-overridable
fact** instead of a hardcode in the Files app, and registers `html-viewer` (web_view) as the
default `text/html` renderer so the headline behavior falls out of the general mechanism.

Actors: the **Files app** (the first and, in this spec, only consumer), **handler apps**
(built-in components or installed iframes that declare `fileHandlers`), and the **OS**
(manifests, `launch`, the new registry + selection store + API).

### 3.2 Container

Real BOS containers touched (no new containers invented):

- **Next.js app process** — hosts the registry (`src/lib/file-handlers/`), the new API
  route, the modified built-in apps, and the client Files app. Everything in this feature
  runs inside this single process; there is **no worker-thread service and no own port**
  (contrast `target-marketplace-item.md` — not applicable here, and the "does this need a
  service facet?" check is answered *no*: file opening is a UI/launch concern, not a
  background/protocol one).
- **Data root (`data/`)** — the selection file `data/system/file-handlers.json` (new).
  Branch isolation (base vs. Supervisor preview) is inherited for free because
  `dataDir()` honors `BOS_DATA_DIR` (`src/os/data-dir.ts`) — no extra work.

```mermaid
flowchart LR
  subgraph Next["Next.js process"]
    FE["Files app (client)\nsrc/apps/files/"]
    API["/api/file-handlers\n(GET list, POST selected)"]
    REG["file-handlers registry\nsrc/lib/file-handlers/registry.ts"]
    SEL["selection store\nsrc/lib/file-handlers/selection.ts"]
    OS["os-store launch(appId, params)\nsrc/store/os-store.ts"]
    WIN["Window.tsx delivery split\nbuilt-in: AppProps.params\niframe: URL query (withFileParams)"]
    HV["html-viewer (built-in)\nreads params.url / params.title"]
    SH["shared client-safe helpers\nsrc/os/file-handlers.ts\nbaseMime · buildLaunchParams · rawUrlFor"]
  end
  FE -->|fetch| API
  API --> REG
  API --> SEL
  REG -->|"built from\nBUILTIN_APPS + listInstalledManifests()"| APPS["manifests\nsrc/os/apps.ts · src/lib/apps/store.ts"]
  SEL -->|atomic JSON| DATA[("data/system/file-handlers.json")]
  FE -->|"buildLaunchParams · baseMime"| SH
  FE -->|launch(appId, {path,action,[url],[title]})| OS
  OS --> WIN
  WIN --> HV
```

### 3.3 Component

The concrete pieces, mapped to the spec's requirements:

- **`AppManifest.fileHandlers`** (`src/os/types.ts`) — the declarative extension point
  (FR-001, FR-013). Mirrors `AppManifest.eventHandlers` (spec 034, read
  `src/os/types.ts`). Each entry: MIME type (exact or `type/` prefix), `capabilities`
  (`render`/`edit`), optional `label`, optional `default`, optional `paramShape`.
- **Shared client-safe helpers** (`src/os/file-handlers.ts`, new, **framework-free** — no React, no Node, no `server-only`; the same shape as `src/os/types.ts`) — the pure, environment-agnostic string logic the *client* needs to drive the contract: `baseMime` (parameter stripping + lowercase), the authoritative **matching** MIME map + `mimeForPath`/`fileBaseMime` (moved here from `src/lib/mime.ts`, ADR-8), `rawUrlFor(path)` (the file-bytes URL, identical string to `fsClient.rawUrl(path, undefined)`), and `buildLaunchParams(decl, path, action)` (the `paramShape`→`launch`-params mapping, ADR-3). This is what fixes the M-1 seam: the caller that assembles params is the client Files app, so these must be client-importable — a `"use client"` component cannot import a `server-only` module (ADR-7).
- **Registry** (`src/lib/file-handlers/registry.ts`, new, `server-only`) — derives
  `{ handlers, effectiveSelected }` per MIME type from `BUILTIN_APPS`
  (`src/os/apps.ts`) + `listInstalledManifests()` (`src/lib/apps/store.ts`) (FR-002, FR-003).
  Stateless — recomputed per request (ADR-1).
- **Selection store** (`src/lib/file-handlers/selection.ts`, new, `server-only`) — the
  only durable state: `Record<baseMime, appId>`, read/written atomically to
  `data/system/file-handlers.json` (FR-010, FR-011).
- **API route** (`src/app/api/file-handlers/route.ts`, new) — `GET ?mime=` → the registry's
  view for that type; `POST` → set/clear the user selection (FR-003, FR-010).
- **Files app** (`src/apps/files/index.tsx`, modified) — the consumer: double-click
  resolves the selected handler and launches it (FR-006); right-click renders the "Open
  with" group from `GET /api/file-handlers` (FR-008, FR-010, FR-012). No-handler path is
  unchanged (FR-007, SC-003).
- **`html-viewer` manifest** (`src/apps/html-viewer/manifest.ts`, modified) — declares the
  `text/html` render handler (FR-005). **No code change** to `html-viewer/index.tsx`: it
  already reads `params.url`/`params.title` (read the file), and the OS hands it exactly
  those via the launch contract.
- **Launch delivery** (`src/components/apps/IframeApp.tsx`, modified) — a `withFileParams`
  helper, the file-opening counterpart to the existing `withEventParams` (034), that
  carries the contract's file params into an installed iframe app's URL (FR-014).

---

## 4. Concrete file / module plan

**Only real paths this feature creates or modifies.** (Existing BOS source the feature
merely *calls into* — `os-store.launch`, `listInstalledManifests`, `fsClient.rawUrl`, the
`/api/fs/raw` route — is listed in §5 Integration points, not here.)

| Path | New/Mod | What |
|---|---|---|
| `src/os/types.ts` | mod | Add `AppFileHandlerDeclaration` type + `AppManifest.fileHandlers?` field (mirrors `eventHandlers?` / `AppEventHandlerDeclaration`). |
| `src/os/file-handlers.ts` | new | **Framework-free, client-safe** (no React / Node / `server-only` — the same convention as `src/os/types.ts`). The shared home for the pure helpers the client must import (M-1/S-1): `baseMime(mime)` (strip `; param`, lowercase); the authoritative **matching** MIME map + `mimeForPath(p)` + `fileBaseMime(p)` (`mimeForPath` moved here from `src/lib/mime.ts`, ADR-8); `rawUrlFor(path)` = `"/api/fs/raw?path=" + encodeURIComponent(path)` (identical to `fsClient.rawUrl(path, undefined)`); and `buildLaunchParams(decl, path, action)` (the `paramShape`→`launch`-params mapping, ADR-3). |
| `src/lib/mime.ts` | mod | The MIME map + `mimeForPath` move to `src/os/file-handlers.ts`; this file keeps `baseMime` and `mimeForPath` as **re-exports from there** so the three existing server-side importers (`/api/services/[id]/config-app`, `/app/apps/[...slug]`, `assistant/tools/server/view-image`) are untouched. It keeps `import path` (a Node builtin, used only for `extname`) and so remains Node-only — the client never imports this file; it uses `src/os/file-handlers.ts` (S-1). |
| `src/lib/file-handlers/registry.ts` | new | `server-only`. `matchDeclared(type, base)` (exact base type or `type/` prefix; `baseMime` + the matching map from the shared module), `handlersFor(baseMime)` → `{ appId, name, icon, label, capabilities, isDefault, selected }[]`, and `effectiveSelected(baseMime)` (user selection if installed+render, else manifest default if installed+render, else none). Does **not** define `buildLaunchParams` — it imports it from `src/os/file-handlers.ts` so client and server assemble params identically (M-1). |
| `src/lib/file-handlers/selection.ts` | new | `server-only`. `readSelection()`, `writeSelection(mime, appId | null)` → `data/system/file-handlers.json` via `writeFileAtomic` (`src/os/atomic-write.ts`) + `dataDir()` (`src/os/data-dir.ts`). |
| `src/app/api/file-handlers/route.ts` | new | `GET ?mime=<type>` → `{ handlers: [...], selected: appId | null }`; `POST { mime, appId }` → set selection, `POST { mime }` (no appId) → clear (revert to manifest default). Thin delegate to registry+selection. |
| `src/apps/html-viewer/manifest.ts` | mod | Add `fileHandlers: [{ type: "text/html", capabilities: ["render"], label: "Web View", default: true, paramShape: { url: "raw", title: "basename" } }]`. No change to `index.tsx`. |
| `src/apps/files/index.tsx` | mod | (a) load the handler view for the menu's file MIME on demand; (b) double-click: resolve `effectiveSelected` and, if present, `launch(selected.appId, buildLaunchParams(decl, path, action))` (from `src/os/file-handlers.ts`) else the existing in-app path; (c) right-click: render the "Open with" group (label = `label ?? name`, icon = the **target app's own `icon`**, checkmark = `selected`), hairline divider, then the existing actions; (d) on pick: `launch` with `action = open|edit`, and if the pick is render-capable, `POST` the selection; (e) **the "Open with" group and the double-click handler resolution are gated on `entry.type === "file"`** — a directory (`entry.type === "dir"`) keeps its existing behavior entirely unchanged (FR-012); the gate sits at the same seam the code already branches on (`openEntry`'s `entry.type === "dir"` check, and the context menu's `"Download as zip"` vs `"Download"` label). |
| `src/lib/apps/store.ts` | mod | Mirror `fileHandlers` through `InstalledApp` (type), `readApp` (read `app.json`), and `toManifest` (emit onto `AppManifest`) — the exact three places `eventHandlers` is handled today (read the file; `readApp`'s `eventHandlers:` line and `toManifest`'s `eventHandlers:` line are the template). |
| `src/lib/os-client.ts` | mod | Add `fileHandlersClient` (`list(mime)`, `setSelected(mime, appId?)`) — thin `fetch` wrappers over `/api/file-handlers`, the same shape as the existing `fsClient`/`settingsClient`. `fsClient.rawUrl` **stays** (it is the conversation-aware variant, still used by the Files app's own image `<img src>`); the shared module's `rawUrlFor` is its no-conversationId form, so a handler's `url` param is built client-side without importing a `server-only` module (M-1). |
| `src/components/apps/IframeApp.tsx` | mod | Add `withFileParams(url, params)` (sibling to `withEventParams`) that encodes the contract's file params — `path`, `action`, and `url`/`title` when present — into `bos*`-prefixed query params; applied to the iframe `src` alongside the existing event-param handling. |

> `gen-apps.mjs` auto-discovers built-in apps (`src/os/apps.ts` header) — no registry edit
> for the `html-viewer`/`files` manifest changes. `html-viewer` and `files` already exist,
> so nothing is discovered anew.

**Deliberately NOT in the plan** (out of scope per A-7): any change to
`data/user-apps/items/agentic-text-editor/` — its `app.json` declaration and its
HTML-preview fix are a separate `marketplace-item` spec that *consumes* this mechanism.

---

## 5. Integration points

Existing BOS mechanisms this design **calls into but does not create or modify**, each
cited to the real route/file:

- **`launch(appId, params)`** — `src/store/os-store.ts` (the `launch:` method). Returns
  `null` if the app isn't in the OS store's `apps` — the Files app treats a `null` return
  as "handler not actually launchable" and falls back to the in-app path (defensive, cheap).
- **Window delivery split** — `src/components/desktop/Window.tsx:287`: `manifest.kind ===
  "iframe"` → `IframeApp` (URL-based, §4 `withFileParams`); else built-in component →
  `createElement(AppComponent, { params: win.params })`. This is *why* the launch contract
  has two delivery paths (ADR-2) — it's a pre-existing OS fact, not something this design
  invents.
- **Built-in app discovery** — `src/os/apps.ts` (`BUILTIN_APPS`, generated by
  `tools/gen-apps.mjs`) — source of built-in handler manifests.
- **Installed app discovery** — `src/lib/apps/store.ts` `listInstalledManifests()` —
  source of installed handler manifests; already returns only `status === "installed"`
  apps (the `!i.broken` / installed filter), which is exactly FR-002's "reflects only
  currently-installed apps."
- **MIME resolution** — `src/lib/mime.ts` `mimeForPath` (and the fuller map in
  `src/app/api/fs/raw/route.ts`) — FR-004. Note `mimeForPath` returns
  `text/html; charset=utf-8`; the registry normalizes via `baseMime` (ADR-4).
- **Raw-file URL** — `src/lib/os-client.ts` `fsClient.rawUrl(path)` → `/api/fs/raw?path=`
  (served by `src/app/api/fs/raw/route.ts`) — how a render handler (html-viewer) actually
  fetches the file's bytes. `html-viewer` already loads this URL in an `<iframe src>`.
- **Iframe launch-param precedent** — `src/components/apps/IframeApp.tsx` `withEventParams`
  (034) — the existing mechanism that hands launch params to an installed iframe app via
  URL query; `withFileParams` copies its shape.
- **Persistence primitives** — `src/os/atomic-write.ts` `writeFileAtomic`,
  `src/os/data-dir.ts` `dataDir()` — the selection store's write path.

---

## 6. The open-file launch contract (FR-014)

**The contract (what the OS hands a handler):** a single, app-agnostic launch that carries,
for every handler app, the file's **VFS `path`** and the requested **`action`**
(`"open"` for the open gesture and for render-capable "Open with" picks; `"edit"` for
edit-capable picks), plus — when the handler's declared `paramShape` asks for them — a
resolved **`url`** and **`title`**. A handler's obligation is entirely "when launched with
a file, read/preview/edit it." Identical *intent* for built-in and iframe apps.

**The mapping (contract → actual params) is handler-specific and declared in the
manifest, not hardcoded** (this is how the spec's "the handler app knows what params it
needs" is honored without violating FR-013's "no per-app code in core"):

- `paramShape.url = "raw"` → OS sets `url = rawUrl(path)` (the file's bytes). This is
  html-viewer's case — it wants to render the document, so it gets the raw-file URL.
- `paramShape.url = "app"` → OS sets `url = <handler's own app URL>` and the file is
  delivered as `path`/`action` query params the app fetches via the BOS SDK. This is the
  marketplace-editor case (follow-on).
- `paramShape` absent / `url` omitted → OS sets only `path` + `action` (a built-in editor
  that reads `path` directly).
- `paramShape.title = "basename"` → OS sets `title = basename(path)`.

The **delivery** differs by app kind (a pre-existing OS fact, ADR-2), but the OS passes the
same set of fields either way; the handler reads only the ones it declared. `html-viewer`
already reads `params.url` + `params.title` (no code change); an installed handler reads
the `bos*` query params it asked for.

---

## 7. ADRs

### ADR-1 — The registry is a per-request derivation, not a long-lived in-memory map

- **Context.** FR-002 requires the registry to "reflect only currently-installed apps
  (uninstall invalidates)." The `eventHandlers` pattern (spec 034) is a boot-time
  `globalThis` registry (read `src/lib/events/register-ui-handlers.ts`), which implies a
  persistent in-memory map plus re-collection on install/uninstall.
- **Options.**
  (a) A long-lived in-memory map populated at boot (`instrumentation.ts`) and rebuilt on
  `installItem`/`uninstallApp`/`purgeApp` — faithful to the event pattern but adds three
  re-collection hook sites and a stale-state failure mode.
  (b) A **stateless derivation**: `GET /api/file-handlers` recomputes from
  `BUILTIN_APPS` + `listInstalledManifests()` each request.
- **Decision.** (b). Handler sets are tiny (a handful of apps); recomputing per menu-open
  is negligible. FR-002's "uninstall invalidates" holds *by construction* — there is no
  cached copy to invalidate. "Boot-time collection" (A-8) is satisfied in the sense that
  the registry exists as soon as the server is up and reads the boot-discovered app set.
- **Consequences.** No re-collection wiring; no preview/base registry divergence; the
  `effectiveSelected` logic (which needs the *current* install set) is trivially correct.
  Cost: a few manifest reads per context-menu open — immaterial. If a future consumer needs
  a hot in-memory copy, add it then (YAGNI).

### ADR-2 — "Identical launch contract" = identical OS-side fields, two delivery paths

- **Context.** FR-014 mandates the contract be "identical for built-in (component) and
  installed (iframe) apps." But the OS delivers launch params to the two kinds
  *differently* (a pre-existing fact, not this feature's doing): built-in apps get
  `params` directly (`Window.tsx` → `createElement(AppComponent, { params: win.params })`);
  installed iframe apps are a `<iframe src>` and get launch params **only via URL query**
  (the sole existing precedent is `withEventParams` in `IframeApp.tsx`, 034 — iframes can't
  receive a React props object).
- **Options.**
  (a) Treat "identical" as "same `launch()` call, same fields" — the OS always passes
  `{ path, action, [url], [title] }`; delivery is direct (built-in) vs. query (iframe).
  (b) Invent a new unified channel (e.g. a broker RPC) so iframes receive a props object
  like built-ins do.
- **Decision.** (a). It is the *only* option that matches how BOS actually delivers to
  iframes today (b would be a large new mechanism for one use case, and would not be
  "identical" — it would be a *different, parallel* system). "Identical" is read as
  "the OS hands the handler the same documented set of file fields and the handler's
  obligation is the same," with delivery adapted to the app kind.
- **Consequences.** `IframeApp.tsx` gains `withFileParams` (small, mirrors `withEventParams`).
  Built-in handlers need zero delivery plumbing (they already receive `params`). The
  contract's documented field set is what "identical" refers to; a reviewer who expected
  literally the same *wire format* for both should push back — but the OS's existing
  architecture makes that impossible without (b).

### ADR-3 — The contract→params mapping lives in the manifest (`paramShape`), not in core

- **Context.** The spec states the mapping is "handler-specific (the handler app knows what
  params it needs)" yet FR-013/SC-004 forbid per-app code in core and require that adding a
  handler is manifest-only.
- **Options.**
  (a) Hardcode the mapping for known handlers in the Files app/registry (rejected — violates
  FR-013/SC-004, the very thing the spec exists to avoid).
  (b) A small, **generic, declarative** `paramShape` in each handler's manifest that the
  platform interprets uniformly.
- **Decision.** (b). `paramShape` is a closed vocabulary (`url: "raw"|"app"`, `title:
  "basename"`) — *not* arbitrary code — so the platform's `buildLaunchParams` stays a single
  generic function with zero per-app branches. html-viewer declares
  `paramShape:{url:"raw",title:"basename"}`; a marketplace editor declares
  `paramShape:{url:"app"}` (or omits it and reads `path` via SDK).
- **Consequences.** "No per-app code in core" holds: the *data* is app-specific, the *code*
  is not. Adding a new token later is an additive change to `buildLaunchParams` + the type,
  not a per-app edit. This is the cleanest reading of the spec's "handler-specific
  mapping" that also satisfies FR-013/SC-004.

### ADR-4 — MIME matching normalizes parameters and supports `type/` prefixes

- **Context.** `mimeForPath` returns *parameterized* types (`text/html; charset=utf-8`,
  `src/lib/mime.ts`), while manifests naturally declare bare types (`text/html`). A naive
  string compare would never match.
- **Options.** (a) Compare raw strings (broken). (b) Normalize both sides to a **base media
  type** (drop parameters, lowercase) and match on exact base type OR `type/` prefix
  (FR-001 allows prefixes like `image/`).
- **Decision.** (b). `baseMime()` in `src/lib/mime.ts` does the normalization; `matchDeclared`
  in the registry does exact-base or prefix matching. The selection is keyed by base type
  (`data/system/file-handlers.json` → `{ "text/html": "html-viewer" }`), so a parameterized
  file type and its bare form share one selection.
- **Consequences.** Correct matching for real MIME strings. The client (Files app) must
  normalize identically to send a `mime=` the server understands — `baseMime` is a pure
  string function, so it's safe to share/replicate (it has no Node imports).

### ADR-5 — Selection persists to `data/system/file-handlers.json`, not OSSettings/config

- **Context.** FR-010 needs one durable fact: the per-type user selection. The spec suggests
  "settingsClient.patch or a small JSON file under data/."
- **Options.**
  (a) Add a field to `OSSettings` / `settingsClient.patch` — but `OSSettings`
  (`src/os/types.ts`) is the *appearance/OS* settings blob; file-handler selection is
  domain state, not OS settings, and `OSSettings` changes ripple through the settings UI.
  (b) A new config *namespace* (`src/lib/config/registry.ts`) — heavier (schema, Settings
  tab, auto tool exposure) than this needs; it's not something a user tunes in Settings.
  (c) A **small dedicated JSON file** under `data/system/`, read/written atomically.
- **Decision.** (c). `data/system/file-handlers.json`. `data/system/` is the established
  home for per-app/per-feature runtime state (capabilities live at
  `system/config/<id>/capabilities.json`, `src/lib/apps/store.ts` `setAppCapabilities`), and
  `writeFileAtomic` (`src/os/atomic-write.ts`) gives crash-safe writes. Branch isolation is
  inherited via `dataDir()`/`BOS_DATA_DIR`.
- **Consequences.** Minimal, self-contained, and consistent with how BOS persists small
  per-feature state. No Settings-tab surface is introduced (out of scope — the user changes
  the selection *in the Files context menu*, not in Settings). FR-011's "uninstall
  invalidates" is enforced at read-time: `effectiveSelected` returns the user's choice only
  if that app is *currently installed and render-capable* for the type, else the manifest
  default (if installed), else none.

### ADR-6 — "Open with" action & selection semantics for edit-capable handlers

- **Context.** FR-014 says a pick "MUST launch … *open* for the open gesture and for
  render-capable 'Open with' choices, or *edit* for an edit-capable 'Open with' choice";
  FR-010 says picking a handler "MUST … set it as the new selected handler"; FR-010 also
  says a selected handler "MUST be render-capable." Taken literally these conflict for an
  *edit-only* handler (pick → set as selected, but selected must be render-capable).
- **Decision (the design's reading, flagged as an open question below).**
  - **Render-capable** "Open with" pick → `action="open"` **and** set as the new selected
    (FR-010 "always open with"; the mockup's Option A).
  - **Edit-only** "Open with" pick → `action="edit"`, and it is **not** made the selected
    (double-click) handler, since a selection must be render-capable (FR-010). It remains a
    valid one-shot "open in this editor" choice (FR-009/FR-014 edge "a handler that can
    only edit … is not a valid *selected* handler"). The checkmark stays on the current
    render selection.
  - The Files app distinguishes the two by each entry's `capabilities` (available in the
    `GET` response). This yields exactly one checkmark (the render selection) and a
    well-defined `action` for every pick.
- **Consequences.** Self-consistent with both FR-005/FR-010's render-selection rule and
  FR-014's action rule. If the intent was instead "every pick sets the selection even for
  edit-only apps (relaxing FR-010's render rule)," that's a one-line change to the Files
  app's pick handler — surfaced as Open Question O-1 rather than silently decided.

---

## 8. Risks / open questions

- **O-1 (spec ambiguity, needs a human call):** How should picking an **edit-only** handler
  in "Open with" affect the *selected* (default) handler? ADR-6's design reading: it does
  **not** (selection stays render-only per FR-010); an alternative: it *does* set it (relax
  FR-010's render rule). The mockup's "always open with" annotation (Option A) is written
  assuming the render-capable case and doesn't settle the edit-only case. **Default in this
  design = ADR-6.** Cheap to flip if the other way is intended.
- **R-1 — Client/server MIME normalization parity.** The Files app (client) computes the
  file's base MIME to (a) request the right handler view and (b) know the no-handler
  fallback. It must normalize the same way `baseMime` does server-side. Mitigation:
  `baseMime` is a dependency-free string function; keep it in `src/lib/mime.ts` and have
  the client import the same logic (or replicate it verbatim with a comment pointing at
  the server copy). A drift here would only cause a *spurious no-handler fallback*, not a
  crash — low severity, but worth a shared helper.
- **R-2 — `withFileParams` must not clobber the app's own `url`.** For an installed iframe,
  the manifest's `params.url` is the app's entry page; the contract's `url` param (from
  `paramShape`) is a *separate* field delivered as a `bos*` query param, so there is no
  collision as long as `withFileParams` reads the contract fields off `win.params`
  (path/action/url/title) and never overwrites `params.url`. Mirrors how `withEventParams`
  adds `bos*` params without touching `url`.
- **R-3 — `launch` returns `null` for an unlaunchable app.** The registry's "selected"
  should only ever point at installed apps, but as a final guard the Files app treats a
  `null` launch return as a fallback to the in-app path (no dead window).
- **R-4 — html-viewer is `hidden: true`.** It is a valid handler and must appear in "Open
  with" by its own label/icon (FR-008 / A-6). The Files app must *not* filter handlers by
  `hidden` — it renders the target app's own `icon` and `label ?? name` regardless of dock
  visibility. (Grounded: `html-viewer/manifest.ts` has `hidden: true`, `icon: "Code2"` —
  matches the mockup's "Web View / Code2 glyph" row.)
- **O-2 (follow-on dependency, out of scope):** The `agentic-text-editor` item (A-7) will
  declare a `text/html` handler (`paramShape:{url:"app"}`) and must implement the contract
  on the handler side (read `bosFilePath`/`bosFileAction`, fetch via SDK). This design's
  contract (§6) is that item's integration contract — it should be linked from the item's
  spec.

---

## 9. UI mockup reference

**Path:** `/Specs/user-specs/core-platform/036-file-type-handlers/mockup.html`

Two states, both mapping directly onto the Component design (§3.3, Files app):

- **Frame 1 — `report.html` (two handlers for `text/html`):** the right-click menu shows an
  **`Open with` section header**, then one row per handler — **Web View** (checked = current
  selected default; icon = html-viewer's own `Code2` manifest icon) and **Editor**
  (unchecked; its own icon), a **hairline divider**, then the pre-existing **Download**
  action. Maps to: `GET /api/file-handlers?mime=text/html` → rows; `selected` → the
  checkmark; each row's icon = the **target app's manifest `icon`** (never a semantic
  glyph — the mockup's note calls this out explicitly); the Files app renders the existing
  Download action after the divider. Picking the unchecked row = launch with `action` +
  `POST` the selection (FR-010, Option A; ADR-6).
- **Frame 2 — `notes.txt` (no handlers for `text/plain`):** the menu has **no `Open with`
  section and no divider** — it collapses to the pre-existing Download action; double-click
  uses the unchanged in-app behavior (FR-007 / FR-008 edge case 4 / SC-003). Maps to:
  `GET` returns an empty `handlers` list → the Files app renders the menu exactly as it
  does today.

The mockup confirms the two decisions I flagged as load-bearing: (1) row icon = the target
app's own manifest icon (R-4), and (2) the no-handler menu is byte-for-byte the current
menu (SC-003).
