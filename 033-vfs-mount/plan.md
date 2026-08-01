# Implementation Plan: VFS Mount (WebDAV)

**Spec**: `spec.md` (20 FRs, 10 SCs, 6 user stories) · **Feature branch**: `033-vfs-mount`

## 1. Architecture decision that the draft missed: HTTP verb transport

WebDAV requires `PROPFIND`, `MKCOL`, `COPY`, `MOVE` in addition to the standard verbs. **Next.js 16's App Router route handlers do not support this.** Verified directly against the installed `next@16.2.9`:

```
// node_modules/next/dist/server/web/http.js
const HTTP_METHODS = ['GET','HEAD','OPTIONS','POST','PUT','DELETE','PATCH'];
```

`route-modules/app-route/module.js#resolve(method)` returns a bare `400` response for any method outside that list *before* your handler code ever runs. A `route.ts` at `src/app/api/vfs/webdav/[...path]/route.ts` exporting `PROPFIND`/`MKCOL`/`COPY`/`MOVE` is silently unreachable for those methods — this would have surfaced as a confusing “works for GET/PUT, 400 for PROPFIND” bug well into implementation if unaddressed.

**Decision:** implement the whole WebDAV protocol in `src/middleware.ts` (root Next.js Middleware, `runtime: "nodejs"`), scoped via `matcher` to `/api/vfs/webdav/:path*`. Middleware intercepts every request that matches, for *any* HTTP method, before Next's per-method route resolution — it has no `isHTTPMethod` gate. Confirmed present as a real (non-experimental-flag-gated) code path in this Next version: `next-server.js`'s `loadNodeMiddleware()`. Middleware can terminate the request by returning a `Response` directly, so no `route.ts` is needed at all for this path.

