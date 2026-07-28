# Dynamic Integration Plugins

**Status:** Draft  
**Scope:** Platform  

---

## 1. Problem Statement

All integrations (GSuite, Telegram, Google Photos, …) are compiled into the BOS server bundle at build time. Adding a new integration requires editing TypeScript source, rebuilding, and redeploying. This prevents users or third parties from distributing integrations as marketplace items, and it forces every BOS instance to ship every integration regardless of whether it is used.

The goal is a plugin model where an integration — its OAuth manifest, server-side adapter logic, and settings configuration — can be packaged, installed from the marketplace, and loaded at runtime without recompiling BOS.

---

## 2. Goals

- **G1** — A marketplace item can declare an `integration` facet that, when installed, registers a fully functional integration (OAuth flow, adapter methods, LLM tools, settings panel).
- **G2** — Integration plugins are loaded at server startup via dynamic `import()`, the same mechanism used by the existing server plugin system.
- **G3** — The built-in integrations (GSuite, Telegram) continue to work without any changes to how they are authored or registered.
- **G4** — The LLM capabilities registry derives its tool list from the live adapter registry at request time, not from a hardcoded compile-time array.
- **G5** — A plugin-provided settings UI uses JSON Schema-driven panels (no custom React components) unless the plugin ships an iframe panel for advanced configuration.
- **G6** — Installing or uninstalling an integration plugin does not require a server restart (activation takes effect on next request; deactivation requires a restart or a reload trigger — see §8.3).

## 3. Non-Goals

- **NG1** — TypeScript/JSX hot compilation on install. Plugins are pre-compiled to CommonJS JS before packaging.
- **NG2** — Full UI extensibility (custom React components injected into the settings shell). Settings panels are JSON Schema or iframe only.
- **NG3** — Moving the built-in integrations out of the source tree in the same release. Migration is optional and incremental (§9).
- **NG4** — Client-side (browser) plugin loading. Adapter code is server-only.

---

## 4. Current Architecture — Coupling Map

The table below lists each subsystem, whether it currently supports dynamic registration, and what blocks it.

| Subsystem | File | Dynamic today? | Blocker |
|---|---|---|---|
| Manifest registry | `src/lib/integrations/registry.ts` | Yes — `registerIntegration()` exists, backed by `globalThis` | Service barrels are hardcoded in `index.ts` |
| Adapter registry | `src/lib/integrations/actions/adapter-registry.ts` | Yes — `registerAdapter()` exists | Same hardcoded imports; throws on duplicate (no HMR guard) |
| Capabilities registry | `src/lib/agent/capabilities-registry.ts` | **No** | Hardcoded imports of every method-descriptor file; CAPABILITIES is a frozen array built at module load |
| Webhook registry | `src/lib/integrations/webhooks/registry.ts` | **No** | Static `HANDLERS` object literal; no registration function |
| Settings tabs | `src/lib/config/registry.ts` + `src/apps/settings/index.tsx` | **No** | `REGISTRATIONS` array and `CUSTOM_TABS` map are hardcoded; no runtime registration |
| OAuth manager | `src/lib/integrations/oauth/manager.ts` | Yes | No coupling to specific integrations |
| Invoke route | `src/app/api/integrations/[id]/services/[serviceId]/invoke/route.ts` | Yes | Reads registries at request time; works as-is once registries are populated |
| Integrations list route | `src/app/api/integrations/route.ts` | Yes | Same — reads registries at request time |

---

## 5. Plugin Package Format

An integration plugin is a directory installable from the marketplace. It is placed at:

```
data/integration-plugins/<integration-id>/
```

### 5.1 Required files

```
<integration-id>/
├── integration-plugin.json   # plugin manifest (§5.2)
├── index.js                  # compiled entry point (§5.3)
└── client.js                 # framework-free method descriptors (§5.4)
```

### 5.2 `integration-plugin.json`

```jsonc
{
  "id": "google-photos",           // must match the IntegrationManifest id
  "name": "Google Photos",
  "version": "1.0.0",
  "description": "Browse albums and search media in Google Photos.",
  "bosVersion": ">=1.4.0",         // semver range of compatible BOS versions
  "sdkVersion": "1",               // BOS Integration SDK major version (§6)
  "entry": "index.js",             // server-side entry (default: index.js)
  "clientEntry": "client.js"       // framework-free descriptors (default: client.js)
}
```

### 5.3 `index.js` — server entry

The entry file runs in the BOS Node.js process. It MUST:
1. Import from `@bos/integration-sdk` (§6) for all BOS internals.
2. Call `registerIntegration(manifest)` and `registerAdapter(...)` synchronously during module evaluation, or in an exported `activate()` async function.
3. Not start network connections, timers, or background tasks at module load.

