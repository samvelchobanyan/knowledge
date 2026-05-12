---
type: decision
severity: required
status: stable
date: 2026-05-12
applies_to: flutter-projects-with-local-notifications
keywords: [notifications, flutter-local-notifications, timezone, scheduled-notifications, local-only]
supersedes:
superseded_by:
related: [[local-notifications-stack]], [[adr-007-inexact-alarm-as-default]], [[local-notifications-platform-gotchas]]
code_refs:
  - go_sport/pubspec.yaml
  - go_sport/lib/core/notifications/notification_service.dart
---

# ADR-006: flutter_local_notifications + timezone for scheduled local reminders

## Context

The app needs to deliver local notifications scheduled minutes-to-hours in advance — specifically, "remind me 10 minutes before this radio program starts" for user-selected program slots from a published radio schedule. Requirements:

- Local-only: subscription state lives only on the device (no backend involvement, no FCM)
- Survives app close (notification fires even if the app isn't running)
- Survives device reboot (re-registers scheduled notifications on `BOOT_COMPLETED`)
- Tap deep-links into the radio schedule screen
- Time-zone-correct ("at 10:50 local time" means user's actual local time, not UTC)

## Decision

Use `flutter_local_notifications: ^17.2.3` for scheduling/displaying local notifications and `timezone: ^0.9.4` (its required dependency from version 9.x) for timezone-aware scheduling. Subscriptions persist via `shared_preferences: ^2.3.2` through a `ReminderStorage` storage class (see [[local-state-via-storage-class]]). The full stack is described in [[local-notifications-stack]].

The notification scheduling strategy on Android (exact vs inexact alarms) is a separate decision — see [[adr-007-inexact-alarm-as-default]].

## Alternatives considered

- **FCM (server-pushed notifications)** — rejected. Would require a backend that knows the user's subscriptions and schedules outbound pushes, which violates the local-only requirement and adds backend work that isn't otherwise needed. FCM also doesn't guarantee delivery at a specific minute; it's eventually-consistent.
- **`awesome_notifications`** — not explicitly evaluated at decision time. `flutter_local_notifications` was chosen as the more conservative, longer-established option with broader documentation.
- **Custom platform-channel implementation** — rejected as disproportionate work for the use case.

## Consequences

- (+) Subscriptions and scheduled notifications stay on-device; no backend dependency
- (+) Notifications fire while the app is killed; Android receivers re-register on boot via `BOOT_COMPLETED`
- (+) `zonedSchedule` with `tz.TZDateTime.from(at, tz.local)` handles DST transitions correctly
- (+) Notification tap routes through one payload constant (`kRadioSchedulePayload`) to the schedule screen via `go_router`
- (-) Notifications are unreliable when the user reinstalls the app or migrates devices — local state is lost; subscribers see no reminders silently
- (-) Aggressive vendor battery optimization (Xiaomi, Huawei, Oppo) can kill scheduled notifications; there's nothing the app can do about this at the code level
- (-) `flutter_local_notifications` 17+ requires Android core library desugaring (`isCoreLibraryDesugaringEnabled = true` + `coreLibraryDesugaring` dependency in `android/app/build.gradle.kts`)
- (-) iOS foreground notifications require `UNUserNotificationCenter.current().delegate` registration in `AppDelegate.swift` to be visible while the app is in foreground — see [[local-notifications-platform-gotchas]]
- (-) Time-zone init must happen before any scheduling; forgetting `tz_data.initializeTimeZones()` leads to wrong-time fires

## Notes for future projects

Re-evaluate if:

- A unified notification UX requires both server-pushed and locally-scheduled notifications with shared deep-link handling — combining FCM + flutter_local_notifications is supported but adds coordination complexity
- The reminder requirement broadens to types of notifications that benefit from server delivery (cross-device sync, server-driven reminders)
- A Flutter macro or platform-channel improvement makes the Android desugaring / iOS delegate boilerplate go away
