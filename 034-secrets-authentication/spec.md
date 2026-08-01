# Feature Specification: Generic Service Secrets & Bastion Credential Routing

**Feature Branch**: `034-secrets-authentication`

**Created**: 2026-08-01

**Status**: Draft

**Input**: User description: "A generic, BOS-wide client/secret mechanism that any protocol or
service can use to authenticate headless (non-browser-session) clients. It must work identically
whether BOS runs standalone or behind the multi-user Bastion reverse proxy — including when
Bastion is configured with Keycloak — without giving Bastion an identity-provider admin API
credential, without a native OAuth-capable client per protocol, and without Bastion hardcoding any
per-protocol path knowledge."

## Background

BOS can run standalone (a single instance, no reverse proxy) or behind Bastion, a multi-user
reverse proxy that spawns one isolated container per user and normally routes requests by browser
session cookie. Some BOS services need to be reachable by clients that are not browsers and cannot
carry a session cookie at all — for example, a filesystem-mount client that only ever speaks HTTP
Basic auth. Such a service needs two things this codebase does not yet provide generically:

1. A way to issue and verify its own scoped, revocable credentials, independent of BOS's own
   interactive login.
2. When running behind Bastion, a way for Bastion to determine *which user's container* a headless,
   cookie-less request belongs to, so it can be routed — without Bastion needing to hardcode
   knowledge of that specific protocol's URL path, and without Bastion needing to ask the configured
   identity provider "who owns this credential" (some providers, notably an OIDC provider like
   Keycloak where Bastion holds no local user directory, cannot answer that question at all without
   a new privileged admin-API credential).

Two designs for closing gap 2 were considered and explicitly rejected for this feature:

- **Giving Bastion an identity-provider admin-API service account** (e.g. a Keycloak client with
  `view-users`) so it could resolve a bare username to an account. Rejected: it adds a new
  privileged, separately-managed credential and doesn't fit this system's "no extra services, all
  state lives in files" model — and it still only tells Bastion "does this username exist," not
  whether the presented credential is genuine.
- **A native, OAuth-capable client per protocol** (mirroring how tools like
  `google-drive-ocamlfuse`/`rclone` perform their own OAuth handshake and refresh tokens locally).
  Rejected: it requires shipping and maintaining a bespoke client binary instead of letting users
  mount with whatever stock client their protocol already has (e.g. `davfs2`, a native Finder
  WebDAV client, `curl`).

This feature defines the alternative: any BOS service mints its own scoped secret through one
shared, generic mechanism; Bastion resolves routing purely by hashing a presented credential and
looking it up in a small, non-reversible, per-user companion index it already has filesystem access
to — never by asking an identity provider who owns it, and never by hardcoding a path. Actual
authentication of the request always remains the target service's own responsibility, inside its
own container — Bastion only ever routes.

**This feature is self-contained.** It does not require any other feature to exist first. Where a
compatible headless-client consumer already exists elsewhere in the codebase, adopting this
mechanism is optional integration work (see User Story 6) — this spec's own core requirements,
acceptance criteria, and success metrics stand on their own and are verifiable with a
self-contained example service defined as part of delivering this feature.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A service mints and verifies its own scoped secrets (Priority: P1)

A BOS developer building any server-side feature needs to issue a revocable, per-account credential
that isn't the user's own login — for example, a scoped API token for a headless client. They call
one shared mechanism, giving it a `service` name of their choosing; they get back a secret shown
once, and can later verify a presented candidate, list issued secrets (without ever exposing the
raw value again), and revoke individually.

**Why this priority**: Everything else in this feature is built on top of this — without it there
is nothing to route to.

**Independent Test**: Call the mechanism directly (no HTTP, no Bastion involved) under a
test-only `service` namespace: create a secret, verify the correct raw value is accepted and an
incorrect one is rejected, list it back with metadata only (never the raw value), revoke it, and
confirm it's rejected afterward.

**Acceptance Scenarios**:

1. **Given** a freshly created secret for `service = "example-protocol"`, **When** the raw value is
   presented for verification, **Then** it is accepted.
