# BOS Plugin Infrastructure

**Status:** Draft  
**Scope:** Platform  
**Directory:** `032-dynamic-integration-plugins` (original scope was integration-only; broadened)

---

## 1. Problem Statement

BOS has two distinct plugin extension points today — the existing server plugin system
(LLM pipeline hooks, `dataDir/plugins/`) and the integration framework (OAuth + adapter
methods, compiled into the bundle). Neither covers the general case of a marketplace
item that needs to:

- expose custom authenticated API routes,
- register a new engine or capability type,
- inject a settings panel, and
- serve its own frontend app — all without recompiling BOS.

The goal of this spec is a general **plugin infrastructure** layer that any plugin type
can build on. Integration plugins (OAuth + adapters) are the first concrete consumer;
voice engine plugins (`033-pluggable-voice-engines`) are the second. More will follow.

---

## 2. Goals

- **G1** — Any marketplace item can expose API surface through a single generic
  catch-all route, without BOS pre-building type-specific routes.
- **G2** — A marketplace item can register a settings panel that appears in the BOS
  Settings app when the item is installed.
- **G3** — A marketplace item can serve its own iframe application through its plugin
  routes, inheriting BOS same-origin status (WebRTC-capable, no opaque sandbox).
- **G4** — A marketplace item can declare an `integration` facet that registers OAuth
  manifests, adapter methods, and LLM tools at runtime.
- **G5** — Plugin loading uses dynamic `import()` via Next.js `instrumentation.ts`,
  running once at server startup before any request is handled.
- **G6** — The LLM capabilities registry derives its tool list from the live adapter
  registry at request time, not from a hardcoded compile-time array.
- **G7** — Built-in integrations (GSuite, Telegram) continue to work unchanged.
- **G8** — Installing or uninstalling a plugin does not require a server restart.

## 3. Non-Goals

- **NG1** — TypeScript/JSX hot compilation on install. Plugins are pre-compiled
  CommonJS JS.
- **NG2** — Custom React component injection into the settings shell. Panels are
  JSON Schema-driven or iframe only.
- **NG3** — Moving built-in integrations out of the source tree in the same release.
- **NG4** — Client-side (browser) plugin loading. Plugin entry files are server-only.

---

## 4. Current Architecture — Coupling Map

| Subsystem | File | Dynamic today? | Blocker |
|---|---|---|---|
| API routes | Next.js App Router | **No** | File-system based; compiled at build time |
| Manifest registry | `src/lib/integrations/registry.ts` | Yes — `registerIntegration()`, `globalThis`-backed | Service barrels hardcoded in `index.ts` |
| Adapter registry | `src/lib/integrations/actions/adapter-registry.ts` | Yes — `registerAdapter()` | Same hardcoded imports; throws on duplicate |
| Capabilities registry | `src/lib/agent/capabilities-registry.ts` | **No** | Hardcoded descriptor imports; frozen array at module load |
| Webhook registry | `src/lib/integrations/webhooks/registry.ts` | **No** | Static `HANDLERS` object literal |
| Settings tabs | `src/lib/config/registry.ts` + `src/apps/settings/index.tsx` | **No** | `REGISTRATIONS` array and `CUSTOM_TABS` map are hardcoded |
| OAuth manager | `src/lib/integrations/oauth/manager.ts` | Yes | No coupling to specific integrations |
| Invoke route | `src/app/api/integrations/[id]/services/[serviceId]/invoke/route.ts` | Yes | Reads registries at request time |
| Integrations list route | `src/app/api/integrations/route.ts` | Yes | Reads registries at request time |

---

## 5. Plugin Package Format

A plugin is a directory installable from the marketplace, placed at:

```
data/bos-plugins/<plugin-id>/
```

### 5.1 Required files

```
<plugin-id>/
├── bos-plugin.json     # plugin manifest (§5.2)
└── index.js            # compiled server entry (§5.3)
```

### 5.2 `bos-plugin.json`

