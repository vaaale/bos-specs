# Feature Specification: Configuration Wizard (First-run Setup)

**Feature Branch**: `031-setup-wizard`

**Created**: 2026-07-28

**Status**: New (supersedes `000-browseros-core` FR-014)

**Input**: "Replace the single AI Provider configuration dialog shown on first start with a 6-step wizard: (1) AI Provider, (2) Dev Harness, (3) Data Isolation, (4) Git Repositories, (5) Marketplace Sources, (6) Setting Up progress screen. All configured repositories must be cloned/pulled on completion; unconfigured repos must still have their GitFS directories created so they appear in Settings → Versions."

> This spec owns the **first-run configuration wizard**: the modal shell, step navigation, what each step contains, and the completion sequence (Step 6 repo setup + marketplace cloning). Each step's *detailed field behaviour* is owned by the spec that governs that settings surface (`006-data-isolation`, `029-settings-dev-harness`, `030-settings-mcp-servers`); this spec owns only the wizard framing and what happens when the user presses Finish. It fully supersedes `000-browseros-core` FR-014, which is now a pointer.

## Clarifications

### Session 2026-07-28

- Q: Can the user close the wizard without completing it? → A: **No.** The wizard is a full-screen modal (same z-index as the current dialog) that blocks the desktop until Finish is reached. There is no dismiss/close button.
- Q: Can the user navigate back to earlier steps? → A: **Yes.** Back/Next buttons allow free forward/backward navigation through all steps before the user reaches Step 6.
- Q: Are any fields required? → A: **Step 1 (AI Provider) requires a provider to be selected** — BOS cannot function without some AI configuration. All other steps (2–5) are fully optional: the user may leave every field blank and click Next; all can be configured later in Settings.
- Q: What if a repo clone fails in Step 6? → A: **Log the error, mark that operation failed in the UI, continue with the rest.** Once all operations are attempted (even if some failed), the wizard marks `setupComplete = true` and opens the desktop. Failed items are shown clearly so the user knows to retry in Settings → Versions or Settings → Marketplace.
- Q: Does configuring a BOS Central Specs URL replace the default spec-store seeding entirely? → A: **Yes, exclusively.** If a URL is provided, Step 6 clones that repo into the system spec-store location instead of running the seed routine. If left empty, the existing seed routine runs as before *and* an empty git repo is initialised at the same location so the store appears in Settings → Versions. A URL-backed install will contain whatever the remote has; seed content is not merged in.
- Q: The existing wizard had a simplified Dev Harness section (single transport dropdown). Should it be retained? → A: **No — replaced by the full DevHarnessTab.** Step 2 embeds `DevHarnessTab` exactly as it appears in Settings, giving the same experience in both places.
- Q: What branch names apply to the Step 4 repo defaults? → A: BOS Source defaults to branch `main`; BOS Central Specs defaults to `master`; User Apps has no branch default (left blank unless the user enters one).
- Q: Are marketplace branch names exposed in Step 5? → A: **No.** The marketplace API (`POST /api/marketplace { op: "add", url }`) handles cloning and uses the remote's default branch. Step 5 collects URLs only.
- Q: What is the exact completion sequence at the end of Step 6? → A: (1) Repo clones / git-inits and marketplace `add` operations are fired — repo operations in parallel (they are independent), marketplace adds sequentially (to avoid filesystem races). (2) Each operation's result (success / failed / skipped-empty) is shown live. (3) Once all are settled, `POST /api/system/setup` marks `setupComplete = true`. (4) The wizard modal closes and the desktop renders.
- Q: Which marketplace entries are pre-checked in Step 5? → A: **BOS Central marketplace only** (`https://github.com/vaaale/bos-marketplace.git`) is checked by default. Claude Superskills (`https://github.com/ericgandrade/claude-superskills.git`) and Anthropic Skills (`https://github.com/anthropics/skills.git`) are listed as suggestions and are **unchecked** by default.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Complete first-run setup without confusion (Priority: P1)

A user starting BOS for the first time sees a clear, stepwise wizard instead of a dense single-form dialog. They can step forward and backward freely, skip optional steps, and end up at the desktop with a working AI provider configured.

**Why this priority**: The wizard is the very first interaction with BOS. A confusing or blocking first-run experience kills adoption immediately.

**Independent Test**: Start BOS with a clean `data/` directory; confirm the wizard modal appears before the desktop, navigate all 6 steps, press Finish, confirm the desktop renders and the AI provider is usable.

**Acceptance Scenarios**:

