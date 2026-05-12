---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [storage, local-state, riverpod, shared-preferences, secure-storage, token-storage, in-memory-cache, no-repository]
related: [[adr-005-secure-storage-for-tokens]], [[dto-to-domain-mapping]], [[async-init-provider-override]]
code_refs:
  - go_sport/lib/core/auth/token_storage.dart
  - go_sport/lib/core/notifications/reminder_storage.dart
---

# Local state via storage class

## Problem

An app needs to persist some local state that isn't a domain entity fetched from an API: auth tokens, feature flags, notification subscriptions, user-preference cache, "user dismissed the banner" flags, a small set of IDs the user has favourited locally. Wrapping these in the repository pattern is awkward — there's no remote/local split to abstract, no mapping layer needed, and the `*Impl` / `abstract class` ceremony adds friction without saving anything. But sprinkling `SharedPreferences.getInstance()` (or `FlutterSecureStorage()`) calls across the codebase is worse: initialization races, untyped string keys, hidden coupling, and no single source of truth for the in-memory state.

## Solution

A concrete storage class lives in `core/<feature>/<name>_storage.dart`. The class:

- Has a **domain-aware API** named after what it stores (`accessToken`, `subscribedIds`, `add(slotId)`), not a generic key-value surface (no `get(String key)`)
- **Owns one underlying store** internally (`FlutterSecureStorage`, `SharedPreferences`, a JSON file) — consumers don't see it
- Has an **`init()` method** that loads persisted state into an in-memory cache, called once in `main()` before `runApp`
- Provides **synchronous getters** after init; writes are async and update both the cache and the persistent store
- Is **a single concrete class with no `abstract` base** and no `*Impl` suffix

The class is wired into Riverpod via the [[async-init-provider-override]] pattern: declare `Provider<T>((_) => throw UnimplementedError(...))`, construct the instance in `main()` after `await init()`, inject it through `ProviderScope.overrides`.

Reference shapes from go_sport. The class itself never imports Riverpod; orchestrating it with other providers (e.g. scheduling a notification on each subscription change) belongs in a `Notifier` that consumes the storage — see `ProgramRemindersNotifier extends Notifier<Set<String>>` for this shape: it reads `_storage.subscribedIds`, calls `_storage.add(...)` on toggle, and pairs each write with `_notifications.scheduleAt(...)`.

```dart
// core/auth/token_storage.dart — secrets (FlutterSecureStorage)
class TokenStorage {
  String? _cachedAccessToken;
  final _secureStorage = const FlutterSecureStorage();

  String? get accessToken => _cachedAccessToken;

  Future<void> init() async {
    _cachedAccessToken = await _secureStorage.read(key: _accessTokenKey);
    // ...
  }

  Future<void> saveTokens({ /* ... */ }) async { /* update cache + write */ }
  Future<void> clearTokens() async { /* update cache + delete */ }
}

// core/notifications/reminder_storage.dart — non-secret (SharedPreferences)
class ReminderStorage {
  final Set<String> _ids = <String>{};
  SharedPreferences? _prefs;

  Set<String> get subscribedIds => UnmodifiableSetView(_ids);

  Future<void> init() async {
    _prefs = await SharedPreferences.getInstance();
    final stored = _prefs!.getStringList(_storageKey);
    if (stored != null) _ids..clear()..addAll(stored);
  }

  Future<void> add(String slotId) async {
    if (_ids.add(slotId)) await _persist();
  }
  // ...
}
```

## When to apply

- State that isn't fetched from an API (no remote source to abstract)
- State accessed from multiple places that should share one source of truth
- State whose persistence layer (secure vs non-secret) follows from the data itself, not from swap-ability
- Initialization can happen once at startup before the UI mounts

## When NOT to apply

- Data fetched from a backend API — use the repository pattern with DTOs (see [[dto-to-domain-mapping]])
- State that lives only as long as one screen — use a feature controller / local widget state
- Ephemeral one-shot data with no readers outside the writing site — inline the storage call
- Cases where the underlying store must legitimately swap at runtime (offline-first scenarios, encryption toggles) — that does need an abstraction; this pattern is the wrong starting point

## Trade-offs

- (+) Synchronous getters after init — consumers don't await for already-loaded state
- (+) One file per concern; the underlying store (`secure_storage` → `shared_preferences` → file) can change without touching consumers
- (+) Domain-aware API rules out typo'd keys and untyped values
- (+) Naturally testable via Riverpod provider override; no `abstract class` needed
- (-) Requires async init in `main()` — startup sequence grows; an `init()` left unawaited returns empty getters and the bug is hard to spot
- (-) Asymmetry with the repository pattern — developers familiar with `*Repository` / `*Impl` may instinctively add an `abstract class` "for testability"; resist (the override mechanism already provides it)

## Common pitfalls

- Forgetting to `await storage.init()` before `runApp` — getters return empty/null and downstream logic silently misbehaves
- Adding an `abstract class` to "match the repository style" — undoes the simplicity that motivated the pattern
- Letting the storage class import `Ref` or Riverpod types — keep it a pure Dart class; orchestration with other providers belongs in a Notifier that consumes the storage
- Exposing mutable internal collections directly (e.g. returning the raw `Set` instead of `UnmodifiableSetView`) — invites accidental mutation from outside