```jsonc
{
  "id": "live-avatar",
  "name": "Live Avatar",
  "version": "1.0.0",
  "description": "Lip-synced avatar powered by AVTR-1.",
  "bosVersion": ">=1.4.0",    // semver range of compatible BOS versions
  "sdkVersion": "1",          // BOS Plugin SDK major version (§6)
  "entry": "index.js"         // server-side entry (default: index.js)
}
```

### 5.3 `index.js` — server entry

Runs in the BOS Node.js process. Must import exclusively from `@bos/plugin-sdk` (§6).
Must export `activate(ctx)` and `deactivate(ctx)` async functions.

```js
"use strict";
const sdk = require("@bos/plugin-sdk");

module.exports = {
  async activate(ctx) {
    // Register routes, integrations, voice engines, settings panels …
    sdk.registerRoute("GET", "/session", async (req) => {
      return Response.json({ ok: true });
    });
    ctx.log.info(`${ctx.pluginId} activated`);
  },
  async deactivate(ctx) {
    // Unregister everything registered during activate
    sdk.unregisterRoutes(ctx.pluginId);
    ctx.log.info(`${ctx.pluginId} deactivated`);
  },
};
```

### 5.4 Optional: `client.js` — framework-free metadata

Served to the browser via `GET /api/plugin/<id>/~client` so the client-side
capabilities registry can read plugin tool metadata without importing server-only
code. Must contain only JSON-serialisable data with no Node.js dependencies.

---

## 6. BOS Plugin SDK (`@bos/plugin-sdk`)

A thin re-export layer plugins import instead of reaching into BOS internals.
Versioned independently of BOS internals so plugins have a stable contract.

### 6.1 Exports (v1)

```typescript
// ── Route dispatch ──────────────────────────────────────────────────────────
/**
 * Register an authenticated API route under /api/plugin/<pluginId>/<path>.
 * handler receives a standard Request and must return a Response.
 * All registered routes are automatically unregistered on deactivate.
 */
export function registerRoute(
  method: "GET" | "POST" | "PUT" | "DELETE" | "PATCH",
  path: string,
  handler: (req: Request, ctx: PluginRouteContext) => Promise<Response>,
): void;

export function unregisterRoutes(pluginId: string): void;

// ── Settings panels ─────────────────────────────────────────────────────────
/**
 * Register a settings panel that appears in the BOS Settings app.
 * configSchema follows JSON Schema draft-07; stored under the plugin's
 * config namespace. secretFields lists keys stored in secrets store.
 */
export function registerSettingsPanel(config: {
  pluginId: string;
  label: string;
  icon: string;              // lucide-react icon name
  order?: number;
  configSchema: JSONSchema;
  secretFields?: string[];   // keys whose values are never returned to browser
}): void;

export function unregisterSettingsPanel(pluginId: string): void;

// ── Integration plugins ──────────────────────────────────────────────────────
export function registerIntegration(manifest: IntegrationManifest): void;
export function unregisterIntegration(id: string): void;
export function registerAdapter(
  integrationId: string,
  serviceId: string,
  entry: AdapterEntry,
): void;
export function unregisterAdapter(integrationId: string, serviceId: string): void;
export function registerWebhookHandler(
  integrationId: string,
  serviceId: string,
  handler: WebhookHandler,
): void;
export function unregisterWebhookHandler(integrationId: string, serviceId: string): void;

// ── Voice engines (see 033-pluggable-voice-engines) ──────────────────────────
export function registerVoiceEngine(engine: VoiceEnginePlugin): void;
export function unregisterVoiceEngine(engineId: string): void;

// ── Base classes / utilities ─────────────────────────────────────────────────
export { ServiceAdapter } from "@/lib/integrations/adapters/base";
export {
  IntegrationConfigError,
  IntegrationAuthError,
  IntegrationScopeError,
} from "@/lib/integrations/errors";

// ── Types ────────────────────────────────────────────────────────────────────
export type {
  IntegrationManifest, ServiceDefinition, OAuthConfig,
  AdapterEntry, AdapterMethodMeta, AdapterMethodParameter,
  WebhookHandler, VoiceEnginePlugin, PluginRouteContext,
} from "@/lib/plugins/types";
```

