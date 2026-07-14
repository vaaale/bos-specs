# Tasks: 026 Multi-User Usability

Feature branch: `026-multiuser-usability`. Tasks are grouped by phase; `[P]` marks work that can proceed in parallel with siblings.

## Phase A — Bastion foundations (logging + first-run) — P1

- [ ] A1. Add a persistent per-user log store to the Bastion (`bastion/src/log-store.ts`): append-only `{dataDir}/logs/<username>.log`, with `append`, `read(tail?)`, and `list` operations. Node built-ins only.
- [ ] A2. Wire `lifecycle.ts` `log()`/`updateState()` and `provision.ts` steps to also append to the per-user log (progress + errors with stack).
- [ ] A3. Extend `getInstanceState`/admin+account state responses with `error` reason and add endpoints to fetch the per-user log (`GET /admin/instances/:user/log`, `GET /account/log`).
- [ ] A4. First-run bootstrap (simple auth): detect "no admin" in `SimpleProvider`; add `GET /setup` state + `POST /setup` (create admin from `ADMIN_USER`, race-safe) in a new setup router; route unauthenticated entry to the setup SPA page when bootstrap is pending.
- [ ] A5. First-run landing (Keycloak): setup/landing page explaining IdP ownership + OIDC login link.
- [ ] A6. Bastion `typecheck` green.

## Phase B — Bastion admin portal — P1

- [ ] B1. Introduce Tailwind (or a small shared component set) into `bastion/ui`; establish layout/theme + shared primitives (Button, Card, Table, Dialog, Toast, LogView).
- [ ] B2. Rebuild `Login` + add `Setup` pages with the new UI.
- [ ] B3. Users view: create/remove/toggle-admin; remove flows through confirm → stop/remove container → wipe `user-data/<user>` (admin API `DELETE /admin/users/:user` with `wipeData` default true here).
- [ ] B4. Images view: `dockerode` build with streamed log (SSE or chunked) — new `POST /admin/image/build` (stream) + `GET /admin/images`; set-active-tag writes `bosImage` via `saveConfig`.
- [ ] B5. Containers view: list all (`listBosContainers` + state), Start/Stop/Kill actions (add `POST /admin/instances/:user/kill`).
- [ ] B6. Per-user log viewer wired to A3.
- [ ] B7. Bastion `typecheck` green.

## Phase C — Bastion account page — P1

- [ ] C1. Account page redesign with the Phase B UI.
- [ ] C2. Password set/change (simple auth only; hidden for Keycloak) — reuse `POST /account/password`.
- [ ] C3. Profile image: `POST /account/avatar` (validated upload) storing `{dataDir}/avatars/<username>`, `GET /avatar/:username` served by Bastion, bundled default fallback.
- [ ] C4. Lifecycle: Start/Stop/Restart + Re-provision + "Open my BrowserOS" (proxy to `/`).
- [ ] C5. Wipe data behind a strongly-worded confirm dialog (stop → wipe `data/` → offer restart).
- [ ] C6. Bastion `typecheck` green.

## Phase D — BOS run_command backend selection — P1/P2

- [ ] D1. `loadRcConfig()` / `registry.ts`: detect Bastion mode (`BOS_PUBLIC_PORT`) → force `backend: local`; hide docker-only fields.
- [ ] D2. Custom Command Execution settings component: standalone shows image picker (list local images) + build-from-Dockerfile (default `docker/run-command/Dockerfile`) with streamed log; Bastion mode shows the simplified local view.
- [ ] D3. BOS API routes (server-only): `GET /api/run-command/images` (list) and `POST /api/run-command/image/build` (streamed build), guarded (single build, docker-availability check).
- [ ] D4. Typecheck + lint green.

## Phase E — BOS dev-harness credentials — P2

- [ ] E1. Extend `dev-harness` registry schema + `harness-config.ts` with Claude/OpenCode credential material (secret, write-only).
- [ ] E2. On save, write credentials into a dedicated harness `HOME` (Claude `~/.claude`, OpenCode `auth.json`); update `envForCwd()` in `claude-runner.ts` to set `HOME`/env.
- [ ] E3. DevHarnessTab UI: credential inputs (write-only, set/unset indicator) + container guidance.
- [ ] E4. Typecheck + lint green.

## Phase F — BOS toolbar My profile — P2

- [ ] F1. `Topbar.tsx`: add a "My profile" control (multi-user only) linking to `/app/account`, showing the avatar (`/avatar/<me>`).
- [ ] F2. Typecheck + lint green.

## Cross-cutting

- [ ] X1. Update the user-container `Dockerfile` (root) to bundle run_command runtimes + Claude/OpenCode CLIs (FR-024).
- [ ] X2. E2E/manual verification per Success Criteria SC-001…SC-008.

## Closeout (after implementation)

- [ ] Z1. Merge 026 into `024-docker-multiuser` (admin portal, account UX, first-run, per-user logging, image build, container kill) and `019-tools-and-sandbox` (local backend in Bastion; image convergence); note changes in `discrepancies.md`; add 026 row to `overview.md`.
- [ ] Z2. Update documentation (`docs/dev/**`, `docs/usage/**`).
