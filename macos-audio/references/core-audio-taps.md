# Core Audio Taps

macOS 14.2+ (December 2023). Lower-level API for tapping system audio output. Captures pre-mixer audio (volume-independent). Only requires "System Audio Recording" permission — no screen recording permission, no purple indicator.

## Architecture

```
System Audio Output
       ↓
   Audio Tap (CATapDescription)     ← intercepts the stream
       ↓
   Aggregate Device                  ← virtual device combining tap(s)
       ↓
   IO Proc callback                  ← your code receives PCM buffers
       ↓
   Your App
```

The tap doesn't replace the audio path — it copies audio data without affecting playback. Setting `mute` on the tap prevents audio from reaching speakers (useful for processing/transformation before playback).

## Setup Sequence

1. **Create a `CATapDescription`** targeting either specific process IDs or all processes
2. **Call `AudioHardwareCreateProcessTap`** with the description → returns a tap `AudioObjectID`
3. **Read the tap's UUID** from its `kAudioTapPropertyUUID` or from the description's `.uuid`
4. **Create an aggregate device** dictionary including the tap UUID in `kAudioAggregateDeviceTapListKey`. Set `kAudioAggregateDeviceIsPrivateKey = true` so it doesn't appear in system audio device lists
5. **Call `AudioHardwareCreateAggregateDevice`** → returns an aggregate device `AudioObjectID`
6. **Read `kAudioTapPropertyFormat`** from the tap to get the `AudioStreamBasicDescription`
7. **Create `AudioDeviceCreateIOProcIDWithBlock`** on the aggregate device — this callback receives audio buffers at the device's IO cycle rate
8. **Start the IO proc** with `AudioDeviceStart`

## Process Targeting

```swift
// All processes (system-wide capture)
var tapDesc = CATapDescription(stereoGlobalTapButExcludeProcesses: [])

// Specific processes only
tapDesc = CATapDescription(stereoMixdownOfProcesses: [pid1, pid2])

// All except specific processes
tapDesc = CATapDescription(stereoGlobalTapButExcludeProcesses: [myAppPID])
```

Excluding your own process prevents feedback. Use `ProcessInfo.processInfo.processIdentifier` to get your PID.

## Volume Independence

Core Audio Taps captures audio before the system volume mixer. Even if the user sets volume to 0 or mutes speakers, the tap receives full-amplitude audio. This is a key advantage over ScreenCaptureKit's audio capture for recording scenarios.

## Permission

Requires `NSAudioCaptureUsageDescription` in Info.plist. The system prompts with "System Audio Recording" permission only (not "Screen & System Audio Recording"). No app restart required after granting.

There is no public API to pre-check permission status. The prompt appears on first tap creation attempt. Workaround: attempt to create a tap, catch the failure, and guide the user.

For permission-checking patterns, see [insidegui/AudioCap](https://github.com/insidegui/AudioCap) which reverse-engineers the permission check.

## Cleanup

Destroy resources in reverse order:
1. `AudioDeviceStop` the aggregate device
2. `AudioDeviceDestroyIOProcID` to remove the callback
3. `AudioHardwareDestroyAggregateDevice` 
4. `AudioHardwareDestroyProcessTap`

Failing to clean up leaves zombie aggregate devices. The `1852797029` OSStatus error on tap creation often means a previous aggregate device wasn't destroyed.

## AudioTee (Pre-Built Swift CLI)

[AudioTee](https://github.com/makeusabrew/audiotee) wraps Core Audio Taps into a standalone Swift CLI binary (~600KB universal). Streams raw PCM to stdout. Useful for:
- Electron apps (spawn as child process via [AudioTee.js](https://github.com/makeusabrew/audioteejs))
- Any Node.js / Python / Rust host that can spawn a subprocess
- Quick prototyping without writing CoreAudio boilerplate

Key flags:
- `--sample-rate 16000` — Resamples + converts to 16-bit signed integer
- `--stereo` — Output stereo (default mono)
- `--mute` — Mute tapped processes (audio doesn't reach speakers)
- `--include-processes PID1 PID2` / `--exclude-processes PID1 PID2`
- `--chunk-duration 0.1` — Control buffer chunk size (default 0.2s)

All audio goes to stdout, all logs to stderr. Pipe-friendly design.

## Limitations

- macOS 14.2+ only (despite Apple docs sometimes showing 26.0+ — this is a documentation error; the API has worked since 14.2)
- Only taps the default output device (no multi-device support yet)
- No built-in echo cancellation — if you also capture mic, you need separate AEC handling
- No mic capture — Core Audio Taps is system output only. Pair with AVAudioEngine for mic