```js
// example: google-photos/index.js (compiled from TypeScript)
"use strict";
const sdk = require("@bos/integration-sdk");

const MANIFEST = {
  id: "google-photos",
  name: "Google Photos",
  // …
};

sdk.registerIntegration(MANIFEST);

const { PhotosAdapter } = require("./adapter.js");
const { PHOTOS_METHODS }  = require("./methods.js");

sdk.registerAdapter("google-photos", "photos", {
  createAdapter: () => new PhotosAdapter(),
  methods: PHOTOS_METHODS,
});

// Optional: register a webhook handler
// sdk.registerWebhookHandler("google-photos", "photos", new PhotosWebhookHandler());

module.exports = {
  async activate(ctx) {
    ctx.log.info("google-photos integration activated");
  },
  async deactivate(ctx) {
    sdk.unregisterIntegration("google-photos");
    sdk.unregisterAdapter("google-photos", "photos");
  },
};
```

### 5.4 `client.js` — framework-free descriptors

This file is served to the browser (via a new `/api/integration-plugins/[id]/client` route) so the client-side capabilities registry can enumerate the plugin's tools without importing server-only code. It MUST only contain JSON-serialisable data and no Node.js dependencies.

```js
// example: google-photos/client.js
exports.PHOTOS_METHOD_DESCRIPTORS = [
  {
    method: "albums_list",
    scope: "https://www.googleapis.com/auth/photoslibrary.readonly",
    description: "List the user's Google Photos albums.",
    parameters: [ /* … */ ],
  },
  // …
];
```

### 5.5 Optional: iframe settings panel

If the integration requires a richer settings UI than JSON Schema allows, it can ship a static HTML file:

```
<integration-id>/
└── settings/
    └── index.html    # served at /api/integration-plugins/<id>/settings/
```

The BOS settings shell renders this as a sandboxed iframe when the user opens the integration's settings. The iframe communicates with BOS via `postMessage` using a documented protocol (to be specified separately).

---

## 6. Integration SDK (`@bos/integration-sdk`)

The SDK is a thin re-export layer that integration plugins import instead of reaching into BOS source paths directly. This gives plugins a stable contract that BOS can version independently of its own internal refactors.

### 6.1 Exports (v1)

```typescript
// Registration
export function registerIntegration(manifest: IntegrationManifest): void;
export function unregisterIntegration(id: string): void;
export function registerAdapter(integrationId: string, serviceId: string, entry: AdapterEntry): void;
export function unregisterAdapter(integrationId: string, serviceId: string): void;
export function registerWebhookHandler(integrationId: string, serviceId: string, handler: WebhookHandler): void;
export function unregisterWebhookHandler(integrationId: string, serviceId: string): void;

// Base classes / helpers
export { ServiceAdapter } from "@/lib/integrations/adapters/base";
export { IntegrationConfigError, IntegrationAuthError, IntegrationScopeError } from "@/lib/integrations/errors";

// Fetch utilities
export { gsuiteFetch, gsuiteFetchBinary, buildUrl } from "@/lib/integrations/services/gsuite/client";
// NOTE: gsuiteFetch is GSuite-specific. A generic authedFetch is exposed via ServiceAdapter.authedFetch.

// Types
export type {
  IntegrationManifest, ServiceDefinition, OAuthConfig,
  AdapterEntry, AdapterMethodMeta, AdapterMethodParameter,
  WebhookHandler,
} from "@/lib/integrations/types";
```

### 6.2 Resolution

`@bos/integration-sdk` is a package alias declared in `package.json`'s `exports` map pointing at `src/lib/integrations/sdk/index.ts`. Plugins import it as a bare specifier; at runtime Node.js resolves it to the installed BOS module.

Because plugins are loaded with `import(/* webpackIgnore: true */ path)` after the Next.js build, they import from the running process's `node_modules` — the same mechanism the existing server plugin system uses.

---

## 7. Required Code Changes

### 7.1 Capabilities registry — make dynamic

**Problem:** `src/lib/agent/capabilities-registry.ts` hardcodes imports of every method-descriptor file and builds `CAPABILITIES` at module load.

**Solution:** Split into two parts:

1. **Static capabilities** — all non-integration capabilities (file tools, terminal, etc.) remain as a hardcoded array.
2. **Dynamic integration capabilities** — a new `listIntegrationCapabilities()` function builds the integration portion of the capability list on demand by reading `listAdapterServices()` from the adapter registry and pairing each entry with its method descriptors. The method descriptors are retrieved from a per-adapter `getMethodDescriptors()` call added to `AdapterEntry`.

