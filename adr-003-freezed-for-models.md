---
type: decision
severity: required
status: stable
date: 2026-05-12
applies_to: any-flutter-project
keywords: [freezed, immutable, json-serializable, code-generation, models, entities, dto]
supersedes:
superseded_by:
related: [[freezed-immutable-state]], [[dto-to-domain-mapping]]
code_refs:
  - go_sport/pubspec.yaml
  - go_sport/lib/domain/entities/track.dart
  - go_sport/lib/data/dto/track_dto.dart
---

# ADR-003: Freezed for all immutable models

## Context

The project has three categories of model classes that need to be immutable, value-equality-based, and `copyWith`-friendly:

- Domain entities (`Track`, `Album`, `Artist`, `Playlist`, `Program`, `Story`, `NewsArticle`, ...)
- State classes for Notifiers (`PlayerState`, `AuthState`, `NewsState`, `MusicDashboardState`, ...)
- DTOs (`TrackDto`, `AlbumDto`, ...) that additionally need JSON parsing

Hand-writing equality, `hashCode`, `copyWith`, and `toString` for every class is error-prone and bloats files; missing an `==` update after adding a field silently breaks Riverpod selectors.

## Decision

Use `freezed: ^2.5.7` with `freezed_annotation: ^2.4.4` for all immutable models (entities + state + DTOs). DTOs additionally use `json_serializable: ^6.8.0` and `json_annotation: ^4.9.0` for `fromJson`. Generation is driven by `build_runner: ^2.4.13` in dev dependencies.

## Alternatives considered

No alternatives were explicitly recorded at the time of decision.

## Consequences

- (+) `copyWith`, `==`, `hashCode`, `toString` all generated; schema changes propagate automatically on `build_runner` rerun
- (+) Sealed Freezed classes (`@freezed sealed class AuthState`) work with Dart 3 pattern matching for exhaustive state handling
- (+) Freezed + json_serializable is the de-facto Flutter standard — code from documentation and packages assumes this stack
- (+) Riverpod `.select` and selector providers rely on Freezed-generated `==` for correct memoization
- (-) Generated files (`*.freezed.dart`, `*.g.dart`) require `build_runner` in the dev loop; first build is slow
- (-) Stack traces include generated code paths
- (-) Const-construction has restrictions — Freezed factories must be `const factory` and all fields must be const-compatible; mixing with mutable references is an error
- (-) `build_runner` occasionally produces stale generated files that need `flutter clean` — annoying but rare

## Notes for future projects

Re-evaluate if:

- Dart adds first-class data classes (proposed but not landed as of writing); Freezed's value would diminish substantially
- A non-codegen alternative (macros once they stabilize) becomes viable for the same use case without the build_runner overhead
- The project is small enough that hand-written models are realistically maintainable (~5 entities total)
