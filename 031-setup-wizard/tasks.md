# Tasks: Configuration Wizard (First-run Setup)

## Phase A — Wizard shell & navigation

- [ ] A1. Create `src/components/desktop/setup-wizard/wizard-types.ts` — `WizardState`, `SetupOperation` types.
- [ ] A2. Create `WizardShell.tsx` — full-screen modal overlay (`z-[200000]`), step indicator (1–5 dots, Step 6 hidden from indicator), Back/Next/Finish buttons with correct disabled states.
- [ ] A3. Create `SetupWizard.tsx` — root component; holds `WizardState`; renders `WizardShell` with the current step child; replaces `FirstRunWizard.tsx` import in `Desktop.tsx`.
- [ ] A4. Delete `FirstRunWizard.tsx`.

## Phase B — Step 1: AI Provider

- [ ] B1. Create `Step1AiProvider.tsx` — extract provider/model/baseUrl/apiKey fields from the deleted `FirstRunWizard.tsx`; wire to `WizardState`; disable Next when `provider` is empty.

## Phase C — Step 2: Dev Harness

- [ ] C1. Create `Step2DevHarness.tsx` — embed `<DevHarnessTab>` with a brief intro paragraph explaining it can be configured later in Settings.

## Phase D — Step 3: Data Isolation

- [ ] D1. Create `Step3DataIsolation.tsx` — embed `<DataFsTab>` with a brief intro paragraph.

## Phase E — Step 4: Git Repositories

- [ ] E1. Create `Step4GitRepos.tsx` — three labeled rows (BOS Source / BOS Central Specs / User Apps), each with URL + branch inputs; pre-fill defaults per FR-007; soft warning if URL filled but branch empty.

## Phase F — Step 5: Marketplace

- [ ] F1. Create `Step5Marketplace.tsx` — checked table with the three default rows (BOS Central ✓, Claude Superskills ☐, Anthropic Skills ☐); Add button for custom rows; wires rows into `WizardState`.

## Phase G — Step 6: Setting Up

- [ ] G1. Create `Step6SettingUp.tsx` — non-interactive progress list; fires all operations on mount; shows spinner → ✓/✗ per operation; calls `POST /api/system/setup` then invokes `onComplete` callback when all settled.
- [ ] G2. Implement Step 6 operations in a `runSetup(state: WizardState)` helper:
  - Write Step 1 values: `PATCH /api/config { namespace: "ai-provider", ... }`.
  - BOS Source remote: if URL given, `POST /api/git-remotes` to add `origin` remote to `bos-src`.
  - BOS Central Specs: if URL given, call a new `/api/system/setup/clone-specs` endpoint (or extend existing setup route) to clone and skip seeding; if empty, ensure git-init via existing seed path.
  - User Apps: `POST /api/system/setup/init-user-apps` (or equivalent) — clone if URL given, else git-init.
  - Marketplace adds: sequential `POST /api/marketplace { op: "add", url }` for each checked row.

## Phase H — Seed guard

- [ ] H1. In `src/lib/specs/seed.ts`, add a guard: if the target system spec-store path already contains a `.git` directory, skip seeding for that store (it was cloned by the wizard).

## Phase I — Verification

- [ ] I1. `npx tsc --noEmit` — clean.
- [ ] I2. `npm run lint` — clean.
- [ ] I3. Manual test: fresh `data/` directory → wizard appears, complete all steps → desktop opens, Settings → Versions shows expected GitFS entries.
- [ ] I4. Manual test: restart after setup → wizard does NOT appear.
- [ ] I5. Manual test: leave all optional steps blank → no errors, desktop opens, `data/user-apps/` and spec store exist as git repos.
