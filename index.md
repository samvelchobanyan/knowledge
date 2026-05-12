# Flutter-kb Index

This is the navigation backbone of the wiki. Agents read this file first when working with the knowledge base. Maintained automatically by Claude Code; do not edit by hand.

Each entry has the format:

```
- [[page-name]] — severity: <value>, applies_to: <value> — <one-line description>
```

For project-scoped ADRs, entries are grouped under "Project-specific knowledge" with subheadings per project.

---

## Principles

- [[design-system-tokens-mandatory]] — severity: required, applies_to: any-flutter-project — All visual values come from DSColors / DSSpacing / DSRadius / DSTypography; no hardcoded colors, sizes, or font styles in features
- [[domain-vs-screen-state]] — severity: required, applies_to: any-flutter-project — Domain state in domain/state/ (long-lived Notifier); screen state in feature controller (short-lived); screens read domain state directly
- [[dto-to-domain-mapping]] — severity: required, applies_to: any-flutter-project — All API responses parsed via DTOs with fromJson + toDomain(); DTOs never leave the data layer
- [[layered-architecture]] — severity: required, applies_to: any-flutter-project — Five layers (core, design_system, domain, data, features) with strict downward dependency direction
- [[semantic-vs-navigational-ownership]] — severity: required, applies_to: any-flutter-project — Navigation follows user flow, not content category; news under Home, library screens under Music, schedule under Radio
- [[shared-widgets-stay-dumb]] — severity: required, applies_to: any-flutter-project — Shared tiles hold no state and no provider subscriptions; separate classes over conditional flags; state via props, actions via callbacks

## Patterns

- [[api-client-with-config-and-interceptors]] — severity: recommended, applies_to: any-flutter-project — ApiClient wraps Dio, takes AppConfig + interceptor list at construction; env selection via --dart-define=ENV
- [[async-init-provider-override]] — severity: recommended, applies_to: any-flutter-project — Provider<T>(throw) declaration + ProviderScope.overrides for infrastructure that needs await init()
- [[dual-mode-player-state]] — severity: recommended, applies_to: flutter-projects-with-playback — Single PlayerState with PlaybackMode discriminator; switching modes preserves the other mode's queue
- [[event-stream-vs-state-for-discrete-events]] — severity: recommended, applies_to: any-flutter-project — Discrete events (notification taps, deep links) as Stream + StreamProvider, not StateProvider — avoids stale-state and dedup pitfalls
- [[freezed-immutable-state]] — severity: recommended, applies_to: any-flutter-project — Freezed @freezed for product types + sealed Freezed for sum types; copyWith / == / hashCode generated, Dart 3 patterns supported
- [[local-notifications-stack]] — severity: recommended, applies_to: flutter-projects-with-local-notifications — NotificationService + ReminderStorage + Notifier orchestrator + dumb tile; four-layer separation for scheduled local notifications
- [[local-state-via-storage-class]] — severity: recommended, applies_to: any-flutter-project — Concrete storage class (in-memory cache + sync getters + async init) for local non-API state; no abstract repository
- [[parallel-data-loading-with-records]] — severity: flexible, applies_to: any-flutter-project — Dart 3 record .wait to fan out independent repository calls in parallel
- [[selector-providers-for-rebuilds]] — severity: recommended, applies_to: any-flutter-project — Narrow Provider<T> selectors over a large Notifier; isolate high-frequency from low-frequency updates
- [[two-layer-audio-stack]] — severity: recommended, applies_to: flutter-projects-with-playback — PlayerNotifier (state + user API) + AppAudioHandler (just_audio + audio_service infrastructure); two layers, not three

## Decisions (cross-project)

- [[adr-001-riverpod-for-state-management]] — severity: required, applies_to: any-flutter-project — Riverpod 2.x chosen as the single state-management and DI mechanism
- [[adr-002-just-audio-plus-audio-service]] — severity: required, applies_to: flutter-projects-with-playback — just_audio + audio_service stack for playback with background and lock-screen support
- [[adr-003-freezed-for-models]] — severity: required, applies_to: any-flutter-project — Freezed + json_serializable for entities, state classes, and DTOs via build_runner
- [[adr-004-go-router-stateful-shell]] — severity: required, applies_to: any-flutter-project — go_router with StatefulShellRoute.indexedStack for tab state preservation and deep-link resolution
- [[adr-005-secure-storage-for-tokens]] — severity: required, applies_to: any-flutter-project — FlutterSecureStorage for auth tokens, fronted by TokenStorage with in-memory cache and synchronous getters
- [[adr-006-flutter-local-notifications-plus-timezone]] — severity: required, applies_to: flutter-projects-with-local-notifications — flutter_local_notifications + timezone for locally-scheduled reminders; FCM alternative rejected
- [[adr-007-inexact-alarm-as-default]] — severity: recommended, applies_to: flutter-projects-with-local-notifications — AndroidScheduleMode.inexactAllowWhileIdle as default to avoid Play Store policy risk and Android 14 UX detour

## Pitfalls

- [[local-notifications-platform-gotchas]] — severity: required, applies_to: flutter-projects-with-local-notifications — Android core library desugaring requirement (FLN 17+); iOS UNUserNotificationCenter delegate must be registered; vendor battery optimization kills scheduled notifications on Xiaomi/Huawei/Oppo

## Project-specific knowledge

<no project-scoped pages yet — first project-scoped ADRs will arrive via continuous capture during in-project work>
