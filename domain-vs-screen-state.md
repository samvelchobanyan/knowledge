---
type: principle
severity: required
status: stable
applies_to: any-flutter-project
keywords: [riverpod, state-management, notifier, domain-state, screen-state, controller, single-source-of-truth]
related: [[layered-architecture]], [[freezed-immutable-state]], [[selector-providers-for-rebuilds]]
---

# Domain state vs. screen state

## Rule

Distinguish two kinds of state and place each in its proper layer:

- **Domain state** — business data and operations: tracks, playlists, articles, current playback, notification subscriptions, auth status. Lives in `domain/state/` as a `Notifier<T>` (or `AutoDisposeNotifier<T>`) exposing a Freezed state class and business operations (`load`, `toggle`, `refresh`). Long-lived across screen navigation.
- **Screen state** — UI-only state: search query, filter selection, expanded/collapsed flags, scroll position, "which panel is showing" toggle in a complex widget. Lives next to the screen in `features/<feature>/presentation/<screen>/<screen>_controller.dart`. Short-lived; dies when the screen unmounts.

Screens watch domain state directly via `ref.watch(domainStateProvider)`. Screens **MUST NOT** proxy domain data through their controller (no `controller.articles` that internally reads `newsState.articles` — the screen reads `newsStateProvider` itself).

## Rationale

Two failure modes this rule prevents:

1. **Putting business data in a screen controller.** Then logging out from a Profile screen doesn't clear NewsScreen's already-loaded articles (different controller, different lifetime). The controller dies with the screen; business data shouldn't.
2. **Proxying domain data through the screen controller.** Then every domain change rebuilds the controller, which rebuilds the screen. Selector-based optimization becomes impossible (see [[selector-providers-for-rebuilds]]); the controller becomes a shadow copy of the domain state that has to be kept in sync.

The split also matches a deeper concern: domain state is *what the app knows*; screen state is *what the user is looking at right now*. They have different lifecycles and ownership.

## Implications

- A new business concept → new file in `domain/state/` with `<concept>_state.dart` (Freezed state class + `Notifier` + provider)
- A screen with no UI-only state → no controller file at all; `<screen>_screen.dart` reads `domainStateProvider` directly
- A screen with UI-only state → `<screen>_controller.dart` next to the screen, exposes only the UI state, never the domain data
- Domain Notifiers extend `Notifier<T>` (Riverpod 2 modern API). `AutoDisposeNotifier<T>` is used when state should die with the last listener (e.g., a dashboard that should re-fetch when re-entered). **`StateNotifier` is not the current canonical** — older spec examples may show it, but every Notifier on `dev` uses `Notifier` / `AutoDisposeNotifier`. New state classes should follow the current code, not the older spec snippets.
- UI-only toggles that look domain-like aren't. Example: `MiniPlayerWidget._isMusicMode` (which panel is expanded in the mini-player) is screen state and lives inside the widget's `State`; it is intentionally separate from `PlayerState.mode` (what the player is actually playing). The two can be in opposite values simultaneously without contradiction — the user can toggle which panel is shown without changing playback.

## When this principle does NOT apply

- Trivial widget-local state (a `TextEditingController`, an `AnimationController`, an `_isHovered` flag) belongs in widget `State`, not in a Riverpod controller. The rule targets state that other widgets might care about, not every `setState`.
- Forms with multi-step flows (registration, password reset) may legitimately hold transient cross-screen state in a domain-state Notifier even though the data isn't "business" in the traditional sense — once the flow completes, the Notifier resets. The criterion is shared lifetime across screens, not the moral category of the data.