### 6.2 Resolution

`@bos/plugin-sdk` is a `package.json` exports alias pointing at
`src/lib/plugins/sdk/index.ts`. Plugins load with
`import(/* webpackIgnore: true */ path)` after the Next.js build and resolve the
alias from the running process's module graph — the same mechanism the existing
server plugin system uses.

---

## 7. Required Code Changes

### 7.1 Generic plugin catch-all route

**New file:** `src/app/api/plugin/[pluginId]/[...path]/route.ts`

This is the single API route in BOS core that dispatches to all plugin-registered
handlers. On each request it:

1. Verifies the BOS session cookie (same auth as all other BOS routes).
2. Looks up `pluginId` in the plugin route registry.
3. Finds the handler matching `method` + `path`.
4. Calls `handler(request, ctx)` and returns the `Response`.
5. Returns `404` if no matching handler is found, `503` if the plugin is not active.

The route registry is a `globalThis`-backed map:

```typescript
// src/lib/plugins/route-registry.ts
type RouteHandler = (req: Request, ctx: PluginRouteContext) => Promise<Response>;

interface RouteEntry {
  method: string;
  path: string;       // normalised, e.g. "/session"
  handler: RouteHandler;
}

const KEY = "__bos_plugin_routes__";
const registry: Map<string, RouteEntry[]> =
  (globalThis as any)[KEY] ??= new Map();

export function registerRoute(pluginId, method, path, handler): void { … }
export function unregisterRoutes(pluginId): void { … }
export function matchRoute(pluginId, method, path): RouteHandler | undefined { … }
```

The `globalThis` backing survives Next.js HMR re-evaluations in development.

**The same route serves the plugin-provided app.** A plugin that ships a frontend
app registers a handler for `GET /app` (returns `index.html`) and
`GET /app/:asset` (returns static assets). Because the URL
`/api/plugin/<id>/app` is served by BOS's own route handler, the resulting iframe
is same-origin — full WebRTC access, no opaque sandbox restriction.

### 7.2 Settings panel registry

**New file:** `src/lib/plugins/settings-registry.ts`

```typescript
interface SettingsPanelEntry {
  pluginId: string;
  label: string;
  icon: string;
  order: number;
  configSchema: JSONSchema;
  secretFields: string[];
}

// globalThis-backed
export function registerSettingsPanel(entry: SettingsPanelEntry): void;
export function unregisterSettingsPanel(pluginId: string): void;
export function listSettingsPanels(): SettingsPanelEntry[];
```

**Modified:** `src/lib/config/registry.ts` — `listRegistrations()` appends entries
from `listSettingsPanels()` so dynamically registered panels appear alongside
built-in tabs without any changes to the settings shell React code.

**Modified:** `src/app/api/config/route.ts` — the `GET /api/config` response already
serialises `listRegistrations()`, so plugin panels appear automatically.

**Secret field handling:** When a settings panel declares `secretFields`, the config
`GET` route redacts those keys (replaces values with `"••••••"`) and the `PATCH`
route writes them to the secrets store instead of the plain config store.

### 7.3 Capabilities registry — make dynamic

**Problem:** `src/lib/agent/capabilities-registry.ts` hardcodes every
method-descriptor import and builds `CAPABILITIES` at module load.

**Solution:**

1. Add `methodDescriptors` to `AdapterEntry` in `adapter-registry.ts`:

```typescript
export interface AdapterEntry {
  createAdapter: () => ServiceAdapter;
  methods: readonly AdapterMethodMeta<any>[];
  capabilities?: AdapterCapabilities;
  methodDescriptors: readonly AdapterMethodDescriptor[]; // NEW
}
```

2. Replace the hardcoded `CAPABILITIES` array with a `listCapabilities()` function
   that reads `listAdapterServices()` at call time and generates capability entries
   from each entry's `methodDescriptors`. Static (non-integration) capabilities
   remain hardcoded; only the integration portion is dynamic.

