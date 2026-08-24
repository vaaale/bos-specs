# Test Results: 034-event-notification-system

**Feature Spec**: [spec.md](./spec.md)
**Run at**: 2026-08-24T07:10:00Z
**Environment**: Chromium (Playwright), Playwright v1.60.0
**Test file**: `e2e/034-event-notification-system.spec.ts` (serial mode)

## Summary

| Metric    | Value |
|-----------|-------|
| Total     | 6     |
| Passed    | 6     |
| Failed    | 0     |
| Skipped   | 0     |
| Flaky     | 0     |
| Duration  | ~90s  |

## Results by Story

- **US1 — View & Triage Notifications**
  - ✅ Emits, lists, and marks events read (20.5s)
  - ✅ Historical toggle shows only read events (17.2s)
  - ✅ Mark all as read clears it (9.1s)

- **US3 — Automatic Headless Event Processing**
  - ✅ Processes an event via a registered headless handler (3.5s)
  - ✅ A failing handler retries, then permanently fails without blocking the event (13.1s)

- **US5 — Click an Event to Launch Its UI Handler**
  - ✅ Clicking an event marks it read and shows the generic (default) view (10.3s)

## Notes

- All tests self-clean: `unreadTotal` confirmed 0 after run; per-run unique type namespaces prevent cross-run collision.
- Serial mode (`test.describe.configure({ mode: "serial" })`) prevents parallel-worker race on the global "mark all as read" action (precedent: `e2e/039-service-tool-exposure.spec.ts`).
- The `buildstudio_run_tests` initial 0/6 failure was due to a stale base-server (port 3000) serving broken static chunks post-promotion — infra issue, not code. Verified 6/6 against a clean `npm run dev` on port 3100.
- Unit tests: 41/41 passing (`npm run test:unit -- tests/events/`).
