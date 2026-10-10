# Implementation Plan Checklist: Files — Multi-Selection & Drag-and-Drop

**Purpose**: Validate plan completeness and quality before proceeding to tasks
**Created**: 2026-09-26
**Feature**: [plan.md](../plan.md)

## Plan Completeness

- [x] Technical context is filled in (language, deps, storage, testing, platform, performance, constraints)
- [x] Constitution Check has passed (all principles verified, or violations have justification in Complexity Tracking)
- [x] All [NEEDS CLARIFICATION] items from the spec are resolved
- [x] All Load-Beating design questions are answered
- [x] Project structure is defined
- [x] Deliverables section defines concrete outputs
- [x] Quality gates (typecheck, lint, docs sync) are defined
- [x] Plan references design.md rather than repeating it

## Should-Fix Integration (design review §7)

- [x] SF-1 (preventDefault on VFS drag so drop fires) baked in as PD-1 with a required test
- [x] SF-2 (drag-start window raise vs cross-window drop) baked in as PD-2 with a non-overlapping-window e2e decision
- [x] SF-3 (descendant/self-drop no-op before op-decision) baked in as PD-3 with a required test
- [x] Nits (N-2 ghost DOM, N-3 fresh reason, N-4 single refresh trigger) baked in as PD-4

## Consistency with Spec & Design

- [x] App Target (builtin-app, src/apps/files/) consistent across spec, design, plan
- [x] All FR-001…FR-016 map to a design decision (D1–D6) and/or a deliverable
- [x] All SC-001…SC-006 appear in the quality gates / deliverables
- [x] FR-013 no-regression (upload path, existing data-testids, single-item behaviors) is explicitly protected (D4, R3)

## Notes

- Items marked incomplete require plan updates before `__SPECKIT_COMMAND_TASKS__`.