3. Update all call sites to call `listCapabilities()` per assistant run
   initialisation rather than importing the array once.

Built-in adapters populate `methodDescriptors` from their existing
`*_METHOD_DESCRIPTORS` exports with no other changes.

### 7.4 Webhook registry — add registration functions

**Modified:** `src/lib/integrations/webhooks/registry.ts`

Add `registerWebhookHandler` and `unregisterWebhookHandler`. The static `HANDLERS`
object becomes the seed; dynamic registrations are stored in a separate
`globalThis`-backed map so they survive HMR.

### 7.5 Adapter registry — add unregister + HMR guard

**Modified:** `src/lib/integrations/actions/adapter-registry.ts`

- Add `unregisterAdapter(integrationId, serviceId)`.
- Change duplicate-registration guard from "throw" to "replace" (matches manifest
  registry behaviour; prevents HMR breakage in dev).
- Back the registry map with `globalThis.__bos_adapter_registry__`.

### 7.6 Plugin loader

**New file:** `src/lib/plugins/loader.ts`

Scans `dataDir()/bos-plugins/` at startup, reads and validates each
`bos-plugin.json`, dynamically imports `index.js`, and calls `activate(ctx)`.
Per-plugin error isolation: one bad plugin must not prevent others from loading.
Loaded plugins recorded in `globalThis.__bos_loaded_plugins__`.

**New file:** `instrumentation.ts` (Next.js startup hook, at repo root)

```typescript
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    const { loadAllPlugins } = await import("./src/lib/plugins/loader");
    await loadAllPlugins();
  }
}
```

### 7.7 Marketplace facets

**Modified:** `src/lib/marketplace/schema.ts` — add new facet types:

```typescript
// Existing:
integration?: { entrypoint: string; version: string; }

// New:
voiceEngine?: {
  entrypoint: string;   // path to the directory containing bos-plugin.json
  version: string;
  engineId: string;     // matches VoiceEnginePlugin.id
}

// Updated app runtime options:
app?: {
  entrypoint: string;
  runtime: "iframe" | "plugin-served";  // plugin-served = served via plugin routes
  version: string;
  icon?: string;
}
```

**`plugin-served` runtime:** BOS opens the app in a window pointing at
`/api/plugin/<pluginId>/app` rather than `/apps/<id>/`. No files are copied or
symlinked; the plugin route handler serves HTML and assets from the plugin directory.

**Modified:** `src/lib/marketplace/client.ts` — install/uninstall handlers for
`integration` and `voiceEngine` facets call `activatePlugin(id)` /
`deactivatePlugin(id)` from `src/lib/plugins/loader.ts`.

---

## 8. Lifecycle

### 8.1 Install

```
marketplace install <id>
  → copy plugin directory to data/bos-plugins/<id>/
  → validate bos-plugin.json
  → dynamic import index.js
  → activate(ctx)  →  registerRoute(), registerSettingsPanel(), registerIntegration(), …
  → plugin immediately available (no restart)
```

### 8.2 Uninstall

```
marketplace uninstall <id>
  → deactivate(ctx)  →  unregisterRoutes(), unregisterSettingsPanel(), …
  → delete data/bos-plugins/<id>/
  → plugin removed (in-flight requests complete normally)
```

### 8.3 Server restart

`instrumentation.ts` re-loads all plugins in `data/bos-plugins/` before the first
request. Registrations survive restarts without user action.

### 8.4 Update

Uninstall followed by install of the new version. Plugin-specific data (OAuth tokens,
config) is stored under the plugin id and survives the update.

---

## 9. Security Considerations

- **Code execution:** Plugin `index.js` runs in the main BOS Node.js process with
  full filesystem and network access. Identical risk model to the existing server
  plugin system. Plugins must come from trusted sources.
- **Route authentication:** The catch-all route `GET|POST /api/plugin/[id]/[...path]`
  requires a valid BOS session cookie — the same auth as all other BOS API routes.
  Plugin route handlers do not need to implement their own auth.
