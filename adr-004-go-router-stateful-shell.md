---
type: decision
severity: required
status: stable
date: 2026-05-12
applies_to: any-flutter-project
keywords: [navigation, go-router, stateful-shell-route, bottom-navigation, tab-state-preservation, deep-link]
supersedes:
superseded_by:
related: [[semantic-vs-navigational-ownership]]
code_refs:
  - go_sport/pubspec.yaml
  - go_sport/lib/core/navigation/app_router.dart
  - go_sport/lib/core/navigation/main_shell.dart
  - go_sport/lib/core/navigation/routes.dart
---

# ADR-004: go_router with StatefulShellRoute.indexedStack

## Context

The app has three primary tabs (Home / Music / Radio) plus many auth and profile flows reached from any context. Each tab needs to preserve its navigation stack independently — switching from Music's "Album Detail" to Radio and back should land on Album Detail, not reset Music to its dashboard. Deep-links from notifications must resolve to specific screens, and the redirect logic for authenticated / guest / unauthorized users has to run before any matched route renders.

## Decision

Use `go_router: ^14.2.0` with `StatefulShellRoute.indexedStack` as the navigation backbone. Three `StatefulShellBranch`es (Home, Music, Radio) each maintain their own navigator stack. Global routes outside the shell handle auth flows (`/login`, `/registration-*`, `/confirm-*`, etc.) and cross-cutting screens (`/profile`, `/expired-guest`). A top-level `redirect:` callback enforces auth state.

Route configuration lives in `core/navigation/app_router.dart` (`createAppRouter(tokenStorage)` builds the `GoRouter`); route paths are named constants in `core/navigation/routes.dart` (`AppRoutes.music`, `AppRoutes.musicMyFavorites`, etc.). `MainShell` (`core/navigation/main_shell.dart`) wraps the active branch and holds the `BottomNavBar`, mini-player, and guest-timer bar.

Which branch owns which route follows the [[semantic-vs-navigational-ownership]] principle, not domain categorization.

## Alternatives considered

No alternatives were explicitly recorded at the time of decision.

## Consequences

- (+) Each tab preserves its stack across branch switches via `indexedStack`
- (+) Declarative route configuration in one place; route names are constants, refactor-safe
- (+) `redirect:` callback handles auth gating uniformly — no per-screen "am I logged in?" check
- (+) Deep-link resolution is the same code path as in-app navigation — the notification tap handler can `_router.push(AppRoutes.radioSchedule)` directly
- (+) Active community, consistent with current Flutter team recommendations
- (-) Branch navigator keys (`_homeBranchNavigatorKey`, etc.) need explicit declarations; their use cases (e.g., resolving the active branch's context for a modal) are subtle (see open investigation in `log.md` — modal-in-stateful-shell-route candidate)
- (-) Page transitions need custom builders (`fadeSlidePage`) for non-default animations — `go_router` doesn't ship a transition system, only `pageBuilder`
- (-) `state.extra` is dynamically typed (`state.extra as Album`) — runtime type errors are possible if the call site mismatches
- (-) Complex redirect chains (auth + onboarding + force-update banners) can produce surprising precedence; keep the redirect function readable

## Notes for future projects

Re-evaluate if:

- A simpler app with no tab structure ships — `Navigator` 1.0 or basic `MaterialApp.router` without `StatefulShellRoute` is enough
- A breaking change in `go_router` substantially alters the shell-route API (the API was unstable through 12.x → 14.x)
- The app gains web-routing requirements where URL semantics need stronger guarantees than `go_router` provides
