---
type: decision
severity: required
status: stable
date: 2026-05-12
applies_to: any-flutter-project
keywords: [riverpod, state-management, dependency-injection, provider, notifier]
supersedes:
superseded_by:
related: [[domain-vs-screen-state]], [[async-init-provider-override]], [[selector-providers-for-rebuilds]], [[freezed-immutable-state]]
code_refs:
  - go_sport/pubspec.yaml
  - go_sport/lib/main.dart
  - go_sport/lib/domain/state/
---

# ADR-001: Riverpod for state management and DI

## Context

The project needs a state management library for domain state (long-lived business data) and screen state (UI-specific state), plus a dependency-injection mechanism for infrastructure (API client, audio handler, storage classes, notification service). The choice is load-bearing: it shapes how every screen reads data, how features test, and how infrastructure gets wired up.

## Decision

Use **Riverpod 2.x** (`flutter_riverpod: ^2.5.1`) as the single state-management and DI mechanism. Domain Notifiers (`Notifier<T>`, `AutoDisposeNotifier<T>`) live in `domain/state/`; screen controllers (when needed) live next to screens. Infrastructure is exposed through `Provider<T>` with `ProviderScope` overrides for async-initialized singletons (see [[async-init-provider-override]]).

## Alternatives considered

No alternatives were explicitly recorded at the time of decision.

## Consequences

- (+) Single mechanism for both state and DI — no second framework for "providing" services
- (+) Compile-time provider references; renaming a provider catches every consumer
- (+) `.select` and `Provider<T>` selectors enable per-slice rebuilds (see [[selector-providers-for-rebuilds]])
- (+) `ProviderScope.overrides` makes test setup straightforward — swap an infra instance, no DI container ceremony
- (+) Strong Dart 3 + Freezed integration (sealed states, pattern matching on `state`)
- (-) Learning curve: `Provider` vs `Notifier` vs `AutoDisposeNotifier` vs `FutureProvider` vs `StreamProvider` distinctions take time, and team members coming from `BLoC` or `Provider` need ramp-up
- (-) `ref.watch` / `ref.read` / `ref.listen` semantics differ subtly — wrong choice produces silent rebuild bugs
- (-) Generated provider syntax (Riverpod 2.x annotations) exists alongside manual provider syntax; the codebase currently uses manual `final fooProvider = ...` — consistency matters, but the option to migrate later is open

## Notes for future projects

Re-evaluate if:

- Riverpod 3.x (with substantial breaking changes or new features like offline persistence) ships and offers a clear migration path
- The team grows substantially with members from a different state-management background — the cost-of-switching can shift
- A project's state model is so trivial that `setState` + `InheritedWidget` would suffice (smaller apps may not need Riverpod at all)
