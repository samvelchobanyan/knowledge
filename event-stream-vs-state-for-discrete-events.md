---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [stream, stream-provider, state-provider, riverpod, events, notifications, deep-link, dedup]
related: [[local-notifications-stack]], [[async-init-provider-override]]
code_refs:
  - go_sport/lib/core/notifications/notification_service.dart
---

# Event Stream vs. State for discrete events

## Problem

Some app inputs are discrete events, not state: a notification tap (user pressed the alert), a deep-link arrival, a one-shot system signal. The instinct (especially in a Riverpod-heavy codebase) is to model them as `StateProvider<String?>` where the event payload is the state value and `null` means "no event pending". This produces two real bugs:

1. **The state must be manually reset to `null` after handling** — forget once, and downstream listeners re-fire stale events on every rebuild that re-runs the listener.
2. **Same-value events are dropped.** Riverpod doesn't notify listeners when `state = sameValue`, so a second notification with identical payload silently fails to trigger handling.

State models things that *are*; events are things that *happened*. Conflating them ships the bugs above as built-in features.

## Solution

Model discrete events as `Stream<T>` and expose them through `StreamProvider<T>`. Riverpod's `ref.listen(streamProvider, ...)` fires on every emission, dedup-free, with no reset ceremony.

```dart
// core/notifications/notification_service.dart
class NotificationService {
  final StreamController<String> _tapController =
      StreamController<String>.broadcast();

  Stream<String> get onTap => _tapController.stream;

  Future<void> init() async {
    // ...
    await _plugin.initialize(
      settings,
      onDidReceiveNotificationResponse: (response) {
        final payload = response.payload;
        if (payload != null && payload.isNotEmpty) {
          _tapController.add(payload);
        }
      },
    );
  }
}

final notificationServiceProvider = Provider<NotificationService>(
  (_) => throw UnimplementedError('Override in ProviderScope'),
);

final notificationTapProvider = StreamProvider<String>((ref) {
  return ref.watch(notificationServiceProvider).onTap;
});
```

Consumer side — `ref.listen` fires per emission, no manual reset:

```dart
@override
Widget build(BuildContext context) {
  ref.listen<AsyncValue<String>>(notificationTapProvider, (prev, next) {
    next.whenData(_handleNotificationPayload);
  });
  return MaterialApp.router(/* ... */);
}

void _handleNotificationPayload(String payload) {
  if (payload == kRadioSchedulePayload) {
    _router.push(AppRoutes.radioSchedule);
  }
}
```

**Cold-start launch** (app opened from a notification while killed) is the one case where a stream is awkward — the event happened before any listener existed. Handle it as a separate one-shot read of an `initialPayload` field stored on the service:

```dart
class NotificationService {
  String? _initialPayload;
  String? get initialPayload => _initialPayload;

  Future<void> init() async {
    // ...
    final launchDetails = await _plugin.getNotificationAppLaunchDetails();
    if (launchDetails?.didNotificationLaunchApp ?? false) {
      _initialPayload = launchDetails?.notificationResponse?.payload;
    }
  }
}

// In MainApp.initState, after the router is built:
WidgetsBinding.instance.addPostFrameCallback((_) {
  final payload = ref.read(notificationServiceProvider).initialPayload;
  if (payload != null) _handleNotificationPayload(payload);
});
```

One field for the cold-start case + a stream for runtime taps. Both feed into the same handler.

## When to apply

- Notification taps, deep links, push messages, system broadcasts
- Anything where "happened twice with the same payload" is a meaningful event, not a no-op
- Fire-and-forget signals consumed by zero-or-more reactive listeners

## When NOT to apply

- Long-lived state with discrete *transitions* (`AuthState`, `PlayerStatus`) — those *are* state, not events
- Counter-style accumulating values — that's state too
- Single-consumer one-off async actions — `Future<T>` and `await` are simpler

## Trade-offs

- (+) No manual reset, no stale-event bugs from forgotten `state = null`
- (+) Duplicate events fire correctly; the listener handles them per emission
- (+) `StreamProvider` integrates cleanly with `ref.listen` and `AsyncValue` patterns
- (+) Service stays a pure Dart class — Riverpod is layered on top
- (-) Cold-start (event before listener exists) needs a parallel `initialPayload` field — the stream-only model breaks down at app launch
- (-) Stream subscriptions need a `broadcast` controller if multiple consumers might listen (single-subscription streams throw on second listen)
- (-) `ref.listen` doesn't fire for the current value when first attached — that's the right behavior for events but can surprise developers expecting state-like semantics

## Common pitfalls

- Using a non-broadcast `StreamController` when more than one widget might subscribe — second listener throws
- Forgetting to close the `StreamController` on shutdown — usually harmless because the service lives for the app lifetime, but in tests with multiple service instances it leaks
- Mixing the cold-start `initialPayload` and the stream — call them at different sites: `initialPayload` once in `initState` (post-frame), the stream listener in `build` via `ref.listen`. Don't try to unify them through the stream; the timing doesn't work.
- Modelling everything as a stream because streams are pretty — long-lived state forced into stream shape loses the simple "what is the current value" question
