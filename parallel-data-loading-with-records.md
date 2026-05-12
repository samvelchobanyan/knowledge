---
type: pattern
severity: flexible
status: stable
applies_to: any-flutter-project
keywords: [dart-3, records, parallel, futures, wait, dashboard, repositories, performance]
related: [[domain-vs-screen-state]]
code_refs:
  - go_sport/lib/features/music/presentation/music/music_dashboard_controller.dart
---

# Parallel data loading with Records

## Problem

A dashboard or summary screen needs data from several independent sources before it can render — counts from one repository, featured items from another, user-specific data from a third. Calling them sequentially with `await` chains adds up: six 200ms calls in series is 1.2 seconds before the screen renders, versus 200ms if they fan out in parallel.

## Solution

Use Dart 3 record `.wait` to fan out independent `Future`s in parallel and destructure the results:

```dart
Future<void> load() async {
  state = state.copyWith(isLoading: true, error: null);

  try {
    final (
      favorites,
      playlists,
      albumsCount,
      artistsCount,
      episodes,
      programs,
      albums,
      artists,
    ) = await (
      _repository.getFavoritesCount(),
      _repository.getPlaylistsCount(),
      _repository.getAlbumsCount(),
      _repository.getArtistsCount(),
      _repository.getEpisodesCount(),
      _repository.getProgramsCount(),
      _albumRepository.getFeaturedAlbums(),
      _artistsRepository.getFeaturedArtists(),
    ).wait;

    state = state.copyWith(
      isLoading: false,
      favoritesCount: favorites,
      playlistsCount: playlists,
      // ... etc
    );
  } catch (e) {
    state = state.copyWith(isLoading: false, error: e.toString());
  }
}
```

The record `(future1, future2, ...).wait` waits for all futures concurrently and returns a record of their results in the same positional order. Pattern destructuring on the left binds them to local variables.

## When to apply

- Multiple independent reads needed before a screen can render
- The reads have no data dependencies (the second doesn't need the first's result)
- All reads are reasonably reliable; an error in any one cancels the whole batch (single try/catch covers all)

## When NOT to apply

- Reads with data dependencies (second needs the first's result) — keep them sequential
- Reads where partial success is meaningful (e.g., a dashboard that shows whatever it could load) — use `Future.wait` with `eagerError: false` or wrap each in try/catch individually; record `.wait` throws on any error
- Two or fewer reads — the syntactic weight of record `.wait` outweighs the benefit; a simple double-`await` is fine
- Reads with vastly different latency profiles where one slow read would block fast ones unnecessarily — consider per-section loading with separate state slices instead

## Trade-offs

- (+) Single network-round-trip-equivalent time for many reads → measurably faster initial render
- (+) Pattern destructuring keeps the result mapping readable
- (+) One try/catch covers all reads — simpler error path
- (-) All-or-nothing: any one read's failure throws, losing partial results
- (-) Positional record style ties result names to declaration order; reordering futures without reordering destructured variables silently corrupts mapping (no compile error if types align)
- (-) Records don't let you name the awaited tuple before destructuring, so the local-variable pattern can grow unwieldy past ~8 items

## Common pitfalls

- Reordering the futures without reordering the destructured variables → silently swapped data (no compile error, since types may match)
- Treating record `.wait` like `Future.wait(futures, eagerError: false)` — record `.wait` always throws on first error; for partial success use the older API or wrap each in try/catch
- Awaiting the record itself before `.wait` (`await (future1, future2)`) does nothing useful — the record wraps unawaited futures, not awaited values
