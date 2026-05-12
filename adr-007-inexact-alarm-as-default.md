---
type: decision
severity: recommended
status: stable
date: 2026-05-12
applies_to: flutter-projects-with-local-notifications
keywords: [android, alarms, schedule-exact-alarm, use-exact-alarm, inexact, play-store-policy, doze]
supersedes:
superseded_by:
related: [[adr-006-flutter-local-notifications-plus-timezone]], [[local-notifications-stack]], [[local-notifications-platform-gotchas]]
code_refs:
  - go_sport/lib/core/notifications/notification_service.dart
  - go_sport/android/app/src/main/AndroidManifest.xml
---

# ADR-007: inexactAllowWhileIdle as default Android scheduling strategy

## Context

Android scheduling for local notifications has three meaningful options as of Android 14:

- **`USE_EXACT_ALARM`** — exact firing time, auto-granted to the app, no user interaction. *But* Google Play policy restricts this to apps where exact timing is the *core* functionality (alarm clocks, calendar reminders). Apps using it for adjacent features risk Play Store enforcement action.
- **`SCHEDULE_EXACT_ALARM`** — exact firing time. On Android ≤13, auto-granted at install. On Android 14+, the app must lead the user to system settings (`ACTION_REQUEST_SCHEDULE_EXACT_ALARM`) to toggle it on manually.
- **`AndroidScheduleMode.inexactAllowWhileIdle`** (via `flutter_local_notifications`) — no exact-alarm permission needed; firing time may be ±5–15 minutes depending on Doze mode and system load.

The use case in `go_sport` is "remind me 10 minutes before a radio program starts" — convenience, not safety-critical timing. Off by a few minutes is fine.

## Decision

Use `AndroidScheduleMode.inexactAllowWhileIdle` as the default scheduling mode in `NotificationService.scheduleAt`. Do **not** declare `USE_EXACT_ALARM` or `SCHEDULE_EXACT_ALARM` in `AndroidManifest.xml`. The manifest declares `POST_NOTIFICATIONS` and `RECEIVE_BOOT_COMPLETED` only.

## Alternatives considered

- **`USE_EXACT_ALARM`** — rejected. Play Store policy explicitly restricts it to apps where exact timing is core functionality. A radio-reminder feature does not qualify; using it risks app review issues at publish time.
- **`SCHEDULE_EXACT_ALARM` with `inexactAllowWhileIdle` as fallback** — rejected as overengineered. On Android 14+ the app would have to send the user into system settings to toggle the permission, which is poor UX for the value it adds (±5 minutes precision improvement for non-critical notifications).
- **`AndroidScheduleMode.exact`** without the permission — rejected. Throws at scheduling time on devices where the permission isn't granted.

## Consequences

- (+) No Play Store policy risk — `inexactAllowWhileIdle` doesn't require either exact-alarm permission
- (+) No "send the user to system settings" UX detour required on Android 14+
- (+) Simpler manifest and simpler code path — no per-version branching
- (+) Works reliably in Doze mode; the OS chooses an economical wake-up window
- (-) Firing time may drift by ±5 minutes typically, up to ±15 minutes in heavy Doze. For "10 minutes before the program" that's acceptable; for "exactly when the alarm goes off" it would not be.
- (-) If the use case ever shifts to require exact timing, this ADR needs revisiting and the Play Store policy analysis re-done

## Notes for future projects

Re-evaluate if:

- The app gains an alarm-clock-style feature where exact timing is genuinely the value the user is paying for — then `USE_EXACT_ALARM` becomes defensible under Play Store policy
- A user-reported pattern emerges of missed notifications because Doze pushed delivery too late — measure first; vendor battery-killing is a more common cause and not solvable by changing scheduling mode
- Android tightens the rules around `inexactAllowWhileIdle` in a future version (the API has been quietly evolving)