1. **Given** a fresh BOS install (no `data/config/` files), **When** BOS starts, **Then** the wizard modal covers the desktop and the desktop is not accessible until Finish.
2. **Given** the wizard is open on any step, **When** the user clicks Back, **Then** the previous step renders with its previously entered values intact.
3. **Given** the user skips Steps 2–5 entirely (clicks Next without entering anything), **When** they reach Step 6, **Then** setup completes without error (empty operations are skipped, not failed).
4. **Given** setup has been completed once (`setupComplete = true`), **When** BOS restarts, **Then** the wizard does NOT appear.

### User Story 2 — Configure AI provider (Priority: P1)

The user selects their AI provider and model on Step 1, exactly as in the current wizard dialog.

**Why this priority**: Without AI configuration BOS cannot function at all.

**Independent Test**: Select Anthropic, enter an API key, advance to Step 2; confirm `data/config/ai-provider.json` is written with the correct values after Finish.

**Acceptance Scenarios**:

1. **Given** Step 1, **When** it renders, **Then** the provider dropdown, model field, base URL, and API key fields match those in the current `FirstRunWizard.tsx` AI Provider section.
2. **Given** no provider selected, **When** the user clicks Next on Step 1, **Then** an inline validation error appears and navigation is blocked.
3. **Given** a provider selected and Finish reached, **When** Step 6 completes, **Then** `data/config/ai-provider.json` holds the entered values.

### User Story 3 — Configure Dev Harness (Priority: P2)

Step 2 gives the user the same Dev Harness configuration experience as Settings → Dev Harness, without requiring them to find it post-setup.

**Why this priority**: Harness configuration is the most common reason users return to Settings immediately after setup; surfacing it in the wizard saves that trip.

**Independent Test**: On Step 2, select Claude Code, enter an API key, save; confirm the generated harness config file reflects it identically to how Settings → Dev Harness would save it.

**Acceptance Scenarios**:

1. **Given** Step 2, **When** it renders, **Then** it shows the full `DevHarnessTab` component (top-level harness selector, two CLI panels) — identical to Settings → Dev Harness.
2. **Given** settings saved in Step 2, **When** the user later opens Settings → Dev Harness, **Then** the same values are shown (single config namespace `dev-harness`).

### User Story 4 — Configure data isolation (Priority: P2)

Step 3 lets the user choose the data-isolation method that suits their host filesystem.

**Why this priority**: Choosing the wrong isolation method silently degrades performance or correctness; surfacing it once at setup is better than discovering the issue later.

**Independent Test**: On Step 3, change the method from the default; after Finish confirm `data/config/datafs.json` holds the selected method.

**Acceptance Scenarios**:

1. **Given** Step 3, **When** it renders, **Then** it shows the full `DataFsTab` component — identical to Settings → Data Isolation.
2. **Given** the user leaves Step 3 at the default, **When** Finish completes, **Then** `data/config/datafs.json` holds the probed-best-compatible method.

### User Story 5 — Configure source repositories (Priority: P2)

Step 4 lets the user connect BOS to its upstream git remotes and optionally a user-apps repo, pre-filled with sensible defaults.

**Why this priority**: Without git remotes the user cannot pull BOS updates or store their own apps in version control.

**Independent Test**: Enter a URL for BOS Source only, leave others blank; after Step 6 confirm a git remote named `origin` exists on the `bos-src` filesystem, `data/user-apps/` exists as an empty git repo, and the system spec store exists as a git repo (seeded from local seed, not cloned).

**Acceptance Scenarios**:

1. **Given** Step 4, **When** it renders, **Then** three rows appear: BOS Source (pre-filled URL + `main`), BOS Central Specs (pre-filled URL + `master`), User Apps (empty URL/branch). All rows have a clear URL field and a branch field; all are optional.
2. **Given** a URL entered for BOS Source, **When** Step 6 runs, **Then** a remote named `origin` with that URL and branch is added to the `bos-src` GitFS instance.
3. **Given** a URL entered for BOS Central Specs, **When** Step 6 runs, **Then** the system spec store is cloned from that URL (seed routine is skipped); the spec store appears in Settings → Versions.
4. **Given** BOS Central Specs left blank, **When** Step 6 runs, **Then** the existing seed routine runs and a git repo is initialised at the spec-store location so it appears in Settings → Versions.
5. **Given** User Apps left blank, **When** Step 6 runs, **Then** `data/user-apps/` is created as an empty git repo and appears in Settings → Versions.
6. **Given** User Apps URL entered, **When** Step 6 runs, **Then** the repo is cloned into `data/user-apps/`.

