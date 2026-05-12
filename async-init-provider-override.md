---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [riverpod, provider, override, async-init, provider-scope, dependency-injection, main]
related: [[local-state-via-storage-class]], [[api-client-with-config-and-interceptors]]
code_refs:
  - go_sport/lib/main.dart
  - go_sport/lib/core/auth/token_storage.dart
  - go_sport/lib/core/notifications/notification_service.dart
---

# Async-init provider override

## Problem

Some infrastructure dependencies need asynchronous initialization before they're usable: `TokenStorage` reads tokens from `flutter_secure_storage`, `ReminderStorage` reads subscriptions from `SharedPreferences`, `NotificationService` calls `initializeTimeZones()` and `plugin.initialize()`, `AppAudioHandler` needs `AudioService.init(...)`. Riverpod's regular `Provider((ref) => Foo())` factory is synchronous — it can't `await` an init step. Wrapping in `FutureProvider` propagates async-ness up to every consumer (every UI widget has to deal with `AsyncValue<T>` even though, post-startup, the value is always ready).

## Solution

Declare the provider as `Provider<T>` that throws on read, then override it in `ProviderScope.overrides` with the already-initialized instance built in `main()`:

```dart
// In the class file:
final tokenStorageProvider = Provider<TokenStorage>(
  (_) => throw UnimplementedError(
    'tokenStorageProvider must be overridden in ProviderScope',
  ),
);

class TokenStorage {
  Future<void> init() async { /* read from secure storage */ }
  String? get accessToken => _cachedAccessToken;
  // ...
}

// In main.dart:
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  const env = String.fromEnvironment('ENV', defaultValue: 'dev');
  final config = env == 'prod' ? AppConfig.prod : AppConfig.dev;

  final tokenStorage = TokenStorage();
  await tokenStorage.init();

  final apiClient = ApiClient(config, [AuthInterceptor(tokenStorage)]);

  final notificationService = NotificationService();
  await notificationService.init();
  await notificationService.requestPermissions();

  final reminderStorage = ReminderStorage();
  await reminderStorage.init();

  try {
    final audioHandler = await AudioService.init(
      builder: () => AppAudioHandler(),
      config: const AudioServiceConfig( /* ... */ ),
    );

    runApp(
      ProviderScope(
        overrides: [
          audioHandlerProvider.overrideWithValue(audioHandler),
          apiClientProvider.overrideWithValue(apiClient),
          tokenStorageProvider.overrideWithValue(tokenStorage),
          notificationServiceProvider.overrideWithValue(notificationService),
          reminderStorageProvider.overrideWithValue(reminderStorage),
        ],
        child: MainApp(tokenStorage: tokenStorage),
      ),
    );
  } catch (e, stackTrace) {
    debugPrint('AudioService Init Error: $e');
    runApp(MaterialApp(/* fallback UI showing the error */));
  }
}
```

Consumers downstream do `ref.watch(tokenStorageProvider)` and get an already-initialized object synchronously. The "must be overridden" error fires only if someone forgets the override or constructs a `ProviderScope` without it (e.g., in a test that didn't set it up) — a loud, easy-to-diagnose failure.

The `try/catch` around `AudioService.init` shows a defensive variation: if a critical async init fails, run a fallback `MaterialApp` that surfaces the error rather than crashing on a white screen. Use this pattern for inits that can plausibly fail at runtime (audio service binding, plugin initialization) and not for inits whose failure would mean the app is unusable anyway.

## When to apply

- Any infrastructure that needs `await` to be ready (storage, audio, notifications, network plumbing)
- Any singleton consumed by many providers but constructed once at startup
- When test setup needs to swap the production implementation (override with a fake in the test's `ProviderScope`)

## When NOT to apply

- Pure factory providers that can construct synchronously — `Provider((ref) => Foo())` is fine, no override needed
- Per-screen ephemeral state — that's `AutoDisposeNotifier`, not infrastructure
- Things constructed lazily and rarely enough that `FutureProvider` + `AsyncValue` is acceptable for consumers (e.g., a remote config loaded once per app lifetime that's checked at a single splash screen)

## Trade-offs

- (+) Consumers see a fully-initialized object synchronously — no `AsyncValue` ceremony
- (+) Provider declaration documents what the type is, while keeping construction explicit in `main()` where order and dependencies are visible
- (+) Test setup is straightforward: build a fake instance, override the same provider in the test's `ProviderScope`
- (+) The `throw UnimplementedError` produces a clear failure mode if a `ProviderScope` is missing the override
- (-) Startup `main()` grows with each dependency — but linearly, and the ordering is explicit
- (-) Forgetting an override silently lets the app build until the first consumer reads — the error is informative but happens at runtime, not compile time
- (-) Each dependency adds another `await` to `main()` before `runApp`; first paint waits for the longest chain

## Common pitfalls

- Constructing the instance inside the `Provider((_) => ...)` factory anyway, defeating the override — the throw is there for a reason; honour it
- Forgetting `await` on `init()` — the override fires with an uninitialized object; subtle later failures (empty getters, nulls where values were expected)
- Putting business logic in the `try/catch` around `AudioService.init` instead of using it for a fallback UI — the catch is for *can't even start* failures, not for app-level error handling
- Ordering inits arbitrarily — interceptors depending on storage need the storage init to complete first; the dependency chain has to be respected manually (the language won't enforce it)
