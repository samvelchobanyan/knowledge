---
type: principle
severity: required
status: stable
applies_to: any-flutter-project
keywords: [widgets, shared-widgets, composition, separation-of-concerns, state-ownership, riverpod, consumer-widget, tiles]
related: [[domain-vs-screen-state]], [[selector-providers-for-rebuilds]]
---

# Shared widgets stay dumb

## Rule

Shared widgets in `features/shared_widgets/` (and any tile-like primitive reused across features) stay "dumb" — they hold no domain state, do not subscribe to providers, and do not contain conditional logic that pivots their structure based on data. Two interrelated forms of this rule:

1. **Prefer separate tile classes over a single tile with conditional flags.** When two contexts need visually different tiles for related data, create two tiles (`TrackTile`, `AlbumTrackTile`, `EpisodeTile`, `ScheduleTile`, `PlaylistTile`, `ArtistTile`, `ProgramTile`) rather than a single `TrackTileBase` with `bool isAlbum`, `bool isEpisode`, etc. branching inside its `build`.

2. **State comes in as props; mutations go out as callbacks.** Dynamic state the widget needs to render (`isPlaying`, `isSubscribed`, `isLive`) is passed as a `required` constructor parameter, not read via `ref.watch` inside the widget. User actions (`onTap`, `onSubscribeToggle`, `onMenuTap`) are `VoidCallback` parameters; the widget invokes them, the parent decides what they mean.

The parent widget — usually a `ConsumerWidget` list or screen — is the one that watches providers, resolves state for each child, and routes callbacks to notifier methods.

## Rationale

Two pressures push against this rule, and both get resisted:

- *"I'll just add an `isAlbum` flag to `TrackTile` so we don't duplicate."* — flags accrete; a year later that tile has six of them, three interact, and reading the build method requires a flow chart.
- *"I'll have `ScheduleTile` watch `programRemindersProvider` directly so the parent doesn't have to."* — then re-rendering the tile means re-running the provider read, no parent-level optimization is possible, and the tile becomes coupled to a specific feature's provider (so it can't be reused in a context where that provider isn't even present).

Keeping shared widgets dumb makes them composable: the same `ScheduleTile` works inside `ScheduleList` (bound to subscriptions) and could equally work inside a "Mark all as reminded" admin screen (bound to a different provider) — the tile doesn't know where its `isSubscribed` value comes from.

## Implications

- Tiles in `features/shared_widgets/` extend `StatelessWidget`, not `ConsumerWidget`. No `ref.watch` inside.
- Two-flag tile contract: one state prop, one action callback per interactive concern. From `ScheduleTile`:

  ```dart
  class ScheduleTile extends StatelessWidget {
    final ScheduledProgram program;
    final bool isLive;
    final bool isSubscribed;
    final VoidCallback onSubscribeToggle;
    // ...
  }
  ```

- Parent orchestration happens in the surrounding list — `ScheduleList` is a `ConsumerWidget`:

  ```dart
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final subscribedIds = ref.watch(programRemindersProvider);
    // ...
    return ScheduleTile(
      program: program,
      isSubscribed: subscribedIds.contains(program.id),
      onSubscribeToggle: () =>
          ref.read(programRemindersProvider.notifier).toggle(program),
    );
  }
  ```

- When tempted to add a conditional branch inside a shared tile, ask whether the two cases are different enough to be different classes. They usually are.
- `EpisodeTile` and `TrackTile` both expose `final bool? isPlaying` (nullable: `null` means "don't show the equalizer indicator at all") — same convention applied consistently.

## When this principle does NOT apply

- Feature-internal widgets (under `features/<feature>/presentation/<screen>/widgets/`) are allowed to subscribe to feature-specific providers — they're not "shared", they're scoped. The rule targets widgets in `shared_widgets/` and `shared/widgets/`.
- Tightly scoped variation handled by a single flag with no behavior change (e.g. `isCompact` toggling a max-lines value) may stay as one widget. The line is whether the flag changes *what the widget does*, not just sizing tweaks.
- Truly generic atoms (`DottedDivider`, `DSNotificationIcon`) that take only visual props (color, isFilled) are not state-bearing — they're below this rule entirely.