### User Story 6 — Configure marketplace sources (Priority: P2)

Step 5 lets the user choose which marketplace repos to add, with BOS Central pre-selected and two community repos offered as opt-in suggestions.

**Why this priority**: Marketplaces are how users discover and install apps and skills; pre-populating the official one reduces friction, and surfacing community options at setup increases discoverability.

**Independent Test**: Accept BOS Central, check Claude Superskills, leave Anthropic Skills unchecked; after Step 6 confirm BOS Central and Claude Superskills appear in `data/config/marketplaces.json` and their repos are cloned into `data/marketplace/`.

**Acceptance Scenarios**:

1. **Given** Step 5, **When** it renders, **Then** a table shows three rows: BOS Central (checked), Claude Superskills (unchecked), Anthropic Skills (unchecked); all rows show name, URL, and a checkbox.
2. **Given** the user can add a custom row, **When** they enter a URL, **Then** a new row appears in the table with a checkbox (checked by default).
3. **Given** checked marketplaces, **When** Step 6 runs the `add` operation for each, **Then** each is cloned via `POST /api/marketplace { op: "add", url }` and appears in the Marketplace app.
4. **Given** an unchecked marketplace row, **When** Step 6 runs, **Then** it is not added (no API call, no clone, no entry in `marketplaces.json`).

### User Story 7 — Observe setup progress (Priority: P1)

Step 6 shows a live progress list of every setup operation (repo clones, git-inits, marketplace adds), lets the user see what succeeded or failed, and auto-advances to the desktop once all operations have settled.

**Why this priority**: Without feedback, the user doesn't know whether clones are happening or whether an error silently blocked their setup.

**Independent Test**: Enter one valid and one invalid repo URL; after Finish observe the progress list shows one ✓ and one ✗ with an error message, and the desktop opens regardless.

**Acceptance Scenarios**:

1. **Given** Step 6 starts, **When** operations begin, **Then** each operation is listed by name with a spinner → ✓ (success) or ✗ (failed) as it settles.
2. **Given** a repo clone fails (e.g. invalid URL or no network), **When** Step 6 finishes, **Then** the desktop opens and the failed item is clearly marked with an error description; no other operation is blocked.
3. **Given** all operations complete (success or failure), **When** the list settles, **Then** `POST /api/system/setup` is called and the desktop opens automatically (no manual "Close" button required).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: On first startup — defined as `setupComplete !== true` in the `system` config namespace — BOS MUST display the wizard modal before the desktop is accessible. The modal MUST NOT be dismissible; the desktop MUST NOT render until `setupComplete = true`.
- **FR-002**: The wizard MUST provide Back and Next navigation buttons. Back is disabled on Step 1; Next on Step 5 is labelled "Finish & Set Up". The user may navigate freely between Steps 1–5; Step 6 is entered only by pressing "Finish & Set Up" and cannot be navigated away from once started.
- **FR-003**: Step 1 (AI Provider) MUST require a provider selection before Next is enabled. All other steps MUST allow Next with no fields filled.
- **FR-004**: Step 1 MUST embed the AI provider fields from the current `FirstRunWizard.tsx` (provider, model, base URL, API key) and persist them to the `ai-provider` config namespace on Finish.
- **FR-005**: Step 2 (Dev Harness) MUST embed `DevHarnessTab` as-is from `src/components/apps/settings/DevHarnessTab.tsx`. Its Save action MUST work normally within the wizard; the wizard's own Next button does not trigger a save. The existing simplified transport dropdown from `FirstRunWizard.tsx` is retired.
- **FR-006**: Step 3 (Data Isolation) MUST embed `DataFsTab` as-is from `src/components/apps/settings/DataFsTab.tsx` (or its equivalent). Its Save action MUST work normally within the wizard.
- **FR-007**: Step 4 (Git Repositories) MUST present three rows: **BOS Source** (default URL `https://github.com/vaaale/browseros.git`, branch `main`), **BOS Central Specs** (default URL `https://github.com/vaaale/bos-specs.git`, branch `master`), **User Apps** (empty URL, empty branch). All rows MUST have a URL field and a branch field; all are optional.
- **FR-008**: Step 5 (Marketplace) MUST present a table pre-populated with: **BOS Central marketplace** (`https://github.com/vaaale/bos-marketplace.git`) checked; **Claude Superskills** (`https://github.com/ericgandrade/claude-superskills.git`) unchecked; **Anthropic Skills** (`https://github.com/anthropics/skills.git`) unchecked. The user MUST be able to add custom rows (URL + name) and check/uncheck any row.
- **FR-009**: Step 6 (Setting Up) MUST be non-interactive. It MUST display a live list of operations: one entry per Step 4 item (clone or git-init) and one per checked Step 5 marketplace (add). Each entry shows a spinner while in progress, then ✓ or ✗.
- **FR-010**: In Step 6, if a Step 4 URL is non-empty, BOS MUST clone/pull that repo into its target location. If the URL is empty, BOS MUST ensure the target directory exists as a git repository (git-init if not already). This MUST apply to all three Step 4 rows — no GitFS directory is silently skipped.
- **FR-011**: In Step 6, if BOS Central Specs has a URL, BOS MUST clone that repo into the system spec-store location and MUST NOT run the seed routine for that store. If the URL is empty, BOS MUST run the seed routine as before (idempotent) and MUST ensure the store directory is a git repository.
- **FR-012**: In Step 6, for each checked marketplace row, BOS MUST call `POST /api/marketplace { op: "add", url }`. Unchecked rows MUST NOT be added.
- **FR-013**: In Step 6, a failure in any single operation (clone error, marketplace add error) MUST NOT prevent the remaining operations from running or the wizard from completing. After all operations are settled, BOS MUST call `POST /api/system/setup` and close the wizard regardless of individual failures.
- **FR-014**: Once `setupComplete = true`, the wizard MUST NOT appear on subsequent starts. The existing `GET /api/system/setup` → `{ firstRun: boolean }` detection mechanism MUST be reused unchanged.
- **FR-015**: The existing `FirstRunWizard.tsx` MUST be deleted and replaced by the new wizard component. No dead code or compatibility shim is needed.

