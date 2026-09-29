# Tasks: Self-Heal "Pin & open preview" performs the Supervisor work

**Input**: `design.md` (authoritative) + `spec.md` + `plan.md`, all in `Specs/user-specs/self-modification/032-self-heal-pin/`
**Branch**: `bos/032-self-heal-pin`
**Prerequisites**: plan.md (required), spec.md (required for user stories)

**Tests**: Requested — the spec's testing approach is TDD on the repo's Playwright unit runner (`npm run test:unit`), and `approve-design` delegates as TDD. Tests come before implementation.

**Scope contract (from plan.md — the Developer may touch only these files):**
- `src/apps/build-studio/selfheal/PreviewStatus.tsx` (MODIFIED — the handler + its imports)
- `tests/self-heal/preview-pin.test.ts` (NEW)

Anything outside this list requires coming back to the spec, not improvising.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story the task belongs to (US1, US2)
- Exact file paths included in every task

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Nothing to initialize — this is a modification to an existing built-in app and existing test suite. Recorded here so the Developer does not reach for `manifest.ts`/`index.tsx`/`gen:apps` (none of which change for this fix).

- [ ] T001 Confirm the app is unchanged in structure — `src/apps/build-studio/selfheal/` already exists with `manifest.ts`/`index.tsx`; do NOT create or register anything. Verify `supervisorPost` is exported from `src/lib/supervisor/client.ts` and reuse it (FR-006); do NOT add a new endpoint or client.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The unit test for the handler's decision matrix. Written first so it FAILS before the implementation exists (TDD). It exercises the `activate` → `pin` → reload contract from the spec's edge cases. Per the finalized design (§7), the decision logic lives in a **pure, framework-free function** inside `PreviewStatus.tsx`, and the test asserts on that function by **stubbing `globalThis.fetch`** (so the real `supervisorPost` wrapper receives the shaped JSON) and **`globalThis.window.location.reload`** — the repo's `test:unit` runner is browser-less (no module mocking, no React renderer), so the test does NOT mock `supervisorPost` or render the component.

**⚠️ CRITICAL**: No user-story implementation until this test is written and observed failing.

