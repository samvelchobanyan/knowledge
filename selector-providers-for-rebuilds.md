---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [riverpod, selector, provider, rebuilds, performance, ref-watch-select, derived-provider, records]
related: [[domain-vs-screen-state]], [[freezed-immutable-state]]
code_refs:
  - go_sport/lib/domain/state/player_state_selectors.dart
---

# Selector providers for rebuilds

## Problem

A single Notifier holds a large state object — `PlayerState` has ~15 fields, including high-frequency ones (position ticks ~5x/sec, buffered position) and low-frequency ones (current track, play/pause, shuffle mode, radio metadata). A widget that reads the whole state via `ref.watch(playerStateProvider)` rebuilds on *every* update, including position ticks it doesn't care about. Conversely, splitting the state across many tiny Notifiers loses transactional consistency (`mode + tracks + currentIndex` must change together).

## Solution

Keep one Notifier as the single source of truth. Layer narrow `Provider<T>` selectors over it that expose only the slice each consumer cares about. Riverpod's `.select` compares the derived value by equality; if unchanged, downstream watchers don't rebuild.

```dart
// domain/state/player_state_selectors.dart

// High-frequency: progress only — only the progress bar should watch this
final playerProgressProvider = Provider<double>((ref) {
  return ref.watch(playerStateProvider.select((s) => s.progress));
});

// Low-frequency: track and mode info — title/artist UI watches this
final playerInfoProvider = Provider<({
  Track? track,
  bool isPlaying,
  bool isRadioMode,
  PlayerStatus status,
  String? displayImageUrl,
  String? radioTitle,
  // ...
})>((ref) {
  return ref.watch(playerStateProvider.select((s) => (
    track: s.currentTrack,
    isPlaying: s.isPlaying,
    isRadioMode: s.isRadioMode,
    status: s.status,
    displayImageUrl: s.displayImageUrl,
    radioTitle: s.radioTitle,
    // ...
  )));
});

// Composite for the seek bar
final playerSeekBarProvider = Provider<({
  Duration position,
  Duration bufferedPosition,
  Duration duration,
  bool canSeek,
})>((ref) {
  return ref.watch(playerStateProvider.select((s) => (
    position: s.position,
    bufferedPosition: s.bufferedPosition,
    duration: s.effectiveDuration,
    canSeek: s.effectiveDuration > Duration.zero &&
             s.status != PlayerStatus.loading,
  )));
});
```

Each UI element subscribes to exactly the slice that drives its visual change: the progress bar to `playerProgressProvider`, the title text to `playerInfoProvider`, the seek bar to `playerSeekBarProvider`. Position ticks rebuild only the progress bar and seek bar; a track change rebuilds title/artwork; mode-switch rebuilds the mode-specific UI.

## When to apply

- A large state object with mixed-frequency update cadences
- A state slice that drives a visible UI element distinct from others (one slice → one UI region)
- When profiling reveals that a widget rebuilds far more often than its visible content changes

## When NOT to apply

- Small states (~3 fields) where every consumer cares about most fields — selectors add indirection without saving rebuilds
- Slices that change at the same cadence (no rebuild reduction even with selectors)
- One-off ad-hoc reads inside a callback (`ref.read(...)`) — selectors are for reactive watching, not one-shot reads

## Trade-offs

- (+) Surgical rebuilds — high-frequency updates don't trigger UI that doesn't depend on them
- (+) Selectors document state slicing — the file lists the named projections used by UI
- (+) Dart 3 records make multi-field tuple selectors ergonomic without inventing new classes
- (-) Indirection — a debugger trace passes through three providers (state → selector → consumer) instead of one
- (-) Record-typed selectors are anonymous; renaming a field requires updating every consumer manually
- (-) Selector equality depends on Freezed-generated `==` for the underlying state — hand-rolled state classes silently break selectors

## Common pitfalls

- Forgetting that `.select` compares by `==`: returning a new `List` each time (`[...s.tracks]`) bypasses memoization. Return the underlying list reference or a primitive computed from it.
- Building a selector that watches half the fields, then a consumer that reads `playerStateProvider` directly anyway — the selector saved nothing.
- Over-fragmenting: 20 single-field selectors for one Notifier add more code-reading cost than they save. Group selectors by *consumer* (one selector per UI region), not by *field*.
