# Tasks: VFS Mount (WebDAV)

**Source**: `/Specs/bos-system-specs/033-vfs-mount/spec.md`

**Estimated duration**: 1.5 days

---

## Task 1: WebDAV Service Implementation

**User Stories**: 1, 5, 6

**Description**: Implement the WebDAV service that exposes the VFS as a local-mountable drive via RFC 4918.

**Requirements**:
- Service runs as a plain Node worker thread on a configurable host/port
- Handles all required HTTP methods: `OPTIONS`, `PROPFIND` (depth 0 and 1), `GET`, `PUT`, `DELETE`, `MKCOL`, `COPY`, `MOVE`, `HEAD`
- Routes all HTTP methods to a single path prefix (`/api/vfs/webdav/[...path]`)
- Returns proper WebDAV headers (`DAV: 1, 2`, `Allow:`, `Content-Type`, `Content-Length`)
- Supports three deployment modes: Docker Compose + Bastion, Standalone Docker, Local dev
- Path traversal (`../`, encoded variants, `./`, `//`, Windows separators) must be rejected with `403`
- `PROPFIND` depth-1 responses bounded at 5000 children

**Tests**:
- [ ] Mount the WebDAV endpoint with `davfs2` or macOS Finder; open a file in a local editor, save it, then verify the change is visible inside BOS's Files app.
- [ ] External user on Linux/macOS can mount the VFS via WebDAV in three deployment modes.
- [ ] Each HTTP method (GET, PUT, DELETE, MKCOL, COPY, MOVE, PROPFIND, OPTIONS) behaves correctly according to WebDAV (RFC 4918).

---

## Task 2: Authentication & Token Management

**User Stories**: 2, 3

**Description**: Implement authentication and token lifecycle for the WebDAV service.

**Requirements**:
- All requests require `Authorization: Bearer <token>`
- Unauthenticated requests receive `401` with `WWW-Authenticate: Bearer realm="BOS"`
- Tokens are stored hashed in the service's own config with AES-256-GCM encryption at rest
- The Settings panel provides:
  - Token generation (cryptographically random 256-bit values)
  - Token revocation (independent, no restart needed)
  - Mount instructions for macOS (`mount_webdav`) and Linux (`davfs2`)
  - The `use_locks 0` note for `davfs2` config

**Tests**:
- [ ] Attempt a `PROPFIND` request with no credentials; confirm 401. Then retry with a valid token; confirm 207.
- [ ] Generate a token via the Settings panel; use it to mount from an external machine; revoke it; confirm the mount stops responding.

---

## Task 3: Settings Panel UI

**User Stories**: 4

**Description**: Build the Settings panel UI for the WebDAV service.

**Requirements**:
- Loaded at `Settings → Integrations → Local Mount`
- Provides token management (generate/rotate/revoke)
- Shows mount instructions for macOS and Linux
- Displays the `use_locks 0` note for `davfs2`

**Tests**:
- [ ] External user can discover the WebDAV URL and authentication method for each deployment mode.

---

## Task 4: Testing & Integration

**Description**: Verify the complete feature works end-to-end.

**Requirements**:
- Run integration tests against a WebDAV client (`curl` or mock)
- Verify all user stories pass
- Confirm the service works across all three deployment modes
- Test path traversal rejection
- Test `PROPFIND` depth-1 bounding at 5000 children

**Tests**:
- All tests from Tasks 1, 2, and 3

---

## Notes

- All I/O goes through the VFS layer via HTTP API (`/api/fs` and `/api/fs/raw`)
- No `src/` involvement required
- The service must be opt-in (no token = 404)
- The service must stream file reads/writes (no full-file memory buffering)