2. **Given** that same secret, **When** a different, incorrect value is presented, **Then** it is
   rejected.
3. **Given** two different services each hold a secret, **When** a secret minted for one service is
   presented under the other service's namespace, **Then** it is rejected (namespace isolation).
4. **Given** a secret has been revoked, **When** its original raw value is presented again, **Then**
   it is rejected.

---

### User Story 2 - Standalone BOS authenticates a presented secret directly (Priority: P1)

An operator (or developer) runs BOS with no Bastion in front of it at all. A headless client
presents a secret previously minted through the mechanism in User Story 1. BOS authenticates it by
checking its own local, encrypted store directly — it has no dependency on Bastion, and no
dependency on the routing companion index described in User Story 3.

**Why this priority**: This is the majority deployment mode for local development, and the
simplest case that must never regress or gain an accidental dependency on Bastion-only machinery.

**Independent Test**: With no Bastion process running at all, mint a secret and verify it directly
against the running BOS instance.

**Acceptance Scenarios**:

1. **Given** no Bastion is involved, **When** a valid secret is presented to BOS directly, **Then**
   it is accepted using only BOS's own local store — the routing companion index is never consulted
   to make this decision.

---

### User Story 3 - Bastion routes a headless request behind Keycloak (Priority: P1)

An operator runs BOS behind Bastion configured with `AUTH_PROVIDER=keycloak`. A headless client
presents, via `Authorization: Basic <anything>:<secret>`, a secret that was minted by a user who is
fully authenticated through ordinary Keycloak SSO (e.g. via a browser session in Settings, or any
other in-container minting path). Bastion must route this request to that user's container —
without ever calling the identity provider to resolve a username, and without any admin-API client
configured anywhere in Bastion.

**Why this priority**: This is the scenario that is structurally impossible to support today
without the rejected admin-API dependency — it's the core reason this feature exists.

**Independent Test**: Stand up Bastion with `AUTH_PROVIDER=keycloak` and no Keycloak admin client
credentials configured anywhere. Using the example service from User Story 1, mint a secret for a
real, Keycloak-authenticated user; present it via Basic auth to Bastion; confirm it is routed to
that user's container.

**Acceptance Scenarios**:

1. **Given** a secret minted by a real, currently-authenticated user, **When** it is presented via
   Basic auth with no session cookie, **Then** Bastion resolves and routes the request to that
   user's container.
2. **Given** a value that was never minted as a secret by anyone, **When** it is presented via Basic
   auth, **Then** Bastion responds `401` with `WWW-Authenticate: Basic`, not a redirect to `/login`.
3. **Given** Bastion has no Keycloak admin-API client configured (the rejected approach from
   Background), **When** a valid secret is presented, **Then** routing still succeeds — it never
   depends on resolving a username through the identity provider.

---

### User Story 4 - Bastion routes a headless request under simple auth (Priority: P2)

The same routing behavior as User Story 3, but with Bastion's `simple` (local username/password)
auth provider instead of Keycloak.

**Why this priority**: Confirms the routing mechanism behaves identically across both supported
identity providers — ranked P2 only because, unlike Keycloak, this case has no structural blocker
to begin with; it's parity, not a new capability.

**Independent Test**: Repeat User Story 3's test with `AUTH_PROVIDER=simple`.

**Acceptance Scenarios**:

1. **Given** `AUTH_PROVIDER=simple` and a minted secret, **When** it is presented via Basic auth,
   **Then** routing behaves identically to the Keycloak case in User Story 3.
2. **Given** a secret is revoked, **When** it is subsequently presented, **Then** the request is
   rejected — checked at the authenticating container, not by Bastion (Bastion only routes).

---

### User Story 5 - A second, independent service adopts the mechanism with zero Bastion changes (Priority: P3)

A BOS developer adds a *second* service (distinct from whatever example was used to build/verify
this feature) that also needs headless Basic-auth clients. They call the same generic mechanism
under their own `service` namespace. They make no changes anywhere in Bastion's routing code.