- **Scope isolation:** The SDK does not export `getSecretsStore()`. Plugins access
  their own secrets only through the settings panel `secretFields` mechanism.
- **`client.js`:** Served unauthenticated (metadata only). Must not contain
  credentials or server-side logic.
- **`plugin-served` apps:** Served at `/api/plugin/<id>/app` — same-origin with BOS,
  behind session auth. No opaque sandbox; treat as trusted user content.

---

## 10. Open Questions

| # | Question | Options |
|---|---|---|
| OQ1 | `@bos/plugin-sdk` — npm package or path alias? | (a) Published npm package — clean versioning; (b) `package.json` exports alias — simpler, tied to BOS version |
| OQ2 | Hot-activation safety: is `import()` + registry mutation mid-request safe? | Registries are plain Maps — likely safe for read-heavy BOS; a mutex may be needed for production |
| OQ3 | Plugin distribution format: git repo (GitFS model) or tarball? | Git aligns with user-apps; tarballs are simpler for versioned releases |
| OQ4 | Should built-in integrations migrate to the plugin format? | After the loader is proven stable |

---

## 11. Affected Files

### New files
- `instrumentation.ts` — Next.js startup hook
- `src/lib/plugins/loader.ts` — plugin loader
- `src/lib/plugins/sdk/index.ts` — `@bos/plugin-sdk` entry point
- `src/lib/plugins/route-registry.ts` — catch-all route handler registry
- `src/lib/plugins/settings-registry.ts` — settings panel registry
- `src/lib/plugins/types.ts` — shared plugin types
- `src/app/api/plugin/[pluginId]/[...path]/route.ts` — generic catch-all route

### Modified files
- `src/lib/integrations/registry.ts` — add `unregisterIntegration()`
- `src/lib/integrations/actions/adapter-registry.ts` — `unregisterAdapter()`, HMR guard, `methodDescriptors` field
- `src/lib/integrations/webhooks/registry.ts` — `registerWebhookHandler()` / `unregisterWebhookHandler()`
- `src/lib/agent/capabilities-registry.ts` — replace frozen array with dynamic `listCapabilities()`
- `src/lib/config/registry.ts` — `listRegistrations()` includes plugin settings panels
- `src/app/api/config/route.ts` — no change required (reads `listRegistrations()` already)
- `src/lib/marketplace/schema.ts` — add `voiceEngine` facet; add `plugin-served` runtime
- `src/lib/marketplace/client.ts` — handle `integration` + `voiceEngine` facets
- `package.json` — add `@bos/plugin-sdk` exports alias
- All built-in adapter files — add `methodDescriptors` field to `registerAdapter()` call

---

## 12. Phasing

### Phase A — Plugin infrastructure foundations
Plugin loader, catch-all route + route registry, settings panel registry, SDK stub,
`unregister*` functions, webhook registry open, adapter registry HMR guard +
`methodDescriptors`, capabilities registry made dynamic.

Validation: a hand-crafted plugin in `data/bos-plugins/hello/` registers a route
`GET /hello` that returns `{ ok: true }`, a settings panel, and appears in
`/api/plugin/hello/hello`.

### Phase B — Integration plugin facet
`integration` facet in marketplace schema + install/uninstall. Validation: the
Google Photos integration packaged as a plugin, installed from marketplace,
functional end-to-end.

### Phase C — Voice engine facet
`voiceEngine` facet + `registerVoiceEngine()` in SDK. Detailed in
`033-pluggable-voice-engines`. Validation: the live-avatar plugin installed,
voice engine selectable in voice settings.

### Phase D — `plugin-served` app runtime
`plugin-served` app runtime + plugin route serving HTML/assets. Validation:
live-avatar app opened as a BOS window, served same-origin via plugin route,
WebRTC functional.

### Phase E — SDK package + developer tooling
TypeScript SDK types, build template, documentation for third-party plugin authors.
