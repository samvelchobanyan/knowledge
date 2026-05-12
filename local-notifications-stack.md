---
type: pattern
severity: recommended
status: stable
applies_to: flutter-projects-with-local-notifications
keywords: [notifications, flutter-local-notifications, timezone, shared-preferences, reminders, scheduling, deep-link, payload]
related: [[adr-006-flutter-local-notifications-plus-timezone]], [[adr-007-inexact-alarm-as-default]], [[local-state-via-storage-class]], [[async-init-provider-override]], [[event-stream-vs-state-for-discrete-events]], [[shared-widgets-stay-dumb]], [[local-notifications-platform-gotchas]]
code_refs:
  - go_sport/lib/core/notifications/notification_service.dart
  - go_sport/lib/core/notifications/reminder_storage.dart
  - go_sport/lib/domain/state/program_reminders_state.dart
  - go_sport/lib/features/shared_widgets/schedule_tile.dart
  - go_sport/lib/features/shared_widgets/schedule_list.dart
---

# Local notifications stack

## Problem

A feature needs to schedule local notifications based on user subscriptions (e.g., "remind me 10 minutes before this radio program starts") and route the user to a relevant screen when they tap the notification. The naive approach — `flutter_local_notifications` calls scattered through feature code — produces uninitialized-plugin errors, untyped payloads, lost subscriptions across app restarts, and tightly-coupled UI/persistence/scheduling logic. The notification plugin is also init-heavy (timezone setup, permission requests, platform channel handlers) and shouldn't be touched outside one place.

## Solution

A four-layer stack, each layer with one job:

1. **`NotificationService`** (`core/notifications/notification_service.dart`) — the plugin wrapper. Owns `FlutterLocalNotificationsPlugin`, calls `tz_data.initializeTimeZones()`, registers `onDidReceiveNotificationResponse`, exposes `init()` / `requestPermissions()` / `scheduleAt(...)` / `cancel(...)` / `cancelAll()`. Pushes notification-tap payloads into a `Stream<String> onTap` (see [[event-stream-vs-state-for-discrete-events]]) and captures cold-start launch payload in `String? initialPayload`. Owns the Android notification channel definition.

2. **`ReminderStorage`** (`core/notifications/reminder_storage.dart`) — the subscription persistence. A storage class in the [[local-state-via-storage-class]] shape: `Set<String>` in memory, persisted via `SharedPreferences`, `init()` + sync `subscribedIds` getter + async `add()` / `remove()`. No knowledge of notifications — it just stores IDs.

3. **`ProgramRemindersNotifier`** (`domain/state/program_reminders_state.dart`) — the orchestrator. `extends Notifier<Set<String>>`. Reads `_storage.subscribedIds` on build, exposes one method `toggle(ScheduledProgram program)` that:
   - if subscribed → `_storage.remove(id)` + `_notifications.cancel(id.hashCode)`
   - if not → `_storage.add(id)` + `_notifications.scheduleAt(...)` *if* the reminder time hasn't already passed

   Updates its own state immutably so UI re-renders.

4. **UI** — `ScheduleTile` (dumb, in `features/shared_widgets/`) takes `isSubscribed` and `onSubscribeToggle` as props. `ScheduleList` (the parent `ConsumerWidget`) watches `programRemindersProvider`, computes per-tile subscription state, and routes `toggle(program)` callbacks (see [[shared-widgets-stay-dumb]]).

```
NotificationService                ReminderStorage
  • plugin wrapper                   • persists Set<String>
  • Stream<String> onTap             • core/notifications/reminder_storage.dart
  • initialPayload (cold-start)
        │                                  │
        └──────────────┬───────────────────┘
                       ▼
              ProgramRemindersNotifier
              extends Notifier<Set<String>>
                  toggle(program) →
                    add → schedule
                    remove → cancel
                       │
                       ▼ ref.watch / ref.read(...).notifier
              ScheduleList (ConsumerWidget)
                       │ isSubscribed + onSubscribeToggle
                       ▼
              ScheduleTile (StatelessWidget, dumb)
                       │ DSNotificationIcon
                       ▼
                     user tap
```

Both `NotificationService` and `ReminderStorage` are wired in via [[async-init-provider-override]] (constructed in `main()` after `await init()`, injected through `ProviderScope.overrides`).

**Notification payload as a constant string.** A single use case keeps payload as a constant (`const String kRadioSchedulePayload = 'radio_schedule'`). The deep-link handler in the root widget switches on this constant. When a second notification type appears, refactor to a sealed payload class — but only then.

**Notification channel registration.** Android requires a channel for each notification category. Hardcode it inside `NotificationService._buildDetails()` rather than scattering channel IDs across features:

```dart
NotificationDetails _buildDetails() => const NotificationDetails(
  android: AndroidNotificationDetails(
    'radio_reminders', 'Radio programs',
    channelDescription: 'Reminders for scheduled radio programs',
    importance: Importance.high,
    priority: Priority.high,
  ),
  iOS: DarwinNotificationDetails(),
);
```

## When to apply

- Any feature requiring scheduled local notifications (reminders, alarms, time-based prompts)
- When a notification tap should deep-link into the app
- When subscriptions must persist across app restarts

## When NOT to apply

- Server-pushed notifications only (FCM / APNS): different stack, no local scheduling, no `flutter_local_notifications` dependency
- One-off transient in-app banners or toasts: use a banner widget or `ScaffoldMessenger`, not the notification system
- Features that need only foreground in-app reminders without OS-level alerts: a `Timer` in a Notifier is simpler

## Trade-offs

- (+) Each layer has one concern — service doesn't know about state, storage doesn't know about notifications, UI doesn't know about either
- (+) Adding a second reminder type means adding a new Notifier on top of the same `NotificationService` + a new `Storage`; the plugin layer doesn't change
- (+) Storage and service swap independently in tests via Riverpod overrides
- (+) Schedule-on-toggle, cancel-on-untoggle keeps OS state in sync with user intent without a reconciliation pass
- (-) Four files for what feels like one feature — overhead is real for small reminder sets, justified by separation of concerns
- (-) Time-zone handling has to be initialized before scheduling — easy to forget in tests or alternate entry points
- (-) Platform gotchas pile up at the boundary; see [[local-notifications-platform-gotchas]]

## Common pitfalls

- Skipping the `if (reminderAt.isAfter(DateTime.now()))` check in `_subscribe` — `zonedSchedule` with a past time silently does nothing (or, depending on plugin version, fires immediately and confuses the user)
- Cancelling by raw `slotId` instead of `slotId.hashCode` — `flutter_local_notifications` IDs are `int`; keep the hashing consistent everywhere (use a single `_notificationId(slotId)` helper)
- Forgetting that the notification channel must exist before any notification scheduled with it appears — it gets created on first use, but production users on misconfigured devices have reported missed first notifications
- Forgetting that scheduled notifications survive app uninstall+reinstall on some Android variants (rare but real) — `cancelAll()` on first launch of a fresh install is a defensive option
