---
type: decision
severity: required
status: stable
date: 2026-05-12
applies_to: any-flutter-project
keywords: [auth, tokens, secure-storage, flutter-secure-storage, keystore, keychain]
supersedes:
superseded_by:
related: [[local-state-via-storage-class]]
code_refs:
  - go_sport/lib/core/auth/token_storage.dart
  - go_sport/lib/core/network/interceptors/auth_interceptor.dart
---

# ADR-005: FlutterSecureStorage for auth tokens

## Context

The app stores authentication state — access token, refresh token, and a "user chose to continue as guest" flag — and uses it to authorize every backend API call. The chosen mechanism has direct security consequences:

- Tokens must survive app restart, so they live in persistent storage, not in memory only
- Tokens must not be readable by other apps on the device and must not appear in plain unencrypted device backups
- The HTTP layer (a Dio `AuthInterceptor`) must attach the access token on every request; a `Future<String?>` round-trip on every call is unacceptable, so the token must be readable synchronously after startup

## Decision

Use `flutter_secure_storage` as the persistence layer for auth tokens, fronted by a concrete `TokenStorage` class (`core/auth/token_storage.dart`) with an in-memory cache and synchronous getters. `TokenStorage.init()` loads the cached values from secure storage once at app startup before `runApp`. After init, the interceptor reads `tokenStorage.accessToken` synchronously and attaches the Bearer header.

The `TokenStorage` class is wired into Riverpod via a provider declared with `throw UnimplementedError` and overridden in `ProviderScope.overrides` with the real, initialized instance built in `main()`.

## Alternatives considered

No alternatives were explicitly recorded at the time of decision.

## Consequences

- (+) Tokens encrypted at rest via platform-backed Keychain (iOS) / Keystore (Android); not exposed in plain device backups
- (+) Synchronous getters after init — the `AuthInterceptor` attaches the Bearer token without `await` on every request
- (+) Tokens survive app restarts but are wiped on uninstall (no migration concern)
- (+) Provider override mechanism keeps the class testable without an `abstract` interface
- (-) Startup grows by one async secure read; mitigated by doing it once before `runApp`
- (-) `flutter_secure_storage` occasionally returns null on Android after Keystore reset (factory reset, security incidents on the device); calling code must tolerate null and fall back to "unauthenticated"
- (-) This decision is narrow to **auth tokens specifically**. It does not generalize to all local persistence; for non-secret local state (notification subscriptions, UI flags), `flutter_secure_storage` would be slow and ceremonious. The structural shape — concrete storage class with in-memory cache and async init — is captured separately as [[local-state-via-storage-class]] and applies regardless of underlying store. A future project might reuse this ADR for tokens while picking `shared_preferences` for non-secret state via the same structural pattern.

## Notes for future projects

Re-evaluate this decision if:

- A cross-platform secure store with substantially better Android Keystore reliability becomes the community default
- The auth model requires reactive watching of token changes (current code reads tokens synchronously; if the interceptor needs to react to a refresh in flight, the synchronous-cache design needs revisiting)
- Token storage moves out of the device entirely (server-side session, OAuth device flow with no persisted token), which would obsolete this decision rather than refine it
