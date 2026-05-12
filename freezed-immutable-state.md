---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [freezed, immutable, sealed-class, copy-with, data-class, value-object, dart-3-patterns]
related: [[domain-vs-screen-state]], [[adr-003-freezed-for-models]]
code_refs:
  - go_sport/lib/domain/entities/track.dart
  - go_sport/lib/domain/state/player_state.dart
  - go_sport/lib/core/auth/auth_state.dart
---

# Freezed immutable state

## Problem

The app needs immutable data classes for two distinct purposes: **domain entities** (`Track`, `Album`, `Artist` — value objects passed around the app) and **state classes** (`PlayerState`, `AuthState`, `NewsState` — what a Notifier holds). Hand-writing equality, `hashCode`, `copyWith`, and `toString` for every class is error-prone (forgetting to update `==` after adding a field is a classic bug) and bloats the file. Add JSON parsing for DTOs and the boilerplate doubles.

## Solution

Use `freezed` (with `freezed_annotation` and `build_runner`) for all immutable model classes. Two shapes are common:

**Product type** — a class with named fields, generated `copyWith`/`==`/`hashCode`/`toString`:

```dart
@freezed
class PlayerState with _$PlayerState {
  const factory PlayerState({
    @Default(PlaybackMode.music) PlaybackMode mode,
    @Default([]) List<Track> tracks,
    @Default(0) int currentIndex,
    @Default(Duration.zero) Duration position,
    // ...
  }) = _PlayerState;
}
```

**Sealed sum type** — for closed sets of alternatives (states with mutually exclusive shapes, union types):

```dart
@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.unauthorized() = AuthUnauthorized;
  const factory AuthState.guest() = AuthGuest;
  const factory AuthState.authenticated({
    required String name,
    required String avatarUrl,
    required int userId,
  }) = AuthAuthenticated;
}
```

Sealed Freezed classes work with Dart 3 pattern matching (`switch (state) { AuthGuest() => ..., AuthAuthenticated(:final name) => ..., ... }`), giving exhaustive checks at compile time.

DTOs additionally use `json_serializable` (also generated via `build_runner`) for `fromJson`. Domain entities never have `fromJson` (see [[dto-to-domain-mapping]]).

## When to apply

- Domain entities — every entity in `domain/entities/` is `@freezed`
- State classes for Notifiers in `domain/state/` and feature controllers
- Mutually exclusive variants (auth states, sealed result types, queue source variants) — use the sealed form

## When NOT to apply

- Tiny short-lived value pairs returned from a function — Dart 3 records (`(int, String)`) are simpler
- UI helpers that hold references (`AnimationController`, `PageController`) — those aren't values, aren't immutable
- Classes used only in pure-Dart utility code where pulling `build_runner` into the build pipeline is heavier than the savings

## Trade-offs

- (+) `copyWith` always correct as schema evolves — add a field, regenerate, all callers still compile
- (+) Free `==` and `hashCode` — Riverpod selectors and `ref.watch(...).select(...)` rely on these; hand-rolled equality breaks selectors silently
- (+) Sealed classes + Dart 3 patterns → compile-time exhaustiveness for states with distinct shapes
- (-) Requires `build_runner` in the development loop; first build is slow, watcher mode helps but is another moving piece
- (-) Generated files (`*.freezed.dart`, `*.g.dart`) clutter directory listings even when gitignored from commits
- (-) Stack traces include generated code paths, which read less cleanly than hand-written code

## Common pitfalls

- Forgetting to run `build_runner` after changing a Freezed class — the analyzer shows misleading errors until the part file regenerates
- Mixing mutable and immutable fields (`@Default([]) List<Track>` is fine because lists are wrapped, but `final controller = TextEditingController()` inside a Freezed class breaks the immutability contract)
- Using non-`const` factories (`factory ... =`) when `const factory ... =` is possible — loses const-construction at call sites
- Treating a Freezed list field as mutable (`state.tracks.add(...)` does not change state) — use `state.copyWith(tracks: [...state.tracks, newTrack])` instead