### Key Entities

- **Wizard state** — in-component state: current step index (1–6), per-step form values, Step 6 operation results. Not persisted (the wizard only runs once); each settings namespace is persisted independently by the embedded tab components or on Finish.
- **Step 4 repo config** — transient form values: `{ bosSource: { url, branch }, bosSpecs: { url, branch }, userApps: { url, branch } }`. Consumed only by the Step 6 setup routine; not written to a separate config file.
- **Step 5 marketplace rows** — transient table state: `Array<{ name, url, checked }>`. Consumed only by the Step 6 setup routine; persisted implicitly via the marketplace `add` API calls.
- **Setup operation result** — `{ label: string; status: "pending" | "running" | "ok" | "failed"; error?: string }` — one per Step 6 operation, rendered live.

## Success Criteria *(mandatory)*

- **SC-001**: A fresh BOS start (empty `data/`) shows the wizard before the desktop; the desktop renders only after Finish.
- **SC-002**: Navigating Back from Step 3 to Step 1 and re-advancing preserves all entered values.
- **SC-003**: Completing the wizard with all optional steps blank results in a usable desktop with no JS errors.
- **SC-004**: With a valid BOS Source URL entered, Settings → Versions shows a `bos-src` filesystem with an `origin` remote after setup.
- **SC-005**: With BOS Central Specs URL entered, the system spec store is a clone of that remote (not a seed copy); with it blank, the seed content is present and the directory is a git repo.
- **SC-006**: `data/user-apps/` is a git repository after setup regardless of whether a User Apps URL was entered.
- **SC-007**: Checked marketplaces appear in the Marketplace app after setup; unchecked ones do not.
- **SC-008**: A clone failure in Step 6 leaves the failed item marked ✗ but does not prevent the desktop from opening.
- **SC-009**: `npx tsc --noEmit` and `npm run lint` pass for all changed files.

## Assumptions & Dependencies

- Supersedes `000-browseros-core` FR-014 (pointer retained there for cross-reference stability).
- Step 2 depends on `029-settings-dev-harness` — `DevHarnessTab` is embedded directly.
- Step 3 depends on `006-data-isolation` — `DataFsTab` is embedded directly.
- Step 4 (git-init / clone logic) depends on `007-gitfs` — reuses the same GitFS remote registration path as Settings → Versions.
- Step 5 depends on `028-marketplace-sandbox` — reuses `POST /api/marketplace { op: "add" }`.
- Step 6 completion calls `POST /api/system/setup` — the same endpoint the current wizard calls; no change to that API.
- BOS Central Specs seeding (`src/lib/specs/seed.ts`) must be made conditional on whether the system spec-store location is already a git clone — this is a small guard in the seed routine, not a new spec.