- **Phase 0 below is a go/no-go spike** to empirically confirm this before any other work depends on it (in particular: that Node-runtime middleware can stream a request body and a file read without buffering, and that it doesn't require a `next.config.ts` experimental flag, which would violate the "don't touch build config" rule in `CLAUDE.md`).
- **Documented fallback** if the spike fails: BOS's dev-mode entry point (`npm run dev` → `next dev`) has no reverse proxy in front of it, so the only other place non-standard verbs could be intercepted pre-Next is `tools/supervisor/supervisor.mjs`. That file is explicitly the trusted kernel and "NOT itself self-modified" (its own header comment) — it is off-limits for feature work, and it isn't in the request path for local dev by default anyway. If the spike fails, escalate rather than silently reaching for the Supervisor.

## 2. VFS layer gaps to close

`src/os/vfs.ts` today (confirmed by reading the full file):

```ts
list(vfsPath) / stat(vfsPath) / readText(vfsPath) / readBuffer(vfsPath) /
writeText(vfsPath, s) / writeBuffer(vfsPath, buf) / mkdir(vfsPath) /
remove(vfsPath) / rename(fromPath, toPath) / hostPath(vfsPath) / registerMount(...)
```

Everything reads/writes whole `Buffer`s via `fs.promises`; `writeText/writeBuffer` go through `writeFileAtomic` (`src/os/atomic-write.ts`) for crash-safety only, not streaming. There is no `copy`, no streaming API, and mount-aware dispatch (`findMount` → `FSBackend`, `src/os/fs-types.ts`) governs whether a path falls under `/Specs`, `/Docs`, `/Templates`, or plain local FS.

Required additions (Phase 1):
- `vfs.readStream(vfsPath): Promise<Readable>` / `vfs.writeStream(vfsPath, stream: Readable): Promise<void>` — for the default (unmounted) local backend, implemented with `fs.createReadStream`/`fs.createWriteStream` piped through the same atomic-rename discipline `writeFileAtomic` uses (write to `<path>.tmp-<rand>`, `fsync`, `rename`).
- `vfs.copy(fromPath, toPath): Promise<void>` — read+write for the local backend; for mounted paths, delegate to the backend if it grows a `copy()`, else reject with a clear error (mirroring `rename`'s existing cross-mount-boundary refusal).
- `FSBackend` (`src/os/fs-types.ts`) gets **optional** `readStream?`/`writeStream?`/`copy?` members. Backends that don't implement them (`SpecFS`, `DocsFS`, `ReadonlyFS`) fall back to buffered `readBuffer`/`writeBuffer` inside `vfs.ts` — call this out explicitly as a scoped limitation: FR-009/SC-006 streaming guarantees apply to the default local VFS root, not to files living under a mounted subtree. This is acceptable because SC-006's 200 MB test target is an ordinary `data/vfs/` file, and revisit only if a real workload needs it.
- Reuse `src/os/path-jail.ts`'s `jailResolve` (already the shared traversal-rejection primitive for directory-rooted backends) rather than reinventing path-escape logic — see Phase 3.

## 3. Auth & token storage — grounded in `SecretsStore`, not "CryptFS"

FR-003 asks for "cryptographically random 256-bit value, stored as a bcrypt hash in a CryptFS-encrypted `system` config namespace." Since neither bcrypt nor CryptFS exist in `src/`, satisfy the requirement's *intent* (irreversible hash + encryption at rest) with real primitives:

- Token generation: `crypto.randomBytes(32)` (same primitive already used in `src/lib/integrations/secrets/keyfile.ts`), base64url-encoded for display/transport.
- Token storage: `getSecretsStore().set("vfs-webdav", tokenId, { hash, createdAt, label })` — this is AES-256-GCM-at-rest via the existing `SecretsStore`, satisfying "encrypted." `tokenId` is a short random id used as a lookup key so we never store or scan raw tokens.
- Hashing: FR-003 explicitly says bcrypt. **Adding `bcryptjs` is a new dependency and therefore a `package.json` change** — flagged per `CLAUDE.md` ("don't touch package.json... unless asked"). Two options for sign-off before Phase 3 starts:
  1. Add `bcryptjs` (small, no native build step, already used by `bastion/`) — literal spec compliance.
  2. Use Node's built-in `crypto.scrypt` + `crypto.timingSafeEqual` — equivalent security property (irreversible, salted, constant-time compare), zero new dependency.
  - **Default to option 2** unless the user confirms a new dependency is acceptable; note the substitution in `spec.md`'s implementation notes once decided.
- Revocation: delete the `SecretsStore` entry for that `tokenId`; multiple tokens are just multiple keys under the `"vfs-webdav"` namespace (`listKeys("vfs-webdav")` already exists for enumeration).
- Opt-in gate (FR-012): the middleware checks whether any token exists (`listKeys("vfs-webdav").length > 0`) before doing anything else; if none, return `404`.

## 4. Module layout

```
src/middleware.ts                       # NEW — Node runtime, matcher: /api/vfs/webdav/:path*
src/lib/webdav/
  handler.ts                            # dispatch(request) → Response; wires auth → path-security → method
  auth.ts                               # bearer-token check against SecretsStore("vfs-webdav")
  tokens.ts                             # generate/hash/verify/revoke/list (Phase 3 primitives)
  path-security.ts                      # decode-and-reject encoded traversal, then jailResolve
  xml.ts                                # RFC 4918 multistatus + <d:error> builders
  logging.ts                            # thin wrapper: logger() with COMPONENT = "vfs.webdav"
  methods/
    options.ts  propfind.ts  get.ts  head.ts
    put.ts  delete.ts  mkcol.ts  copy.ts  move.ts
src/os/vfs.ts                           # + readStream/writeStream/copy (Phase 1)
src/os/fs-types.ts                      # + optional readStream?/writeStream?/copy? on FSBackend
data/user-apps/items/vfs-webdav-mount/  # marketplace item (Phase 6)
  services/vfs-webdav-mount/service.json
  settings/                             # Local Mount panel source
  doc/  spec/
src/components/apps/settings/LocalMountTab.tsx   # or item-bundled equivalent — see Phase 6 decision
tests/webdav/*.test.ts                  # unit (Playwright unit runner, testDir "./tests")
e2e/vfs-webdav.spec.ts                  # e2e (Playwright, testDir "./e2e")
```

## 5. Phases

Each phase lists concrete steps and the FRs/SCs/edge cases it closes. Full requirement↔task traceability lives in `tasks.md`.

**Critical: marketplace.json** — The WebDAV service IS a BOS Marketplace item at `data/user-apps/items/vfs-webdav-mount/`. The `marketplace.json` register step (T9) is a **mandatory deliverable** — this feature does not ship as a marketplace item if `data/user-apps/marketplace.json` is not updated. This is a hard gate: do not skip the `marketplace.json` write in any phase.

### Phase 0 — Feasibility spike: Node middleware for non-standard verbs
*Blocks everything else.*
1. Add a throwaway `src/middleware.ts` with `export const config = { runtime: "nodejs", matcher: ["/api/vfs/webdav/:path*"] }` and a handler that returns `new Response("ok", { status: 207 })` for any method.
2. `npm run dev`; `curl -X PROPFIND http://localhost:3000/api/vfs/webdav/` and confirm `207` (not Next's `400`).
3. Confirm `request.body` in middleware is a readable stream that can be piped to `fs.createWriteStream` without full buffering (test with a >10 MB body and watch RSS).
4. Record the outcome in `plan.md` (this section) and either proceed with Phase 1+ or escalate per §1's fallback note.
- **Covers:** unblocks FR-001, FR-010, SC-010.

### Phase 1 — VFS streaming & copy
1. `src/os/fs-types.ts`: add optional `readStream?`, `writeStream?`, `copy?` to `FSBackend`.
2. `src/os/vfs.ts`: add `readStream`, `writeStream` (local backend: real streams; mounted backend: buffered fallback via existing `readBuffer`/`writeBuffer` if the backend doesn't implement the optional methods), `copy` (local: read+write; mounted: backend `copy` or reject).
3. `writeStream`'s local-backend implementation must use the same tmp-file+rename discipline as `writeFileAtomic` so a WebDAV `PUT` racing a BOS-side write can't corrupt data (edge case, Notes §Edge Cases).
- **Covers:** FR-009, FR-007 (atomicity for the streamed path), SC-006, edge case "large files stream," edge case "concurrent writes must not corrupt."

### Phase 2 — Path security
1. `src/lib/webdav/path-security.ts`: reject `\`, doubly-encoded/percent-encoded traversal (`%2e%2e`, `%2f`, `%5c`), `./`, `//` on the **raw** path segment *before* any URL-decoding is trusted, then call `jailResolve` (`src/os/path-jail.ts`) against the VFS root for the final check.
2. Symlink cycle detection: maintain a `Set<string>` of resolved real paths visited during a single `PROPFIND`/`GET` recursive walk; a repeat resolution → `508 Loop Detected`.
3. Unit-test every encoded variant from FR-017 individually (see Phase 8 / T2 in tasks.md).
- **Covers:** FR-005, FR-017, FR-018, SC-005, edge cases "path traversal," "symlink cycles."

### Phase 3 — Token lifecycle & bearer auth
1. `src/lib/webdav/tokens.ts`: `generateToken()`, `hashToken()`, `verifyToken(candidate, hash)`, `revokeToken(tokenId)`, `listTokens()` — built on `getSecretsStore()` namespace `"vfs-webdav"` per §3's chosen hashing approach.
2. `src/lib/webdav/auth.ts`: extract `Authorization: Bearer <token>`, hash + compare against stored tokens, return `{ ok: true }` or a `401` `Response` with `WWW-Authenticate: Bearer realm="BOS"`.
3. Wire the opt-in gate (no tokens ever generated → `404`) into `handler.ts` ahead of auth.
- **Covers:** FR-002, FR-003, FR-004, FR-012, SC-003, SC-004, User Story 2 (all 3 scenarios), User Story 3 scenarios 2–3.

### Phase 4 — Core WebDAV methods (read path)
1. `methods/options.ts`: `DAV: 1, 2`, `Allow: GET, PUT, DELETE, MKCOL, COPY, MOVE, PROPFIND, OPTIONS, HEAD`.
2. `methods/propfind.ts`: `Depth: 0` → own-resource `207`; `Depth: 1` → children via `vfs.list`, capped at 5000 entries (`431` if exceeded, per spec FR-016 — not `413`), XML per RFC 4918 (`displayname`, `getcontentlength`, `getlastmodified`, `getcontenttype`, `resourcetype`).
3. `methods/get.ts` / `methods/head.ts`: stream file content via `vfs.readStream`; directories get a minimal HTML/XML index (root `/` edge case included); `HEAD` returns identical headers with no body.
4. Root `/` special-cased to always return `200` with a minimal listing even before any subdirectory is requested.
- **Covers:** FR-001, FR-006, FR-010, FR-013, FR-015, FR-016, SC-001, SC-002, SC-010, User Story 6 scenarios 1–3, edge cases "HEAD," "root index," "PROPFIND bounding."

### Phase 5 — Mutating WebDAV methods (write path)
1. `methods/put.ts`: stream request body to `vfs.writeStream`; `201` for create, `204`/`200` for update.
2. `methods/delete.ts`: `vfs.remove`.
3. `methods/mkcol.ts`: `vfs.mkdir`.
4. `methods/copy.ts` / `methods/move.ts`: `vfs.copy` / `vfs.rename`, both server-side, both surfacing the existing cross-mount-boundary error as a client-facing `502`/`409` rather than crashing.
- **Covers:** FR-001, FR-007, FR-008, SC-002, User Story 6 scenarios 4–9, edge case "concurrent writes."

### Phase 6 — Errors & logging
1. `xml.ts`: `<d:error>` builder with `<status>`/`<message>` for every 4xx/5xx path.
2. `logging.ts`: `logger().info("vfs.webdav", ...)` per completed request (method, path, status, duration); `.debug` for writes/auth checks; `.warn`/`.error` for auth failures, traversal attempts, and all other error branches — never a swallowed `catch {}`.
- **Covers:** FR-014, FR-019.

### Phase 7 — Settings panel & marketplace packaging
This phase has a real gap to resolve, not just wire up: `src/components/apps/settings/ServiceConfigPanel.tsx` currently renders "Custom settings components aren't loadable yet" wherever a service declares `settingsRegistration.configApp` — so a marketplace item cannot yet ship a genuinely custom React settings component, only an auto-generated JSON-Schema form. FR-020 needs the former (token generation with copy-once display, OS-specific mount instructions).

**Decision:** ship the Local Mount panel as a `customComponent` entry in `src/lib/config/registry.ts` (same mechanism as `skills`/`integrations` today) rather than blocking this feature on building general `configApp` component-loading infrastructure. Register it under `Settings → Integrations → Local Mount` by adding the tab under the existing `integrations` custom component's sub-navigation, or as its own top-level registration if `Integrations` doesn't support sub-panels — confirm which during implementation and adjust `spec.md`'s Key Entities note if the final location differs.
1. `src/components/apps/settings/LocalMountTab.tsx`: token list with revoke buttons, "Generate token" (one-time display), mount instructions (macOS `mount_webdav`, Linux `davfs2`, including the `use_locks 0` note from spec Notes).
2. `src/app/api/config/route.ts` already dispatches by namespace via `getRegistration` — add a small `src/app/api/vfs/webdav/tokens/route.ts` (standard verbs only: `GET` list, `POST` generate, `DELETE` revoke) for the panel to call; this is a normal Next.js route (no non-standard verbs needed here).
3. Package `data/user-apps/items/vfs-webdav-mount/` per the `002-service-daemons` item layout (`services/`, `settings/`, `doc/`, `spec/`) so the feature is marketplace-installable per the spec's framing, even though the actual Settings UI is core-registered per the decision above — document this tension in the item's `doc/`.
- **Covers:** FR-020, User Story 3 (all 3 scenarios), User Story 4 (all 3 scenarios), Key Entities "Mount token," "Local Mount settings panel," "WebDAV service."

### Phase 8 — Deployment modes
1. **Local dev**: works once Phase 0–6 land; endpoint reachable at `http://localhost:3000/api/vfs/webdav`.
2. **Standalone Docker**: no bastion in front — bearer token is the only auth; confirm the middleware doesn't special-case `127.0.0.1`/loopback in a way that would leave the container's exposed port unauthenticated (the draft design's "auth middleware skips localhost" idea from `design.md` §4c is explicitly rejected — inside a container, `localhost` from the client's perspective is the *container*, and Docker port-forwarding means an external client's request also arrives looking like a normal request; there is no safe "trust loopback" shortcut here).
3. **Bastion**: the proxy injects `x-bos-username` (verified in `bastion/src/proxy.ts`), not `X-Forwarded-User`. Confirm during implementation whether WebDAV auth should (a) ignore this header and rely solely on the bearer token (simplest, ships now) or (b) additionally validate the header matches an expected user (deferred — no existing BOS-side code validates this header for any purpose today, so this would be new trust-boundary work). **Default to (a)** for this feature; note this explicitly as a scoping decision in `spec.md` if it doesn't already say so.
4. Mount instructions per mode surfaced in the Settings panel (Phase 7).
- **Covers:** FR-011, SC-008, SC-009, SC-010, User Story 4, User Story 5.

### Phase 9 — Testing & quality gate
See `tasks.md` T11–T14 for the full unit/e2e/manual breakdown and traceability matrix.
- **Covers:** SC-007 and, transitively, verification of every other SC via the tests that exercise it.

## 6. Assumptions & Dependencies (carried over, verified)

- Depends on `006-data-isolation`: all I/O goes through `src/os/vfs.ts`, never raw `fs` — enforced by construction since the WebDAV method handlers only ever call `vfs.*`.
- Depends on `007-gitfs`/SpecFS mounts: confirmed transparent — `vfs.ts`'s public functions already dispatch through `findMount()` for `/Specs`, `/Docs`, `/Templates`; the WebDAV layer needs no special-casing beyond the streaming/copy fallback noted in §2.
- `src/middleware.ts` uses Node.js middleware runtime (Phase 0 spike confirms feasibility); no `route.ts` handler is needed for `/api/vfs/webdav/[...path]`.
- Unit tests live in `tests/` (Playwright unit runner, `playwright.unit.config.ts`); e2e/HTTP-protocol tests live in `e2e/` (Playwright, `playwright.config.ts`) — corrected from the prior plan's "tests live at repository root `tests/` directory," which conflated the two suites.
- No existing harness drives a real `davfs2`/`mount_webdav` mount — that verification is manual (tasks.md T13), not automatable in CI as currently set up.
