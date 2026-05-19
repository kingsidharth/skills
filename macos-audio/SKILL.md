---
name: macos-audio-capture
description: "Capture system audio, microphone audio, and screenshots on macOS. ScreenCaptureKit, Core Audio Taps, AVAudioEngine, SCStream, SCStreamConfiguration, loopback audio, echo cancellation, noise suppression, voice processing, SCScreenshotManager, Electron desktopCapturer, AudioTee, aggregate device, CATapDescription, silence detection, screen lock persistence, CMSampleBuffer, AVAssetWriter, PCM audio streaming. Use when building meeting recorders, transcription apps, screen recording tools, or any app capturing system/mic audio on macOS."
---

# macOS Audio & Screenshot Capture

Capture system audio output, microphone input, and screenshots on macOS without touching system volume levels or requiring third-party audio drivers.

## When to Read Reference Files

| Scenario | Reference File |
|---|---|
| Capture system audio via Apple's recommended framework (macOS 13+) | [references/screencapturekit-audio.md](references/screencapturekit-audio.md) |
| Capture system audio via Core Audio Taps API (macOS 14.2+) | [references/core-audio-taps.md](references/core-audio-taps.md) |
| Microphone capture with echo cancellation / noise suppression | [references/mic-voice-processing.md](references/mic-voice-processing.md) |
| Audio processing: silence detection, level normalization, compression | [references/audio-processing.md](references/audio-processing.md) |
| Screenshot capture (Swift native + Electron route) | [references/screenshot-capture.md](references/screenshot-capture.md) |
| Electron integration for system audio loopback | [references/electron-audio.md](references/electron-audio.md) |
| Persistence: screen lock, app restart, session lifecycle | [references/persistence-lifecycle.md](references/persistence-lifecycle.md) |

## Decision: Which Audio Capture API

Three viable APIs exist. Choose based on your constraints:

**ScreenCaptureKit** (macOS 13+) — Apple's primary recommendation. Captures system audio + mic in a single `SCStream`. Also captures video frames if needed. Requires "Screen & System Audio Recording" permission. Best when you also need screen capture, or want a single unified stream of system + mic audio.

**Core Audio Taps** (macOS 14.2+) — Lower-level. Taps audio pre-mixer (volume-independent). Only requires "System Audio Recording" permission (no screen recording). Best for audio-only apps (transcription, meeting recorders) where you don't want misleading screen-recording indicators. Can target specific processes.

**AVAudioEngine with Voice Processing** (macOS 10.15+) — Mic-only with built-in AEC/noise suppression. Does not capture system audio. Use for mic input when echo cancellation matters (VoIP, meeting recording where you also capture system audio separately).

| Concern | ScreenCaptureKit | Core Audio Taps | AVAudioEngine |
|---|---|---|---|
| System audio | Yes | Yes | No |
| Mic audio | Yes (macOS 14+) | No (mic separate) | Yes |
| Echo cancellation | No | No | Yes (voice processing mode) |
| Permission type | Screen & System Audio | System Audio Only | Microphone |
| Volume-independent | Yes | Yes (pre-mixer) | N/A |
| Process filtering | Via SCContentFilter | Via CATapDescription | N/A |
| Min macOS | 13.0 | 14.2 | 10.15 |

**Typical meeting-recorder architecture**: Core Audio Taps (system audio) + AVAudioEngine with voice processing (mic with AEC) → two separate PCM streams merged or kept as separate tracks.

## Entitlements & Permissions

All audio capture requires the app to be signed and have appropriate Info.plist keys:

- `NSAudioCaptureUsageDescription` — Required for Core Audio Taps and ScreenCaptureKit audio. Customizable prompt text.
- `NSMicrophoneUsageDescription` — Required for mic access (AVAudioEngine, or SCStream with `captureMicrophone`).
- `NSScreenCaptureUsageDescription` — Required for ScreenCaptureKit (even audio-only, since it goes through the screen recording permission path).

Sandbox entitlements (if sandboxed):
- `com.apple.security.device.audio-input` — Mic
- `com.apple.security.device.screen-capture` — ScreenCaptureKit

For Electron apps, the parent process (terminal, IDE, or the .app bundle itself) must contain the relevant Info.plist keys.

## Audio Format Defaults

For speech/meeting recording, prefer:
- **Sample rate**: 48kHz capture → downsample to 16kHz for ASR services
- **Channels**: Mono (sufficient for speech; halves bandwidth)
- **Bit depth**: 16-bit signed integer (after any sample rate conversion)
- **Format**: Raw PCM for streaming; AAC/OPUS for file storage

ScreenCaptureKit defaults: 48kHz stereo Float32. Core Audio Taps: matches output device (typically 44.1kHz or 48kHz, 32-bit float).
