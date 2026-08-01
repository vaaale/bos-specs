# Tasks: VFS Mount (WebDAV)

See `plan.md` for architecture, module layout, and the rationale behind each decision referenced below. Task IDs are stable — use them in commits/PRs.

## T0 — Feasibility spike: Node middleware for non-standard HTTP verbs
**Blocks:** T5, T6, T8. **Goal:** confirm `src/middleware.ts` (Node.js runtime) can intercept `PROPFIND`/`MKCOL`/`COPY`/`MOVE` before Next's App Router rejects them, and can stream request bodies.
- [ ] Throwaway `src/middleware.ts` with `matcher: ["/api/vfs/webdav/:path*"]`, `runtime: "nodejs"`; return `207` for any method.
- [ ] `npm run dev`; `curl -X PROPFIND`/`MKCOL`/`COPY`/`MOVE` against it; confirm non-`400` responses.
- [ ] Pipe a >10 MB request body to `fs.createWriteStream` from inside middleware; confirm no full buffering (watch RSS).
- [ ] Record go/no-go outcome in `plan.md` §1; if no-go, stop and escalate (do not fall back to editing `tools/supervisor/supervisor.mjs`, which is explicitly off-limits).
- **Maps to:** unblocks FR-001, FR-010; SC-010.

## T1 — VFS streaming & copy primitives
**Goal:** give the WebDAV layer streaming read/write and a copy operation without buffering whole files.
- [ ] `src/os/fs-types.ts`: add optional `readStream?`, `writeStream?`, `copy?` to `FSBackend`.
- [ ] `src/os/vfs.ts`: add `readStream(vfsPath)`, `writeStream(vfsPath, stream)` for the default local backend (real `fs.createReadStream`/`fs.createWriteStream`); mounted paths without backend support fall back to buffered `readBuffer`/`writeBuffer` (documented limitation, not silently wrong).
- [ ] `writeStream`'s local implementation uses the same tmp-file+rename discipline as `writeFileAtomic` (`src/os/atomic-write.ts`) so a WebDAV `PUT` racing a BOS-side write can't corrupt the file.
- [ ] `vfs.copy(fromPath, toPath)`: local backend read+write; mounted backend uses `backend.copy` if present, else throws a clear "cannot copy across this mount" error (mirrors existing `rename` behavior).
- [ ] Unit test: `tests/webdav/vfs-streaming.test.ts` — write a >50 MB buffer via `writeStream`, assert peak RSS increase stays bounded (proxy for SC-006 at the VFS-layer level, before the HTTP layer exists).
- **Maps to:** FR-009, FR-007 (streamed-path atomicity); SC-006; Edge Cases "large files (>100MB) MUST stream," "files being written by BOS concurrently with a WebDAV PUT MUST not corrupt data."