```typescript
// New field on AdapterEntry in adapter-registry.ts
export interface AdapterEntry {
  createAdapter: () => ServiceAdapter;
  methods: readonly AdapterMethodMeta<any>[];
  capabilities?: AdapterCapabilities;
  // NEW: framework-free descriptors for the client/capabilities registry
  methodDescriptors: readonly { method: string; scope: string; description: string; parameters: AdapterMethodParameter[] }[];
}
```

Built-in adapters populate `methodDescriptors` from their existing `*_METHOD_DESCRIPTORS` exports. Plugin adapters do the same from their `client.js`.

3. **`CAPABILITIES` export becomes a getter** that concatenates static + `listIntegrationCapabilities()`. Call sites that read `CAPABILITIES` once at startup are updated to call the getter on each assistant run initialisation instead.

### 7.2 Webhook registry — add registration function

Add `registerWebhookHandler` and `unregisterWebhookHandler` to `src/lib/integrations/webhooks/registry.ts`. The static `HANDLERS` object becomes the seed; dynamic registrations are stored separately in a `globalThis`-backed map (same pattern as the manifest registry) so they survive Next.js HMR re-evaluations.

### 7.3 Adapter registry — add unregister + HMR guard

- Add `unregisterAdapter(integrationId, serviceId)` for clean plugin deactivation.
- Change the duplicate-registration guard from "throw" to "replace" (matching manifest registry behaviour) so Next.js HMR doesn't break dev reloads of plugin adapters.
- Store registry in `globalThis.__bos_adapter_registry__` (same pattern as manifest registry).

### 7.4 Integration plugin loader

New file: `src/lib/integration-plugins/loader.ts`

Responsibilities:
- Scan `dataDir()/integration-plugins/` at startup.
- Read and validate each `integration-plugin.json`.
- Dynamically import each plugin's `entry` file (`index.js`) with `import(/* webpackIgnore: true */ entryPath)`.
- Call the exported `activate(ctx)` function if present.
- Record loaded plugins in `globalThis.__bos_loaded_integration_plugins__`.
- Per-plugin error isolation: a failure loading one plugin must not prevent others from loading.

The loader is invoked from Next.js `instrumentation.ts` (`register()` export), which runs once at server startup before any request is handled.

### 7.5 Marketplace facet: `integration`

Add `integration` as a recognised facet in `src/lib/marketplace/schema.ts`:

```typescript
integration?: {
  entrypoint: string;   // path to the directory containing integration-plugin.json
  version: string;
}
```

The marketplace install logic (`client.ts`) handles this facet by:
1. Copying the integration directory to `data/integration-plugins/<id>/`.
2. Calling `activateIntegrationPlugin(id)` — a hot-registration path that loads the plugin immediately without a server restart (best-effort; falls back to "restart required" notice).

Uninstall calls `deactivateIntegrationPlugin(id)`, which calls the plugin's `deactivate()` hook and removes its registrations.

### 7.6 Settings UI — dynamic config tab

For each loaded integration plugin, the settings system automatically creates a config tab driven by the `configSchema` defined in the plugin's `IntegrationManifest` services. No new settings plumbing is required beyond what already exists for built-in services.

If the plugin ships a `settings/index.html`, the integrations settings panel renders an iframe pointing at `/api/integration-plugins/<id>/settings/` in place of the default JSON Schema form.

A new API route `GET /api/integration-plugins` returns the list of loaded plugin manifests so the client can render integration cards in the settings shell.

---

## 8. Lifecycle

### 8.1 Install

```
marketplace install google-photos
  → copy files to data/integration-plugins/google-photos/
  → validate integration-plugin.json
  → dynamic import index.js
  → registerIntegration(), registerAdapter()
  → activate(ctx)
  → integration is immediately available (no restart)
```

### 8.2 Uninstall

```
marketplace uninstall google-photos
  → deactivate(ctx)
  → unregisterAdapter(), unregisterIntegration()
  → delete data/integration-plugins/google-photos/
  → integration removed (no restart; in-flight requests complete normally)
```

### 8.3 Server restart

On startup, `instrumentation.ts` calls the integration plugin loader, which re-imports and re-activates all installed plugins before the first request is served. Plugin registrations therefore survive restarts without any user action.

### 8.4 Update

An update is an uninstall followed by an install of the new version. The OAuth tokens and per-service config stored in the secrets store and state store are keyed by integration id and survive the update as long as the id does not change.

---

## 9. Security Considerations