**Why this priority**: This is the generalization payoff that justifies a shared mechanism instead
of a one-off. Ranked below the P1/P2 stories because it's a property proven by test, not a
user-facing capability on its own.

**Independent Test**: Introduce a second, independent test-only `service` namespace distinct from
the one used in User Story 1/3/4, mint a secret under it, present it via Basic auth to a path
Bastion has never seen before, and confirm it routes correctly with no Bastion code change beyond
what this feature itself introduces.

**Acceptance Scenarios**:

1. **Given** a brand-new service namespace with its own minted secret, **When** a Basic-auth
   request for an arbitrary, previously-unseen path arrives with no session cookie, **Then** it
   resolves and routes using the same generic lookup — the triggering condition is "Basic auth
   present, no session," never a path allowlist.
2. **Given** two different services each hold a secret with different raw values, **When** both are
   presented to Bastion (at different times), **Then** each resolves to its own correct
   service/user pairing, with no cross-service collision.

---

### User Story 6 - An existing headless-client consumer can adopt this mechanism (Priority: P3, optional/non-blocking)

If a compatible headless-client consumer already exists elsewhere in the codebase (for example, a
WebDAV mount feature with its own bespoke token store and its own protocol-specific Bastion routing
special-case), it can be migrated onto this mechanism as a drop-in integration, with no change to
its own user-facing behavior.

**Why this priority**: This is real-world motivation for building a generic mechanism rather than a
one-off, but it is explicitly **not required** for this feature's own completion or acceptance —
whether such a consumer exists, in what shape, and on what timeline is entirely outside this
feature's control. This story exists only to state that the mechanism must be adoptable without
breaking such a consumer's existing users, if and when that migration happens.

**Independent Test**: Not applicable to this feature's own acceptance — verified, if at all, by
whatever feature owns that consumer.

**Acceptance Scenarios**:

1. **Given** an existing consumer's own bespoke secret store, **When** it is migrated to call this
   mechanism under its own `service` namespace, **Then** its previously-issued credentials continue
   to authenticate correctly (or an explicit, documented migration step is provided if they cannot).

### Edge Cases

- A user's container has never been provisioned (no `data/` directory exists yet on disk) — the
  routing scan simply finds nothing there; this is not an error, just a non-match for that user.
- A user's routing-index file is missing, empty, or corrupted — that user is skipped during the
  scan; it must not abort or corrupt the scan for other users.
- A secret is revoked concurrently with an in-flight request using it — acceptable eventual
  consistency; the authenticating container's own check is the final word regardless of what
  Bastion's routing index says at the moment of the request.
- Revoking a secret must remove both its authoritative record and its routing-index entry — a stale
  index entry left behind after revocation is not a security issue (the authenticating container
  still rejects it) but is a hygiene bug this feature must avoid.
- The number of provisioned users grows over time — the routing scan is bounded by however many
  users Bastion has provisioned (operator-configurable cap), not by request volume; this must stay
  small enough that a linear scan per request remains acceptably fast (see Success Criteria).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST provide a generic secret-issuing mechanism, namespaced by an arbitrary
  `service` identifier, usable by any BOS server-side feature.
- **FR-002**: Secret material MUST be stored as a per-secret salted, slow (scrypt-class) hash inside
  an encrypted-at-rest store, scoped to the owning user's own data — never in plaintext, never
  shared across users.
- **FR-003**: Alongside the authoritative encrypted record, minting a secret MUST also write a
  minimal, non-reversible companion record (a fast, unsalted hash of the raw secret, plus which
  `service` it belongs to) to a plaintext file inside that same user's own data directory, so it can
  be read by a process that does not hold the encryption key.
- **FR-004**: Standalone BOS (no Bastion) MUST verify a presented secret directly against its own
  encrypted store and MUST NOT depend on the companion index to do so.
- **FR-005**: When running behind Bastion, a request with no session cookie but a parseable
  `Authorization: Basic` header MUST be routed by hashing the presented password and looking it up
  across every provisioned user's companion index — never by resolving the Basic-auth username
  against an identity provider.
