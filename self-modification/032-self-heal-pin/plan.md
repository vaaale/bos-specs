# Implementation Plan: Self-Heal "Pin & open preview" performs the Supervisor work

**Branch**: `bos/032-self-heal-pin` | **Date**: 2026-09-29 | **Spec**: `/Specs/user-specs/self-modification/032-self-heal-pin/spec.md`

**Input**: Feature specification from `032-self-heal-pin/spec.md` (App Target: `builtin-app`).

## Summary

Fix the **Pin & open preview** button on the self-heal case card (`PreviewStatus.tsx`, class-`e` cases) so that clicking it does the Supervisor work the label promises instead of opening the Settings app. Concretely: issue an idempotent `activate` for the case's `activeFeatureBranch` through the shared `supervisorPost` client, and — only once `activate` reports the preview `ready` — `pin` the preview for that branch and reload the window so the top toolbar flips to `PREVIEW`. A behaviour fix to one existing button in one existing built-in-app file; no new surface, no new Supervisor endpoint, no new client.

**Technical approach.** The single `onClick` navigation (`launch("settings", { tab: "self-modification", branch })`) is replaced by an async handler that (1) sets a local in-progress state, (2) `await supervisorPost("activate", { branch })`, (3) proceeds to `await supervisorPost("pin", { version: "preview", branch })` only when the activate result is `ok !== false` **and** `state === "ready"`, (4) reloads the window only when the `pin` result is `ok`. This mirrors the two-call sequence the existing top-toolbar (`VersionControls.tsx`) and Versions-tab (`VersionsTab.tsx`) controls already perform, reusing the same `supervisorPost` surface (FR-006) and its `replacesServedBuild` reload rule (reload on `pin`). The FR-024 footnote beneath the button is preserved verbatim (FR-005).

## Technical Context

- **Language/Version**: TypeScript (the repo's TS config), React (client components, `"use client"`).
- **Primary Dependencies**: `react` (`useState`), `lucide-react` (icon), `@/lib/supervisor/client` (`supervisorPost`, `PostResult`), existing `@/store/os-provider` import is **removed** (no longer navigates).
- **Storage**: N/A.
- **Testing**: Playwright unit runner (`npm run test:unit`). New test file at `tests/self-heal/preview-pin.test.ts` (the repo convention is `tests/<area>/<name>.test.ts`, not co-located `src/**/__tests__/`). The test verifies the handler's decision logic against the Supervisor response contract.
- **Target Platform**: The BrowserOS desktop web app (Next.js App Router).
- **Project Type**: modification to an existing built-in app (`src/apps/build-studio/selfheal/`).
- **Performance Goals**: N/A — two existing HTTP round-trips to the Supervisor control surface; no new work added.
- **Constraints**:
  - Must NOT open the Settings app (FR-001).
  - Must NOT pin or reload unless `activate` reported `ready` (FR-002 / FR-003).
  - Must show + clear a transient in-progress state and disable the button while in flight (FR-004).
  - Must NOT add a second Promote/Discard path; preserve the FR-024 footnote (FR-005).
  - Must reuse `supervisorPost`; no new endpoint or client (FR-006).
  - Confined to `PreviewStatus.tsx` (handler + imports) and one test file (SC-004). No `package.json`/lockfile/build-config change (Constitution VII).
- **Scale/Scope**: one component handler + one unit test.

## Constitution Check

*GATE: Must pass before implementation.*

- **I. Spec-Driven** — Pass. `spec.md` exists and is the source of truth; this plan and the tasks derive from it.
- **II. Server Authority & SSR Boundary** — Pass (n/a). The handler calls the existing `/__supervisor/*` control surface via `supervisorPost` (a `fetch` to server-owned routes). No secrets or Node APIs move to the client.
- **III. Always Delegate; Claude Codes** — Pass. Implementation is delegated to the Developer (Claude) sub-agent on the feature branch; this plan/tasks are authored here, code is not.
- **IV. Minimize Blast Radius** — Pass. The change is on the `bos/032-self-heal-pin` feature branch; preview/promote via the Supervisor. A single component is touched.
- **V. The VFS Is Not the Source** — Pass. Spec artifacts live in the user-specs store; code changes are made by the Developer on `src/`, never via VFS file tools.
- **VI. Specs & Docs Stay in Sync** — Pass. This is a bug fix to documented 031-self-healing behaviour; the change is recorded here and in `spec.md`. No new `docs/usage` page is warranted (no user-facing doc change — the button's documented intent is now *met*, not altered). If the Developer finds a `docs/dev` note about this button needs correcting, it should be updated in the same change.
- **VII. Respect Boundaries** — Pass. No `package.json`/lockfile/build-config edits. Quality gates `npx tsc --noEmit` and `npm run lint` must pass for the changed code; no `npm run build` while `next dev` is running.

**Gate result: PASS. No violations → no Complexity Tracking entry required.**

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/self-modification/032-self-heal-pin/
├── spec.md    # approved (Gate 1)
└── plan.md    # this file
    (+ tasks.md created in the next step — NOT here)
```

### Source Code (repository root)

This is a modification to an existing built-in app. There is no new app folder, no `manifest.ts`/`index.tsx` to create, and no registry to edit (the app is already discovered by `tools/gen-apps.mjs`). The scope is exactly these two files:

```text
src/apps/build-studio/selfheal/
└── PreviewStatus.tsx        # MODIFIED — the "Pin & open preview" handler (activate → pin → reload) + imports

tests/self-heal/
└── preview-pin.test.ts      # NEW — unit test for the handler's activate/pin decision logic
```

**Structure Decision**: Single-component bug fix within the existing `build-studio` app. The fix lives entirely in `PreviewStatus.tsx` (its `onClick` handler and imports); the test lives in `tests/self-heal/` alongside the other self-heal unit tests. Nothing is created or restructured.

## Design Notes

- **Handler shape.** A local `useState` flag drives the transient state. The handler is `async`, guarded by an early return when `!branch` (defensive — the button only renders when `ready && branch`). Sequence: set in-progress → `activate` → if `a.ok === false || a.state !== "ready"` return (no pin, no reload) → `pin` → if `p.ok` then `window.location.reload()` → finally clear in-progress. This is the same two-call order `VersionControls`/`VersionsTab` use, so the behaviour is consistent with the top toolbar the user is being handed to next.
- **Why `activate` before `pin`.** For a `preview-ready` case the preview is normally already `ready` (`completeFix` verified it), so `activate` is a no-op. It is retained because a preview restored after a Supervisor restart can be `not-built`/`stopped`; `pin` refuses a building preview, so the handler must only pin once `activate` confirms `ready`.
- **Button label.** Renders `"Opening preview…"` while in progress, else `"Pin & open preview"`; `disabled` while in progress (with `disabled:opacity-50`), matching the existing button's visual language.
- **Reload scope.** Only a successful `pin` (and `promote`, which this button never does) replace the served build per `replacesServedBuild`, so the reload is gated on `p.ok` — consistent with the top toolbar.
- **Removed dependency.** `useOSStore` (the `launch` for the Settings navigation) is removed from imports since the button no longer navigates. `ScopeClassBadge`, `Loader2`, and the `HealingCase` type remain.
- **Testing strategy.** The unit test asserts the decision matrix from the spec's edge cases: (a) `activate` not `ready` ⇒ no pin, no reload; (b) `activate` `ready` + `pin` ok ⇒ reload called once; (c) `activate`/`pin` `ok:false` ⇒ no reload, state restored; (d) missing branch ⇒ no Supervisor call. Mock `supervisorPost` (and `window.location.reload`) rather than hitting a live Supervisor.
