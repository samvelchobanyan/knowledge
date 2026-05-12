---
type: decision
severity: required
status: stable
date: 2026-05-12
applies_to: flutter-projects-with-playback
keywords: [audio, playback, just-audio, audio-service, background-playback, lock-screen]
supersedes:
superseded_by:
related: [[two-layer-audio-stack]], [[dual-mode-player-state]]
code_refs:
  - go_sport/pubspec.yaml
  - go_sport/lib/core/audio/app_audio_handler.dart
---

# ADR-002: just_audio + audio_service for playback

## Context

The app needs cross-platform audio playback supporting:

- Music tracks (queue-based, seekable, with shuffle and repeat)
- Live radio streams (URL-based, no seek, with ICY metadata)
- Background playback that continues with the app in background or screen off
- System integration: lock-screen controls, notification controls, headphone-button handling

The choice of playback stack shapes the entire audio architecture — switching later is expensive.

## Decision

Use `just_audio: ^0.9.42` as the underlying player and `audio_service: ^0.18.18` as the system-integration layer. A single `AppAudioHandler` (`core/audio/app_audio_handler.dart`) extends `BaseAudioHandler with QueueHandler, SeekHandler`, owns an `AudioPlayer` from just_audio, and exposes both audio_service's standard surface and additional getters needed by the UI layer (raw `positionStream`, `bufferedPositionStream`, `durationStream`, `icyMetadataStream`, `shuffleIndices`). The structural pattern is captured separately as [[two-layer-audio-stack]].

## Alternatives considered

No alternatives were explicitly recorded at the time of decision.

## Consequences

- (+) `just_audio` handles a wide range of formats (MP3, HLS, DASH, ICY streams) with one API
- (+) `audio_service` integrates with Android `MediaSession` / iOS `MPNowPlayingInfoCenter` automatically once the handler is configured
- (+) `ConcatenatingAudioSource` provides a queue primitive natively — no manual track-end / skip wiring
- (+) ICY metadata stream from `just_audio` exposes "now playing" info for live radio without a separate metadata fetch
- (+) Active community, frequent updates, used in production apps at scale
- (-) `audio_service` adds an Android service declaration and iOS background mode entitlement — boilerplate in platform configs
- (-) Some platform-channel quirks remain (occasional issues with `AudioService.init` on cold start, requiring the try/catch fallback in `main()` shown in [[async-init-provider-override]])
- (-) Two packages instead of one — both need to be kept in compatible versions

## Notes for future projects

Re-evaluate if:

- A unified Flutter audio package emerges that covers the full requirement set (background + lock-screen + ICY) in one dependency
- The app needs DRM-protected streams (just_audio's DRM support is limited; a native-bridged solution may be unavoidable)
- The app drops background-playback requirement entirely — then `just_audio` alone is enough, no `audio_service` needed
