# Claude.ai Memory Dump

**Source:** Distilled memory from Claude.ai conversations between Sam and Claude over the course of go_sport project development.

**Status: hypotheses, not facts.** This document captures what was **discussed and intended** during Claude.ai sessions. It does NOT necessarily reflect what was actually implemented in code. Some decisions described here may have been:
- Implemented as-is in the project
- Modified during implementation in Claude Code based on practical considerations
- Abandoned in favor of different approaches
- Never realized at all

**Verification required.** Before any claim from this document is fixed into a wiki page, it must be verified against the actual project code at `C:\flutter-projects\go_sport\`. The code is the source of truth; this memory is the hypothesis generator. See `CLAUDE.md` Section 1 (source-of-truth hierarchy) and Section 6.1 (bootstrap protocol).

---

## Purpose & context

Sam is building a Flutter mobile application called "go_sport" — a music and radio streaming app. The project is well into active development with a substantial, established architecture. Core technical stack: Flutter/Dart, Riverpod (state management), Freezed (immutable state/sealed classes), just_audio + audio_service (playback), Dio (networking), go_router (navigation), and FlutterSecureStorage (token persistence).

The architecture follows a layered, feature-based structure: `core/`, `domain/`, `data/`, `features/`, with a `shared_widgets/` directory for cross-feature UI components. A formal `ARCHITECTURE_SPEC.md` guides decisions. The design system uses token classes (`DSColors`, `DSSpacing`, `DSRadius`, `DSTypography`) and does not store components — those live in feature or shared widget folders.

Sam works with a team that includes junior developers, which influences naming, explicitness, and architectural documentation decisions. An agentic AI workflow is in use: architectural discussions happen in Claude.ai (leveraging persistent memory), while implementation is delegated to Claude Code (VSCode plugin) acting as the implementer.

---

## Current state (as discussed)

The app has progressed through foundational architecture into feature implementation. Established and implemented areas (per discussion) include:

- **Player**: `PlayerNotifier` (state + business logic), `AppAudioHandler` (audio infrastructure), `PlaybackService` layer for orchestration, background playback via audio_service, queue management with `ConcatenatingAudioSource`, separate Music vs. Radio playback modes, `PlayerQueueState` / `PlaybackState` split for performance, progress sync with backend
- **Mini player**: Animated music/radio mode switching, `playerProgressProvider` and `playerInfoProvider` selectors to minimize rebuilds, progress bar with DSColors theming
- **Full-screen player**: Modal bottom sheet pattern (consistent with StoryOverlay), `PlayerArtworkCarousel` using `PageView.builder` with `viewportFraction`, GLSL fragment shader animated fluid background (`blob.frag` + `PlayerFluidBackground`), custom seek bar with `_RoundedRectThumbShape`, `_FullWidthTrackShape`, buffered position display
- **Navigation**: go_router with `StatefulShellRoute.indexedStack` for tab state preservation, `MainShell` wrapper, global routes outside the shell for cross-tab screens, Material 3 `NavigationBar` with DSColors customization
- **API & auth infrastructure**: `TokenStorage` (in-memory + `FlutterSecureStorage`), `AuthInterceptor`, `ApiClient` accepting `AppConfig` and interceptors, `AppConfig.dev`/`AppConfig.prod` via `--dart-define=ENV`, DTO pattern with `fromJson` + `toDomain()`
- **Home**: Story overlay feature (modal bottom sheet, fullscreen, within `features/home/presentation/story/`)
- **Music feature**: `MusicDashboardController` with parallelized data loading via Dart 3 Records `.wait`, consolidated `features/music/presentation/` structure for favorites/library screens
- **Radio feature**: `features/radio/presentation/` with `radio/` and `schedule/` subdirectories; Radio Player lives in `features/player/`; `GuestSessionNotifier` for guest timer/expiry logic; `GuestBanner` in shell scaffold

Active or recently addressed:
- OTP input: `pinput` package selected over separate `TextField` widgets
- Playlist screen loading skeleton: refactoring to `PlaylistTracksSkeletonSliver` (a `SliverList`-based widget), conceptually aligned but not yet coded
- Guest banner + countdown timer architecture, awaiting implementation confirmation

---

## On the horizon (as discussed)

- Implementing `PlaylistTracksSkeletonSliver` code (session ended at conceptual stage)
- Implementing `GuestSessionNotifier`, `GuestBanner`, and shell integration code (session ended awaiting confirmation)
- Real API calls replacing mock repositories (e.g., `PlaylistRepository`)
- `AuthNotifier` for reactive auth status and logout orchestration (deliberately deferred)
- Riverpod 3.0 offline persistence integration (experimental feature noted, not yet integrated)
- Hero animations (interest expressed, deferred until basic transitions stabilized)
- Radio domain entities finalization (`Radio`, `RadioScheduleSlot`, `Track`, `Program`) and clarifying whether on-air/ICY metadata arrives via API response or separate stream

---

## Key learnings & principles (as discussed)

- **Avoid over-engineering**: Sam consistently pushes back on premature abstraction. Providers, layers, and patterns are added only when actually needed (e.g., deferred `AuthNotifier`, deferred config/tokenStorage providers).
- **Composition over conditionals**: Slot-based composition (`TrackTileBase` with named slots) is preferred over conditional logic inside shared widgets.
- **Domain state as cache**: Domain-level `AsyncNotifier` state serves as the cache layer; repositories do not cache independently.
- **Synchronization via state watching**: When a screen's tracks are loaded into `PlayerState`, the screen switches to watching `PlayerState` directly — elegant sync without complex source-switching.
- **Selector-based rebuild optimization**: Derived providers (`playerProgressProvider`, `playerInfoProvider`) isolate high-frequency updates (position ticks) from low-frequency ones (track changes), preventing unnecessary widget rebuilds.
- **Explicit > implicit**: Sam prefers readable, explicit code patterns over "magical" framework patterns, especially for team legibility.
- **Semantic vs. navigational ownership**: Navigation structure follows user flow ("back goes where you came from"), not content domain ownership.
- **Plan before implement**: Sam prefers conceptual alignment before writing code, often asking to slow down and confirm understanding first.

---

## Approach & patterns (work style)

- Discusses architecture and trade-offs with Claude.ai; delegates implementation to Claude Code (VSCode plugin)
- Asks clarifying questions at each decision point rather than accepting recommendations wholesale; pushes back when something feels over-engineered
- Prefers understanding the "why" behind patterns (asks for before/after comparisons, root cause explanations)
- Communicates in Russian for technical discussions; code remains in English
- Uses Figma as design source; exports SVG icons without hardcoded fill attributes for runtime color flexibility
- Builds incrementally: structure first, then components, then integration
- Maintains `ARCHITECTURE_SPEC.md` as a living document updated when architectural decisions diverge from original specs

---

## Tools & resources

- **IDE**: VSCode with Claude Code plugin (implementation agent)
- **AI workflow**: Claude.ai (architecture discussions) + Claude Code CLI/plugin (implementation)
- **Key packages**: Riverpod, Freezed, just_audio, audio_service, Dio, go_router, FlutterSecureStorage, cached_network_image, pinput, flutter_svg
- **Design**: Figma (source of truth for UI); custom design system with DSColors, DSSpacing, DSRadius, DSTypography
- **Platform**: Windows development environment (Claude Code CLI installation via PowerShell or npm)
