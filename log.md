# Flutter-kb Log

Append-only chronicle of events in the wiki. Three purposes:
- Historical record of when wiki changes happened
- Accumulator of candidates from project work (deposited by project agents during continuous capture)
- Audit trail for lint and re-ingest operations

## Entry format

```
## [YYYY-MM-DD] <event-type> | <context>
<details over one or more lines>
```

## Event types

- `bootstrap` — initial bootstrap completion or partial bootstrap pass
- `ingest` — routine ingest of new knowledge
- `re-ingest` — update of existing pages from changed sources
- `lint` — lint pass with summary of findings
- `create` — targeted page creation
- `candidate` — deposited by a project agent (continuous capture); pending ingest

## Append rules

- Always append at the bottom; never edit existing entries
- Always use the consistent prefix format for parseability
- Always include date in YYYY-MM-DD format

---

## [2026-05-11] init | wiki repository created
Repository initialized at C:\knowledge\flutter-kb\. CLAUDE.md, four templates in _templates/, empty index.md and log.md in place. Bootstrap pending source materials.

## [2026-05-12] candidate | claude-code-desktop | worktree-vs-main-checkout-invisibility
Type: pitfall
Target applies_to: workflows-using-claude-code-desktop
What: Claude Code Desktop creates session worktrees in `.claude/worktrees/` by default. For workflows that require direct file visibility outside the Claude Code session (e.g. wiki bootstrap where the curator needs to review files in Obsidian), this causes invisible writes — files exist on disk but not where the user expects them.
Why interesting: General pitfall of Claude Code Desktop architecture, not project-specific. Applies to any workflow combining Claude Code Desktop with external file viewers/editors (Obsidian vault, IDE indexer, file watcher).
Context: Encountered during wiki bootstrap; 4 calibration pages were written to the worktree path `C:\knowledge\flutter-kb\.claude\worktrees\competent-ritchie-500b53\` and were invisible in the Obsidian vault rooted at `C:\knowledge\flutter-kb\`. Fixed by explicitly requesting writes to the main checkout path.
Source: Claude Code session 2026-05-12, wiki bootstrap calibration step.
Suggested action: Create new pitfall page during future ingest. Workaround: at session start, explicitly request "write files to `<absolute-path>`" rather than relying on the default working directory; or detect worktree mode at session start and request a relocation.

## [2026-05-12] candidate | flutter-kb-system | canonical-branch-for-wiki-operations
Type: principle (methodology)
Target applies_to: flutter-kb-system (curation methodology, not Flutter engineering)
What: Canonical branch for wiki operations is `dev` (or project-specified default). All bootstrap / ingest / re-ingest / lint verifications are against this branch. Continuous capture during in-project work operates on whatever branch the developer is currently on; candidates are canonicalized at next ingest by checking against the canonical branch.
Why interesting: Resolves ambiguity between "wiki describes current code" (which "current" — `dev` or my current feature branch?) and prevents wiki from being polluted by in-flight feature work that may not ship.
Context: Surfaced during 2026-05-12 bootstrap when a calibration pitfall page was written based on code that existed on a non-`dev` branch but was rolled back / never merged to `dev`. Without a canonical-branch rule, the curator agent has no anchor for verification.
Source: Claude Code session 2026-05-12, after Sam pointed out that `branchNavigatorKeyResolver` (referenced in the calibration `modal-in-stateful-shell-route.md`) does not exist on `dev`.
Suggested action: At next CLAUDE.md update, add an explicit section about (a) canonical branch for wiki operations and (b) the relationship between in-project working-branch continuous capture and canonical-branch verification at ingest. Mirror the rule into go_sport's project CLAUDE.md.

## [2026-05-12] candidate | go_sport | modal-in-stateful-shell-route (deferred investigation)
Type: pitfall (open investigation)
Target applies_to: flutter-projects-with-stateful-shell-route
What: Modal opened via `showModalBottomSheet` inside `StatefulShellRoute` resolves to the root navigator, breaking active branch back-stack. A fix using a branch-navigator-key resolver + `useRootNavigator: false` was previously implemented in go_sport but was rolled back during refactoring (timing and reasoning not recorded). Current state on `dev` branch: `MainShell` no longer accepts a `BranchNavigatorKeyResolver`, `FullPlayerScreen.show()` uses `useRootNavigator: true`. Branch navigator keys remain in `app_router.dart` but are not used to resolve the active branch context.
Why interesting: General Flutter pitfall, not project-specific. Cannot be documented as resolved in wiki until investigation determines whether the problem still manifests in current code and whether the original fix should be restored or a different approach taken. Tabling it preserves honesty of the wiki (no contradiction with code) until the underlying state is clarified.
Context: Caught during 2026-05-12 bootstrap calibration. Calibration page `modal-in-stateful-shell-route.md` was written based on the previous (fixed) state and contradicted current `dev`. Page was deleted from main checkout before being canonicalized; this candidate preserves the knowledge for re-evaluation.
Source: Claude Code session 2026-05-12, comparison of `MainShell` / `FullPlayerScreen.show()` between first-pass reading and `dev`-branch re-reading.
Suggested action: Defer to future ingest after Sam investigates current behavior. Open questions: (1) does the back-stack / swipe-back issue still happen on `dev`? (2) is there a different fix that was applied (e.g. the new `RadioFullPlayerScreen` split changes the navigation model)? (3) was the rollback intentional and the current behavior considered acceptable? Worktree copy of the previously-written page remains at `C:\knowledge\flutter-kb\.claude\worktrees\competent-ritchie-500b53\modal-in-stateful-shell-route.md` and can serve as a starting draft when the investigation concludes.

## [2026-05-12] candidate | go_sport | stateful-shell-with-modal-overlays-pattern (paired investigation)
Type: pattern (paired with modal-in-stateful-shell-route pitfall — same investigation)
Target applies_to: flutter-projects-with-stateful-shell-route
What: stateful-shell-with-modal-overlays pattern — describes the branch-navigator-key resolver + useRootNavigator: false approach for opening modals from inside StatefulShellRoute. Was implemented in go_sport earlier, rolled back during refactoring. Pattern is described in conceptual form but should not be canonicalized in wiki until Sam investigates current dev state.
Why interesting: General pattern, paired with the modal-pitfall it solves. Both are deferred together.
Suggested action: Defer to future ingest after Sam investigates. Reconnect with the parallel modal-pitfall candidate (also recorded 2026-05-12) when canonicalized.

## [2026-05-12] candidate | go_sport | spec-vs-code-divergence-statenotifier
Type: divergence (spec vs code)
Target applies_to: project: go_sport
What: ARCHITECTURE_SPEC.md §8.5 shows `StateNotifier<NewsState>` as the canonical state pattern. Current code uses `Notifier` (Riverpod 2 modern API). The two are functionally equivalent here but stylistically diverge.
Why interesting: SPEC documents need to align with current code to prevent confusion for new team members reading SPEC after onboarding.
Suggested action: At next ingest or methodology review, decide: (1) update ARCHITECTURE_SPEC.md to reflect `Notifier` as canonical, OR (2) wiki canonicalizes `Notifier` (already implicitly does via existing pages) and SPEC is treated as historical. Recommend option (1) — keep SPEC and wiki aligned, update SPEC.

## [2026-05-12] bootstrap | initial wiki population
Created 24 pages from `bootstrap-sources/`, all verified against `go_sport` on branch `dev` (canonical branch for wiki operations).

By type: 6 principles, 10 patterns, 7 cross-project ADRs, 1 pitfall.

Sources used for verification:
- `bootstrap-sources/claude-memory.md` — hypotheses-level distilled memory from Claude.ai sessions
- `bootstrap-sources/discussions/discussion-001-program-reminders.md` — full engineering transcript of the program-reminders feature implementation (rich source for the notifications stack)
- `C:\flutter-projects\go_sport\docs\ARCHITECTURE_SPEC.md` and `DOMAIN_SPEC.md` — project architecture specs (close to code, occasionally lag)
- `C:\flutter-projects\go_sport\lib\` on branch `dev` — canonical source of truth
- `C:\flutter-projects\go_sport\android\`, `ios\`, `pubspec.yaml` — platform configs and dependencies

Significant divergences caught during bootstrap (wiki reflects code in all cases):
- `claude-memory.md` claimed a three-layer Player (Notifier + Handler + PlaybackService); code has two layers (`PlayerNotifier` + `AppAudioHandler`). Documented as [[two-layer-audio-stack]].
- `claude-memory.md` claimed split `PlayerQueueState` / `PlaybackState`; code has a single `PlayerState` with `PlaybackMode` discriminator. Documented as [[dual-mode-player-state]].
- `claude-memory.md` claimed `AuthNotifier` was deliberately deferred; it is in fact implemented as a sealed `AuthState` with a `Notifier`. No standalone page; the implementation is illustrated in [[freezed-immutable-state]] and consumed implicitly by [[domain-vs-screen-state]].
- `claude-memory.md` mentioned `GuestSessionNotifier`; the actual implementation is a plain `StatefulWidget` (`GuestTimerBar`) with a local `Timer.periodic`. No page — the simplest viable choice for this concern, and it was already covered by [[avoid-over-engineering]]-style judgment (which we deliberately did not turn into a page).
- `claude-memory.md` mentioned `pinput` for OTP; not in `pubspec.yaml`. Abandoned.
- `claude-memory.md` described slot-based `TrackTileBase` composition; code uses separate tile classes per context (`TrackTile`, `AlbumTrackTile`, `EpisodeTile`, `ScheduleTile`, etc.). Captured correctly as the first form of [[shared-widgets-stay-dumb]].
- ARCHITECTURE_SPEC.md §8.5 shows `StateNotifier` examples; code uses `Notifier`. Wiki tracks code; spec divergence logged as a separate candidate above.

Deferred candidates from this bootstrap (logged earlier in this file, paired so future ingest processes them together):
- `modal-in-stateful-shell-route` pitfall + `stateful-shell-with-modal-overlays` pattern — paired investigation. A fix using `BranchNavigatorKeyResolver` + `useRootNavigator: false` existed in earlier code but was rolled back on `dev`. Reason unknown; current `dev` uses `useRootNavigator: true`. Both pages are tabled until Sam investigates and decides direction.
- `worktree-vs-main-checkout-invisibility` pitfall — Claude Code Desktop UX, surfaced during the bootstrap calibration when 4 pages were initially written to a worktree path invisible to Obsidian.
- `canonical-branch-for-wiki-operations` principle (methodology) — wiki operations verify against `dev`; in-project continuous capture happens on whatever working branch the developer is on, and candidates canonicalize at next ingest after verification against `dev`. Recommend incorporating into both this CLAUDE.md and `go_sport` project CLAUDE.md at next methodology review.

Calibration four-pack written first and reviewed by Sam: [[dto-to-domain-mapping]], [[local-state-via-storage-class]], [[adr-005-secure-storage-for-tokens]], `modal-in-stateful-shell-route`. The fourth was deleted from main checkout after the post-calibration verification pass revealed code on `dev` did not match the documented fix; that knowledge moved to a deferred candidate. The verification methodology — full sweep against canonical branch before canonicalizing remaining pages — also caught the paired `stateful-shell-with-modal-overlays` pattern before it was written. Both catches demonstrate that the verification step is doing real work.

`projects/go_sport/` remains empty by design; first project-scoped ADRs will arrive via continuous capture during in-project work.

## [2026-05-12] candidate | go_sport | domain-vs-screen-state-criterion-refinement
Type: principle (refinement of existing page)
Target applies_to: any-flutter-project
What: Refine the criterion in [[domain-vs-screen-state]] from categorical ("business data vs UI-only state") to operational ("number of consumers"): **global state** = used by ≥2 screens → lives in `domain/state/` as long-lived `Notifier`; **local state** = used by exactly 1 screen → lives in the feature's screen-controller (`features/<feature>/presentation/<screen>/<screen>_controller.dart`), even if it contains domain entities like `List<Album>` or `User`.
Why interesting: The current wiki criterion is subjective and routinely misclassified. A list-of-domain-entities screen (`my_albums`, `album`, `artist`, `playlist`, `program_details`, `schedule`, `music_dashboard`, `my_favorites`, `my_artists`, `my_playlists`, `my_programs`, `new_episodes`) in go_sport keeps its `List<DomainEntity>` inside the screen controller — and that's correct under the operational rule, because each list is consumed by exactly one screen. Moving them to `domain/state/` would force long-lived caching of data that is only meaningful while the screen is mounted. Under Sam's operational rule the existing project layout is consistent, and the few legitimate violations (e.g. `ProfileState.user` — read by both ProfileScreen and EditProfileScreen) become unambiguously identifiable. The categorical rule produced two failure modes: false positives (flagging legitimate local list-state) and false negatives (excusing shared data because it didn't "feel like business data").
Context: Code review of `user_profile` branch in go_sport. Initial flag was `ProfileState.user` as a "domain entity in screen controller" violation. Audit of all `features/*/presentation/*_controller.dart` revealed ≥12 controllers with `List<DomainEntity>` in their state. Sam clarified the project's actual rule: 1 screen → local; 2+ screens → global. Under that rule, `ProfileState.user` is still a violation (EditProfileScreen reads `profileControllerProvider` to prefill fields and re-invokes `getUser()` after update — explicit cross-screen consumption), but the other controllers are not. The current wiki page [[domain-vs-screen-state]] also gives `MiniPlayerWidget._isMusicMode` as an example of "screen state" — that example fits the operational rule cleanly (only the mini-player consumes it).
Source: Claude Code session 2026-05-12, files: `lib/features/user_profile/profile/presentation/profile/profile_controller.dart`, `lib/features/user_profile/edit_profile/presentation/edit_profile/edit_profile_screen.dart` (cross-screen consumer of `User`), plus audit of `lib/features/{albums,artists,playlists,program_details,schedule,music,favorites/*}/presentation/*_controller.dart`.
Suggested action: update [[domain-vs-screen-state]] — replace the categorical framing ("business data vs UI-only") with the operational consumer-count rule. Preserve the existing examples but reframe their justification (`PlayerState` is shared by mini + full + queue screens → global; `MyAlbumsState` is consumed by my_albums_screen alone → local). Keep the secondary guidance about screen controllers not *proxying* domain state (a screen that needs shared state reads the domain provider directly, not via its own controller). Consider adding `User` / profile data as the canonical worked example of "started local, turned out to be shared, needs to be promoted."
