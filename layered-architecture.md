---
type: principle
severity: required
status: stable
applies_to: any-flutter-project
keywords: [architecture, layers, separation-of-concerns, dependency-direction, core, domain, data, features, design-system]
related: [[dto-to-domain-mapping]], [[domain-vs-screen-state]], [[design-system-tokens-mandatory]]
---

# Layered architecture

## Rule

The project is organized into five layers under `lib/`:

- **`core/`** — infrastructure: network (`ApiClient`, interceptors), audio (`AppAudioHandler`), auth (`TokenStorage`), navigation (`app_router`, `MainShell`), DI (`*_providers.dart`), notifications. Knows nothing about specific features.
- **`design_system/`** — visual tokens (`DSColors`, `DSSpacing`, `DSRadius`, `DSTypography`), shared visual components, theme. Knows nothing about features or domain.
- **`domain/`** — entities (Freezed value objects), repository interfaces (abstract classes), and domain state (long-lived `Notifier`s). Pure Dart. No Flutter widgets, no JSON, no HTTP.
- **`data/`** — DTOs (`*_dto.dart`) and repository implementations (`*_repository_impl.dart`). Talks to `core/network/`, returns domain entities. Never reaches into features.
- **`features/`** — presentation layer. Screens (`*_screen.dart`), screen controllers (`*_controller.dart`), feature-local widgets. May depend on domain abstractions and `design_system/`; never on `data/dto/` or repository implementations directly.

Plus `shared/widgets/` for cross-feature low-level helper widgets (e.g. small primitives like `EqualizerIndicator`) and `features/shared_widgets/` for cross-feature UI atoms with feature-domain meaning (tiles, banners, dotted dividers).

Dependency direction is strictly downward: `features → domain ← data`, with `core` and `design_system` providing horizontal infrastructure that any layer may use. There is no path that goes the other way.

## Rationale

Five layers may sound like a lot, but each carries a single concern and has a precise relationship to the others. Two boundaries matter most:

- **Features must not import from `data/`.** The data layer is implementation; the domain is the contract. If a screen imports a DTO, it has just coupled its rendering to API field names, and an API rename now touches the UI.
- **Domain must not depend on Flutter.** A domain entity that knows about `BuildContext` or `Material` cannot be tested in isolation, used in isolates, or repurposed in a future non-Flutter context (a CLI tool, a server-side validation script).

The other rules fall out of these: `core/network/` owns the only HTTP client; only `data/` uses it. `design_system/` is leaf-level (referenced by everyone, references no one above it).

## Implications

- `lib/` has these top-level entries: `core/`, `design_system/`, `domain/`, `data/`, `features/`, `shared/`, plus `main.dart`
- `ApiClient` lives only in `core/network/`. `data/repositories/*_impl.dart` is the only place that injects it. Features never construct or read it.
- Repository **interfaces** live in `domain/repositories/`, **implementations** in `data/repositories/`. Riverpod providers in `core/di/repository_providers.dart` bind the interface to the impl.
- DTOs in `data/dto/` never leave the data layer (see [[dto-to-domain-mapping]])
- Domain state `Notifier`s live in `domain/state/`, not in features (see [[domain-vs-screen-state]])
- `design_system/foundations/` is referenced from anywhere visual; `features/` never hardcodes colors or sizes (see [[design-system-tokens-mandatory]])
- New cross-cutting infrastructure (e.g. notifications, analytics) gets its own folder under `core/`, not a feature folder. Example: `core/notifications/` for the `flutter_local_notifications` stack.

## When this principle does NOT apply

- Tiny utilities with no architectural pretensions (a date-formatting helper, a string extension) can live in `shared/` without forcing the five-layer hierarchy.
- One-off scripts under `tool/` or `scripts/` are out of scope; they live next to the project root and may freely reach into anything.
- Tests in `test/` mirror `lib/` but are not themselves a layer in the dependency rules — they're allowed to import anything they need to verify.