- **Code execution**: Integration plugin `index.js` runs in the main BOS Node.js process with full access to the filesystem, network, and secrets store. This is identical to the existing server plugin model. Plugins must be from trusted sources; the marketplace UI should display provenance and allow admins to review plugin code before activation.
- **Scope isolation**: The SDK does not expose a way for a plugin to read another integration's OAuth tokens. `getSecretsStore()` is NOT exported; adapters access tokens only via `ServiceAdapter.authedFetch`, which reads tokens for the adapter's own integration id.
- **Client entry**: `client.js` is served unauthenticated. It must contain only static metadata (method descriptors) — no credentials, no logic that touches secrets.
- **iframe settings panel**: Sandboxed with `sandbox="allow-scripts allow-forms"` (no `allow-same-origin`). PostMessage protocol is verified by origin.

---

## 10. Open Questions

| # | Question | Options | Decision needed by |
|---|---|---|---|
| OQ1 | Should `@bos/integration-sdk` be a real npm package or a path alias only? | (a) npm package published separately — cleaner versioning, harder to keep in sync; (b) path alias in BOS `package.json` — simpler, but plugins must be built against a specific BOS install | Before SDK design is finalised |
| OQ2 | Hot-activation on install: is it always safe to `import()` and `registerAdapter()` mid-request? | The registries are backed by plain objects (no locks). Likely safe for read-heavy workloads but may need a mutex for production. | Before loader implementation |
| OQ3 | How are plugin-provided OAuth apps handled? A GSuite plugin still requires the user to upload a `client_secrets.json`. Is the existing `/api/integrations/[id]/client-secret` route sufficient, or does a plugin need to declare its own credential type? | Existing route is generic by integration id — it should work as-is. | Early implementation |
| OQ4 | Should the built-in integrations (GSuite, Telegram) be migrated to the plugin format? | (a) Yes — validates the system and reduces bundle size; (b) No — built-ins benefit from TypeScript source and tighter integration; keep as compiled-in | After the loader is proven stable |
| OQ5 | Plugin distribution format: git repo (like user-apps) or tarball/zip? | Git repos align with the existing GitFS model. Tarballs are simpler for versioned releases. | Marketplace spec |

---

## 11. Affected Files

### New files
- `src/lib/integration-plugins/loader.ts`
- `src/lib/integration-plugins/types.ts`
- `src/lib/integrations/sdk/index.ts` (`@bos/integration-sdk` entry point)
- `src/app/api/integration-plugins/route.ts` (list loaded plugins)
- `src/app/api/integration-plugins/[id]/client/route.ts` (serve client.js)
- `src/app/api/integration-plugins/[id]/settings/route.ts` (serve iframe settings HTML)
- `instrumentation.ts` (Next.js startup hook — new file at repo root)

### Modified files
- `src/lib/integrations/registry.ts` — add `unregisterIntegration()`; ensure `globalThis` guard
- `src/lib/integrations/actions/adapter-registry.ts` — add `unregisterAdapter()`; change duplicate guard; add `methodDescriptors` to `AdapterEntry`; use `globalThis` guard
- `src/lib/integrations/webhooks/registry.ts` — add `registerWebhookHandler()` / `unregisterWebhookHandler()`
- `src/lib/agent/capabilities-registry.ts` — remove hardcoded descriptor imports; implement dynamic `listIntegrationCapabilities()`
- `src/lib/marketplace/schema.ts` — add `integration` facet
- `src/lib/marketplace/client.ts` — handle `integration` facet install/uninstall
- `package.json` — add `@bos/integration-sdk` exports alias
- All built-in adapter files — add `methodDescriptors` field to their `registerAdapter()` call

---

## 12. Phasing

### Phase A — Foundations (server-side, no UI changes)
Deliverables: loader, SDK stub, `unregister*` functions, webhook registry open, adapter registry `methodDescriptors`, capabilities registry made dynamic. Built-in integrations continue to work unchanged. No marketplace UI yet.

Validation: the loader successfully imports a hand-crafted `google-photos` integration plugin from `data/integration-plugins/` and the integration appears in `/api/integrations` and functions end-to-end.

### Phase B — Marketplace integration
Deliverables: `integration` facet in marketplace schema + install/uninstall logic + `GET /api/integration-plugins`. Built-in marketplace UI shows integration plugins as installable items.

### Phase C — Settings UI + iframe panel support
Deliverables: dynamic config tabs for plugin-provided services; iframe panel protocol; `client.js` serving route.

### Phase D — SDK package + developer tooling
Deliverables: TypeScript SDK types, build template, documentation. Enables third-party plugin authors to build and publish integration plugins.