- [ ] T002 Write `tests/self-heal/preview-pin.test.ts` covering the handler decision matrix. Stub `globalThis.fetch` (the real `supervisorPost` in `@/lib/supervisor/client` is a thin `fetch`+`r.json()` wrapper, so stubbing fetch drives it) and `globalThis.window.location.reload`; assert against the **pure decision function** in `PreviewStatus.tsx` (do NOT mock `supervisorPost` or render the component — the runner can't). Cases (from spec edge cases):
  - (a) `activate` returns not-`ready` (`state: "building"` or `ok: false`) ⇒ assert `pin` is NOT called and `reload` is NOT called.
  - (b) `activate` returns `state: "ready"`, `pin` returns `ok: true` ⇒ assert `pin` called with `{ version: "preview", branch }` and `reload` called exactly once.
  - (c) `activate` `ready` but `pin` returns `ok: false` ⇒ assert `reload` NOT called and the in-progress flag is cleared (button retryable).
  - (d) missing `activeFeatureBranch` ⇒ assert no `supervisorPost` call at all (defensive guard).
  Run `npm run test:unit` and confirm it FAILS (the handler does not yet do this). 

**Checkpoint**: Foundation ready — the failing test defines the contract.

---

## Phase 3: User Story 1 — The button pins and opens the preview (Priority: P1) 🎯 MVP

**Goal**: Clicking **Pin & open preview** on a `preview-ready` case issues `activate` (idempotent) and, once `ready`, pins the preview and reloads the window so the top toolbar shows `PREVIEW` on the case's branch — with no Settings window opening (FR-001, FR-002, SC-001, SC-003).

**Independent Test**: With a `preview-ready` case whose preview is `ready`, clicking the button flips the top toolbar to `PREVIEW` on the case's branch; no Settings app opens. (Covered by unit test T002 case (b) + the manual check below.)

### Implementation for User Story 1

- [ ] T003 [US1] In `src/apps/build-studio/selfheal/PreviewStatus.tsx`, add an in-progress state: `import { useState } from "react"` and `const [opening, setOpening] = useState(false)`.
- [ ] T004 [US1] In `src/apps/build-studio/selfheal/PreviewStatus.tsx`, remove the Settings navigation dependency: drop `import { useOSStore } from "@/store/os-provider"` and the `const launch = useOSStore((s) => s.launch)` line (FR-001 — the button no longer navigates).
- [ ] T005 [US1] In `src/apps/build-studio/selfheal/PreviewStatus.tsx`, add `import { supervisorPost } from "@/lib/supervisor/client"` (FR-006) and define the async `pinAndOpenPreview` handler: guard `if (!branch) return;`; `setOpening(true)`; `const a = await supervisorPost("activate", { branch })`; `if (a.ok === false || a.state !== "ready") return;` (FR-003); `const p = await supervisorPost("pin", { version: "preview", branch })`; `if (p.ok) window.location.reload();`; `finally { setOpening(false); }` (FR-002, FR-004). Extract this sequence into a **pure, framework-free function** (e.g. `async function pinAndOpen(branch, post, reload)`) so the unit test can call it directly with `globalThis.fetch`/`reload` stubs — the `onClick` handler then calls it with the real `supervisorPost` and `window.location.reload`. (See `design.md` §7 — the browser-less runner cannot mock `supervisorPost` or render React.)
- [ ] T006 [US1] In `src/apps/build-studio/selfheal/PreviewStatus.tsx`, rewire the button: `onClick={() => void pinAndOpenPreview()}`, `disabled={opening}`, add `disabled:opacity-50` to its className, and render `{opening ? "Opening preview…" : "Pin & open preview"}` (FR-004). **Rename `data-testid="self-heal-open-versions"` → `data-testid="self-heal-pin-preview"`** — the old name no longer describes the behaviour (the button no longer opens the versions/Settings view). Verified un-referenced by any test, so the rename is safe.
- [ ] T007 [US1] Verify the FR-024 footnote beneath the button is preserved verbatim (FR-005) — it must still state Promote/Discard live in the Topbar and self-healing never promotes. Do not add a second Promote/Discard control.
- [ ] T008 [US1] Run `npm run test:unit` and confirm T002 case (b) (and the US1-related cases) now PASS.

**Checkpoint**: User Story 1 functional and independently testable.

---

## Phase 4: User Story 2 — A not-built / stopped preview is built through the Supervisor (Priority: P2)

**Goal**: On a `preview-ready` case whose preview is `not-built`/`stopped` (e.g. after a Supervisor restart), the click triggers the build via `activate` and does NOT falsely pin/reload until the preview is `ready` (FR-003, SC-002). This is the same handler from US1 — it is already structured so a non-`ready` `activate` short-circuits.

**Independent Test**: With a `preview-ready` case whose preview is `not-built`/`stopped`, the click calls `activate`, does not pin, and does not reload; once the preview becomes `ready`, a subsequent click pins (US1 case (b)).

### Tests for User Story 2 ⚠️

- [ ] T009 [P] [US2] Confirm `tests/self-heal/preview-pin.test.ts` case (a) — `activate` not-`ready` ⇒ no `pin`, no `reload` — is present and PASSING (it is the US2 behaviour; add it here if T002 did not already cover a `building` return distinct from an `ok:false` return). It is asserted against the pure decision function with a stubbed `globalThis.fetch` (same pattern as T002, not a separate mock).

### Implementation for User Story 2

- [ ] T010 [US2] In `src/apps/build-studio/selfheal/PreviewStatus.tsx`, confirm the handler's non-`ready` short-circuit (the `if (a.ok === false || a.state !== "ready") return;` from T005) satisfies US2 with **no further code change** — `activate` alone starts the build and the top toolbar renders the in-progress state. Do not add a poll loop, retry, or second reload. (If a change is needed it is within `PreviewStatus.tsx` only; otherwise this task is a verification no-op.)

**Checkpoint**: Both user stories work; the handler is complete.

---

## Phase 5: Polish & Cross-Cutting Concerns

- [ ] T011 Run `npx tsc --noEmit` and `npm run lint`; fix every error in the changed files before finishing. (Constitution VII quality gates.)
- [ ] T012 [P] Run the full `npm run test:unit` suite and confirm no regressions in the existing self-heal / build-studio tests.
- [ ] T013 [P] If any `docs/dev` or `docs/usage` note describes this button's behaviour and is now inaccurate, correct it in the same change (Constitution VI). For a pure bug fix that meets the documented intent, this is likely a no-op — verify, don't assume.

**Checkpoint**: Feature complete — typecheck, lint, and the full unit suite are green.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately.
- **Foundational (Phase 2)**: T002 must exist and FAIL before Phase 3 implementation (TDD gate).
- **User Story 1 (Phase 3)**: Depends on T002 (failing test). T003–T006 build the handler; T007 preserves the footnote; T008 turns the tests green.
- **User Story 2 (Phase 4)**: Depends on Phase 3 (same handler). Mostly verification of the existing short-circuit.
- **Polish (Phase 5)**: Depends on all user stories; the typecheck/lint/test gates are the finish line.

### User Story Dependencies

- **US1 (P1)**: The core handler. Independently testable via T002 case (b).
- **US2 (P2)**: A property of the same handler (the non-`ready` branch). No independent code path; independently testable via T002 case (a).

### Within Each User Story

- Tests (T002) are written and FAIL before implementation.
- The handler (T005) is built around the `activate`→`pin` order before the UI state (T003/T006) and the guard (T007).
- US1 is complete before US2 is verified.

### Parallel Opportunities

- T009 (US2 test confirmation) is `[P]` relative to reading T005 — different concern, same file, so it is sequenced after Phase 3 in practice.
- T012 and T013 (Polish) are `[P]` with each other (independent checks), both after the handler is complete.

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: confirm structure (no new app surface).
2. Phase 2: write the failing test (T002).
3. Phase 3: implement the handler + button (T003–T007); tests pass (T008).
4. **STOP and VALIDATE**: the failing test now passes for the ready-preview case.

### Incremental Delivery

1. Setup + Foundational → failing contract in place.
2. Add US1 → ready-preview pin+reload works.
3. Add US2 → not-built/stopped build path verified.
4. Polish → typecheck, lint, full suite green.

---

## Notes

- [P] tasks = different files, no dependencies.
- TDD: T002 must be observed failing before T003–T006.
- The change is confined to `PreviewStatus.tsx` + `tests/self-heal/preview-pin.test.ts` (SC-004); the pure decision function extracted for testability lives inside `PreviewStatus.tsx` (a same-directory sibling is permitted but not required — see `design.md` §4/§7). No new Supervisor endpoint, client, or Promote/Discard control (FR-005, FR-006).
- Commit after each task or logical group on `bos/032-self-heal-pin`.
- Do not run `npm run build` while `next dev` is running (Constitution VII).