- **FR-006**: This routing lookup MUST behave identically regardless of Bastion's configured
  identity provider. It MUST NOT require an identity-provider admin-API credential or any other new
  privileged credential.
- **FR-007**: Bastion's role in this flow is routing only — it MUST forward the presented secret to
  the resolved container (rewritten as a `Bearer` credential) and MUST NOT itself decide whether the
  secret is valid. The target container's own verification (FR-002's store) remains the sole source
  of truth for authentication.
- **FR-008**: The Bastion-side branch that triggers this routing MUST be based on the *shape* of the
  request (session-cookie absent, Basic auth present), not on an allowlist of paths. Adding a new
  service that uses this mechanism MUST require zero changes to Bastion's routing code.
- **FR-009**: Revoking a secret MUST remove both its encrypted record and its companion index entry.
- **FR-010**: The mechanism MUST be designed so that an existing headless-client consumer elsewhere
  in the codebase (if any) can adopt it as a drop-in caller, under its own `service` namespace,
  without changing its own user-facing behavior. This feature does not itself modify any such
  consumer as part of its core scope (see User Story 6).
- **FR-011**: The companion index MUST NOT reveal the raw secret, and MUST NOT be usable on its own
  to authenticate a request (it is a routing aid, not a credential store).

### Key Entities

- **ServiceSecret**: A generic, per-user, per-service credential. Attributes: an identifier, the
  `service` namespace it belongs to, a salted authoritative hash of the raw secret, an optional
  label, and a creation timestamp. Stored in a per-user encrypted store.
- **CredentialsIndexEntry**: A per-user, per-secret routing record. Attributes: a fast, unsalted
  hash of the raw secret (deliberately different from `ServiceSecret`'s authoritative hash — see
  plan.md for why), the `service` it belongs to, and a creation timestamp. Stored in a single
  plaintext file per user, readable by Bastion, never written to or read by any other user's
  container.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A secret minted by a real, Keycloak-authenticated user, with no Keycloak admin client
  configured anywhere in Bastion, is successfully routed by Bastion when presented via Basic auth.
- **SC-002**: The same scenario under `AUTH_PROVIDER=simple` behaves identically.
- **SC-003**: Adding a second, independent service that uses the generic mechanism requires zero
  additional lines changed in Bastion's routing code beyond what this feature itself introduces.
- **SC-004**: Bastion's routing code contains no protocol-specific constant or path check anywhere
  — the triggering condition is exclusively "no session, Basic auth present."
- **SC-005**: Revoking a secret makes it stop working for both authentication and Bastion-side
  routing within one request — no stale successful routing after revocation.
- **SC-006**: The per-request routing lookup on Bastion completes without perceptible added latency
  at the default provisioned-user scale (see plan.md's Scale/Scope).

## Assumptions

- Bastion has filesystem access to every provisioned user's own data directory (needed to read the
  per-user companion index) — this feature assumes that access already exists in whatever
  deployment it's used in; it does not introduce a new shared volume or mount.
- The raw secret values this mechanism issues are high-entropy, random, BOS-generated tokens (never
  user-chosen, low-entropy passwords) — this is what makes a fast, unsalted lookup hash acceptable
  for the companion index without reintroducing brute-force risk.
- OAuth device-code minting (for headless minting without an existing browser session) and a
  native OAuth-capable client are explicitly **out of scope** for this feature — considered and
  deferred as separate, optional future work, not required to satisfy any user story here.
- Giving Bastion an identity-provider admin-API service account is explicitly **out of scope** —
  this feature exists specifically so that dependency is never needed.
- Per-user data isolation (one user's container never reads or writes another user's data) is a
  hard constraint this feature must preserve; only the non-reversible companion index is
  cross-user-readable, and only by the routing layer — never by another user's own container.
- This feature does not assume any other feature (including any pre-existing WebDAV-mount-style
  feature) has been implemented, is in any particular state, or exists at all. Its own acceptance
  criteria are verifiable using a self-contained example service defined as part of delivering this
  feature (see plan.md/tasks.md).