## T2 — Path normalization & symlink-cycle security
**Goal:** reject every traversal/encoding trick before any I/O, and detect symlink cycles.
- [ ] `src/lib/webdav/path-security.ts`: reject raw (pre-decode) `\`, `%2e%2e`, `%2f`, `%5c`, `./`, `//`, then call `jailResolve` (`src/os/path-jail.ts`) as the final authority.
- [ ] Symlink cycle detection: `Set<string>` of resolved real paths visited per recursive `PROPFIND`/traversal; a repeat → `508 Loop Detected`.
- [ ] Unit test: `tests/webdav/path-security.test.ts` — one case per encoded variant in FR-017 (`%2e%2e`, `%2f`, `%5c`, `./`, `//`, `\`), each asserting rejection; plus a symlink-cycle fixture asserting `508`.
- **Maps to:** FR-005, FR-017, FR-018; SC-005; Edge Cases "requests targeting paths outside the VFS root," "symlinks... cycles."

## T3 — Mount token lifecycle & storage
**Goal:** generate, hash, verify, list, and revoke bearer tokens, encrypted at rest.
- [ ] Resolve the bcrypt-vs-`crypto.scrypt` decision from `plan.md` §3 with the user before writing this task (default: `crypto.scrypt`, no new dependency).
- [ ] `src/lib/webdav/tokens.ts`: `generateToken()` (`crypto.randomBytes(32)`, base64url), `hashToken()`/`verifyToken()`, `listTokens()`, `revokeToken(tokenId)` — backed by `getSecretsStore()` (`src/lib/integrations/secrets/store.ts`), namespace `"vfs-webdav"`.
- [ ] Multiple tokens coexist as independent keys; revoking one must not touch the others.
- [ ] Unit test: `tests/webdav/tokens.test.ts` — generate → verify succeeds; revoke → verify fails; generate two, revoke one, confirm the other still verifies; confirm the raw token is never written to disk (only its hash).
- **Maps to:** FR-003, FR-004; Key Entity "Mount token"; User Story 3 scenario 3.

## T4 — Bearer authentication & opt-in gate
**Goal:** enforce auth on every WebDAV request and keep the endpoint invisible until a token has ever been generated.
- [ ] `src/lib/webdav/auth.ts`: parse `Authorization: Bearer <token>`, verify against T3's store; on failure return `401` with `WWW-Authenticate: Bearer realm="BOS"`.
- [ ] Opt-in gate in `src/lib/webdav/handler.ts`: if `listTokens()` is empty, short-circuit `404` before auth runs at all.
- [ ] Unit test: `tests/webdav/auth.test.ts` — no header → `401`; valid token → passes; revoked token → `401`; zero tokens ever generated → `404` regardless of header.
- **Maps to:** FR-002, FR-003, FR-012; SC-003, SC-004; User Story 2 (all 3 scenarios); User Story 3 scenario 2; User Story 5 edge case "WebDAV service is not enabled → 404/503"; User Story 6 scenario 10.

## T5 — Read-path WebDAV methods (OPTIONS, PROPFIND, GET, HEAD)
**Depends on:** T0, T1, T2, T4. **Goal:** implement every read-side method per RFC 4918.
- [ ] `methods/options.ts`: `DAV: 1, 2`; `Allow: GET, PUT, DELETE, MKCOL, COPY, MOVE, PROPFIND, OPTIONS, HEAD`.
- [ ] `methods/propfind.ts`: `Depth: 0` → own-resource `207`; `Depth: 1` → children (`displayname`, `getcontentlength`, `getlastmodified`, `getcontenttype`, `resourcetype`), capped at 5000 entries with `431` (not `413`) if exceeded.
- [ ] `methods/get.ts`: streams file content via `vfs.readStream` with correct `Content-Type`; for a directory, returns a minimal HTML/XML listing (root `/` included, even with no explicit sub-path requested).
- [ ] `methods/head.ts`: identical headers to `GET`, no body.
- [ ] Unit tests: `tests/webdav/headers.test.ts` (DAV/Allow/Content-Type on each method); `tests/webdav/propfind.test.ts` (depth 0 vs 1, >5000-child truncation → `431`).
- **Maps to:** FR-001, FR-006, FR-010, FR-013, FR-015, FR-016; SC-001, SC-002, SC-010; User Story 6 scenarios 1–3; Edge Cases "HEAD requests," "root `/` MUST return 200," "PROPFIND Depth:1... bounded."

## T6 — Write-path WebDAV methods (PUT, DELETE, MKCOL, COPY, MOVE)
**Depends on:** T0, T1, T2, T4. **Goal:** implement every mutating method, server-side and atomic.
- [ ] `methods/put.ts`: streams request body to `vfs.writeStream`; `201` on create, `200`/`204` on update.
- [ ] `methods/delete.ts` → `vfs.remove`.
- [ ] `methods/mkcol.ts` → `vfs.mkdir`.
- [ ] `methods/copy.ts` → `vfs.copy`; `methods/move.ts` → `vfs.rename` — both server-side (no client round-trip), both surfacing a cross-mount-boundary error as a client-facing `409`/`502`, not a crash.
- [ ] Unit test: `tests/webdav/write-methods.test.ts` — PUT create + update, DELETE, MKCOL, COPY, MOVE, each asserted against the real VFS state afterward.
- **Maps to:** FR-001, FR-007, FR-008; SC-002; User Story 1 scenarios 2–3; User Story 6 scenarios 4–9; Edge Case "files being written by BOS concurrently with a WebDAV PUT."

## T7 — RFC 4918 error responses & structured logging
**Goal:** every error path returns compliant XML and is logged, never swallowed.
- [ ] `src/lib/webdav/xml.ts`: `<d:error>` builder with `<status>`/`<message>` for every 4xx/5xx.
- [ ] `src/lib/webdav/logging.ts`: `logger()` (`@/lib/logging`) wired with `COMPONENT = "vfs.webdav"` — `info` per completed request (method, path, status, duration), `debug` for writes/auth checks, `warn`/`error` for auth failures, path-traversal attempts, and every other error branch; audit every `catch` block added in T1–T6 for a matching log call.
- [ ] Unit test: `tests/webdav/errors.test.ts` — assert XML shape for a 401, a 403, a 404, and a 500; assert a log record is emitted for each.
- **Maps to:** FR-014, FR-019.

## T8 — Settings panel: Local Mount token management
**Depends on:** T0, T3, T4. **Goal:** let the BOS user generate/revoke tokens and see mount instructions, per `plan.md` §Phase 7's `customComponent` decision (registry-based, not the not-yet-built `configApp` mechanism).
- [ ] `src/app/api/vfs/webdav/tokens/route.ts` (standard verbs: `GET` list, `POST` generate, `DELETE` revoke) calling into T3.
- [ ] `src/components/apps/settings/LocalMountTab.tsx`: token list + revoke buttons, "Generate token" with one-time display, macOS/Linux mount commands including the `use_locks 0` `davfs2.conf` note (spec Notes).
- [ ] Register the tab in `src/lib/config/registry.ts` and `src/apps/settings/index.tsx`'s `CUSTOM_TABS` map, surfaced under **Settings → Integrations → Local Mount** (or confirm and document the actual final location if `Integrations` doesn't support sub-panels — see `plan.md` §Phase 7).
- [ ] Per-deployment-mode instructions (bastion HTTPS + SSO note, standalone Docker IP/port + bearer, local dev localhost) sourced from T10.
- [ ] E2E test: `e2e/vfs-webdav-settings.spec.ts` — generate a token via the UI, confirm one-time display, revoke it via the UI, confirm the API rejects it afterward.
- **Maps to:** FR-020; Key Entity "Local Mount settings panel"; User Story 3 (all 3 scenarios); User Story 4 (all 3 scenarios).

## T9 — Marketplace item packaging
**Goal:** ship the feature as an installable service item per the `002-service-daemons` layout, satisfying the spec's "marketplace-distributed service item" framing.

**Mandatory deliverable — marketplace.json:** This feature IS a BOS Marketplace item. It does NOT ship as a marketplace item unless `data/user-apps/marketplace.json` is updated to register the `vfs-webdav-mount` item. This is a hard gate — do not mark this task as done without confirming `marketplace.json` has been updated.

**Mandatory — unit tests:** Unit tests are REQUIRED. Every task producing source code must include its corresponding `tests/webdav/*.test.ts` file with full coverage of the requirements it implements. Unit tests are not optional — they are a mandatory deliverable of every implementing task.

- [ ] `data/user-apps/items/vfs-webdav-mount/services/vfs-webdav-mount/service.json` (manifest per `src/core/service/types.ts`'s `ServiceManifest`).
- [ ] `settings/`, `doc/`, `spec/` subfolders per the item layout; `doc/` explicitly notes that the Settings UI is core-registered (T8) rather than loaded from this item's `settings/` folder, and why (the `configApp` component-loading gap — see `plan.md` §Phase 7).
- [ ] **Register the item in `data/user-apps/marketplace.json`** — merge, not regenerate, per `034-user-apps-marketplace-parity`. Verify the entry is present after the merge.
- [ ] **Unit test:** `tests/webdav/item-packaging.test.ts` — assert the marketplace item layout is complete (all subfolders present, `service.json` is valid JSON, `doc/` contains the required notes).
- **Maps to:** Key Entity "WebDAV service"; User Story 1 independent test (mount + verify via Files app).

## T10 — Deployment mode wiring & mount instructions
**Depends on:** T4. **Goal:** the endpoint is reachable and correctly authenticated in all three deployment modes.
- [ ] Local dev: verify `http://localhost:3000/api/vfs/webdav` is reachable end-to-end once T0–T7 land.
- [ ] Standalone Docker: verify bearer-token auth is the *only* gate (explicitly reject any "trust loopback" shortcut — see `plan.md` §Phase 8.2) and the endpoint is reachable from outside the container on the published port.
- [ ] Bastion: confirm current behavior against the real header `x-bos-username` (`bastion/src/proxy.ts`) rather than the draft's invented `X-Forwarded-User`; per `plan.md` §Phase 8.3, ship auth relying solely on the bearer token (default decision) and document the deferred header-validation option.
- [ ] Write the three sets of mount instructions consumed by T8's Settings panel.
- **Maps to:** FR-011; SC-008, SC-009, SC-010; User Story 4 (all 3 scenarios); User Story 5 (all 4 scenarios, including the "not enabled → 404/503" edge case, cross-referenced with T4).

## T11 — Unit test sweep & completeness check
**Goal:** confirm the unit suites from T1–T7 collectively cover every unit-testable requirement; fill any gaps found.
- [ ] Run `npm run test:unit` for all `tests/webdav/*.test.ts` added above; fix failures.
- [ ] Gap check against FR-001–FR-020: confirm each has at least one assertion somewhere in `tests/webdav/`; add any missing case directly (e.g. FR-010's `Expect: 100-continue` handling, if not already covered in T5).
- **Maps to:** SC-007 (partial — `tsc`/`lint` covered in T14); consolidates FR coverage from T1–T7.

## T12 — E2E test sweep: full WebDAV protocol over HTTP
**Depends on:** T0, T4, T5, T6, T7. **Goal:** exercise the real running server (not mocked VFS) across the full method/scenario matrix.
- [ ] `e2e/vfs-webdav.spec.ts` using `./fixtures`' `request` fixture (no browser needed for raw HTTP-verb checks): `OPTIONS`, `PROPFIND` (depth 0 & 1, including >5000-child truncation), `GET`, `HEAD`, `PUT` (create + update), `DELETE`, `MKCOL`, `COPY`, `MOVE`, each against a real dev-server instance.
- [ ] `e2e/vfs-webdav-auth.spec.ts`: no-auth `401`, valid-token `207`/`200`, revoked-token `401` without restart, zero-tokens `404`.
- [ ] `e2e/vfs-webdav-security.spec.ts`: path-traversal attempts (all FR-017 encodings) → `403`; symlink cycle fixture → `508`.
- [ ] `e2e/vfs-webdav-streaming.spec.ts`: `PUT` a 200 MB generated payload, assert response success and (via a supervisor/process RSS sample before/after) peak RSS increase < 50 MB.
- [ ] `e2e/vfs-webdav-roundtrip.spec.ts`: `PUT` a file via the API, confirm it's visible via `GET /api/fs?op=read` (BOS's own Files-app-backing endpoint) and vice versa — the closest automatable proxy for User Story 1's "visible in BOS's Files app" without a real `davfs2` mount.
- **Maps to:** SC-001 (proxy only — real mount is T13), SC-002, SC-003, SC-004, SC-005, SC-006; User Story 1 scenarios 1–3 (proxy); User Story 6 (all 10 scenarios); Edge Cases (all 7).

## T13 — Manual cross-platform mount verification
**Goal:** verify the parts no CI harness can drive — a real `davfs2`/`mount_webdav` mount from an actual client.
- [ ] Install `davfs2` on a Linux test machine; mount `http://<host>:<port>/api/vfs/webdav` with a generated token; create/edit/delete a file locally; confirm the change appears in BOS's Files app within the `davfs2` cache TTL (≤2s typical).
- [ ] On macOS, use Finder → Go → Connect to Server with the same URL; repeat the same read/write/delete check.
- [ ] Repeat the mount from an external machine against each of the three deployment modes (local dev, standalone Docker, Docker Compose + Bastion) per T10.
- [ ] Record results (pass/fail per platform × mode) in the PR description; this task has no automated pass/fail gate.
- **Maps to:** SC-001, SC-002, SC-008, SC-009, SC-010; User Story 1 (both parts of its Independent Test); User Story 5 (Independent Test).

## T14 — Documentation & quality gate
**Goal:** land the feature cleanly per repo conventions.
- [ ] `npx tsc --noEmit` and `npm run lint` pass for all changed files.
- [ ] Update `docs/dev/architecture-overview.md` (or a new `docs/dev/vfs-webdav.md`) describing the middleware-based transport decision from `plan.md` §1, since it's a pattern future features may want to reuse or avoid.
- [ ] Update `docs/usage/` with end-user mount instructions (mirrors T8's in-app copy).
- **Maps to:** SC-007.

---

## Traceability matrix

Every functional requirement, success criterion, user-story scenario, and edge case, mapped to the task(s) that implement and verify it.

### Functional Requirements

| FR | Task(s) |
|---|---|
| FR-001 (WebDAV endpoint, all verbs, 1:1 VFS mapping) | T0, T5, T6 |
| FR-002 (Bearer auth, 401 challenge) | T4 |
| FR-003 (token generation/hash/storage) | T3 |
| FR-004 (multiple independently-revocable tokens) | T3 |
| FR-005 (path traversal → 403) | T2 |
| FR-006 (PROPFIND Depth:1 → 207) | T5 |
| FR-007 (PUT/MKCOL via VFS write path, atomicity) | T1, T6 |
| FR-008 (COPY/MOVE server-side, atomic) | T6 |
| FR-009 (streaming, no full buffering) | T1 |
| FR-010 (`DAV: 1,2`, `Expect: 100-continue`) | T0, T5 |
| FR-011 (three deployment modes) | T10 |
| FR-012 (opt-in 404/503 when no token) | T4 |
| FR-013 (HEAD parity with GET) | T5 |
| FR-014 (logging levels, never swallow errors) | T7 |
| FR-015 (PROPFIND Depth:0 → 207) | T5 |
| FR-016 (PROPFIND bounded at 5000, 431) | T5 |
| FR-017 (encoded traversal variants rejected) | T2 |
| FR-018 (symlink cycle → 508) | T2 |
| FR-019 (RFC 4918 XML error bodies) | T7 |
| FR-020 (Settings panel for token lifecycle) | T8 |

### Success Criteria

| SC | Task(s) |
|---|---|
| SC-001 (davfs2 + mount_webdav mount succeeds) | T12 (proxy), T13 (real) |
| SC-002 (file change visible within poll cycle) | T6, T12, T13 |
| SC-003 (401 vs 207 on auth) | T4, T12 |
| SC-004 (revoked token → 401, no restart) | T3, T4, T12 |
| SC-005 (traversal → 403, nothing read outside VFS) | T2, T12 |
| SC-006 (200MB PUT, <50MB RSS increase) | T1, T12 |
| SC-007 (tsc/lint pass) | T11, T14 |
| SC-008 (Bastion mode reachable via SSO) | T10, T13 |
| SC-009 (standalone Docker reachable via bearer) | T10, T13 |
| SC-010 (local dev reachable on localhost) | T0, T10, T13 |

### User Stories (acceptance scenarios)

| Story | Task(s) |
|---|---|
| US1 — mount as local drive (3 scenarios) | T5, T6, T9, T12, T13 |
| US2 — auth protects the mount (3 scenarios) | T4, T12 |
| US3 — token lifecycle exposed to user (3 scenarios) | T3, T8 |
| US4 — external users get mount instructions (3 scenarios) | T8, T10 |
| US5 — endpoint reachable externally in 3 modes + opt-in edge case (4 scenarios) | T4, T10, T13 |
| US6 — all HTTP verbs behave correctly (10 scenarios) | T5, T6, T12 |

### Edge Cases

| Edge case | Task(s) |
|---|---|
| PROPFIND Depth:1 bounded at 5000 → 431 (FR-016) | T5 |
| Path traversal (`../`, encoded) → 403 | T2 |
| Symlink cycles → 508 | T2 |
| Concurrent BOS write + WebDAV PUT must not corrupt | T1, T6 |
| Files >100MB must stream, not buffer | T1, T12 |
| HEAD parity with GET, no body | T5 |
| Root `/` → 200 with minimal listing | T5 |

### Key Entities

| Entity | Task(s) |
|---|---|
| Mount token | T3 |
| WebDAV service | T0, T9 |
| Deployment modes | T10 |
| Local Mount settings panel | T8 |
