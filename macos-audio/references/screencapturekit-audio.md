# ScreenCaptureKit Audio Capture

macOS 13+ (Ventura). Apple's recommended high-level framework for capturing display content including audio. No third-party drivers needed.

## Core Concepts

**SCShareableContent** — Enumerates available displays, windows, and running applications. First call triggers the permission prompt.

**SCContentFilter** — Defines what to capture. For audio-only, you still need a display filter (the framework requires it), but you can ignore the video frames.

**SCStreamConfiguration** — Controls output format: resolution, frame rate, audio settings. Key audio properties:
- `capturesAudio` — Enables system audio capture
- `captureMicrophone` — Enables mic capture (macOS 14+). Delivers on a separate `.microphone` output type
- `excludesCurrentProcessAudio` — Prevents feedback loops from your own app's audio
- `sampleRate` — Default 48000. Supports 8000–96000
- `channelCount` — Default 2 (stereo). Set 1 for mono
- Audio capture is read-only from the system mixer — it does not alter playback volume or output routing

**SCStream** — The active capture session. Add stream outputs for different types (`.screen`, `.audio`, `.microphone`).

**SCStreamOutput protocol** — Receives `CMSampleBuffer` via `stream(_:didOutputSampleBuffer:of:)`. The `type` parameter distinguishes `.audio` (system) from `.microphone`.

## Audio-Only Capture Pattern

Even for audio-only, SCStream requires a content filter with a display. Set minimal video config to reduce overhead:

```swift
// Minimal video config to reduce resource usage
config.width = 2
config.height = 2
config.minimumFrameInterval = CMTime(value: 1, timescale: 1) // 1 fps minimum
config.capturesAudio = true
config.sampleRate = 48000
config.channelCount = 1
config.excludesCurrentProcessAudio = true
```

Register only `.audio` output — skip `.screen` to avoid processing video frames you don't need. The stream still technically captures video but your delegate won't receive it.

## Dual-Stream Gotcha: System Audio + Mic

When `captureMicrophone = true`, the framework delivers two separate sample buffer types through the same delegate:
- `.audio` — System/app audio
- `.microphone` — Mic input

These arrive with **different `CMFormatDescription`s** (potentially different sample rates, channel counts). Writing both to a single `AVAssetWriterInput` corrupts the output. Use separate writer inputs per stream type, or process them independently.

## Writing to File with AVAssetWriter

For each audio type, create a matching `AVAssetWriterInput` from the first buffer's format description:

```swift
// On first buffer arrival per type:
let formatDesc = CMSampleBufferGetFormatDescription(sampleBuffer)!
let audioSettings = [
    AVFormatIDKey: kAudioFormatMPEG4AAC,
    AVSampleRateKey: 48000,
    AVNumberOfChannelsKey: 1,
    AVEncoderBitRateKey: 128000
] as [String: Any]
let writerInput = AVAssetWriterInput(mediaType: .audio, outputSettings: audioSettings, sourceFormatHint: formatDesc)
```

## SCRecordingOutput (macOS 15+)

Newer alternative to manual AVAssetWriter. Add an `SCRecordingOutput` to the stream and it writes directly to a file. Simpler but less control over format. Delegates: `recordingOutputDidStartRecording`, `recordingOutputDidFinishRecording`.

Known issue: recording stops if `SCStreamConfiguration` is updated on a running stream. Updating only the `SCContentFilter` is safe.

## Streaming Raw PCM

For real-time processing (ASR, silence detection), extract PCM from `CMSampleBuffer`:

```swift
func stream(_ stream: SCStream, didOutputSampleBuffer sampleBuffer: CMSampleBuffer, of type: SCStreamOutputType) {
    guard type == .audio, sampleBuffer.isValid else { return }
    
    guard let blockBuffer = CMSampleBufferGetDataBuffer(sampleBuffer) else { return }
    var length = 0
    var dataPointer: UnsafeMutablePointer<Int8>?
    CMBlockBufferGetDataPointer(blockBuffer, atOffset: 0, lengthAtOffsetOut: nil, totalLengthOut: &length, dataPointerOut: &dataPointer)
    
    guard let data = dataPointer else { return }
    let pcmData = Data(bytes: data, count: length)
    // pcmData is Float32 PCM at configured sample rate / channel count
}
```

## Permission & Lifecycle

- First call to `SCShareableContent.current` triggers the system permission prompt
- Permission stored in System Settings → Privacy & Security → Screen & System Audio Recording
- On macOS 15+, granting permission may require an app restart
- The purple "screen recording" indicator appears in Control Centre even for audio-only capture — this is a known UX issue with no workaround
- Stream survives display sleep but stops on user logout

## Error Handling

Common `SCStreamError` codes:
- `-3801` (`attemptToStartStreamTwice`) — Stream already running
- `-3805` (`connectionInvalid`) — App not properly signed, or permission revoked
- `-3808` (`failedToStart`) — Another stream conflict, or entitlements missing

Implement `SCStreamDelegate.stream(_:didStopWithError:)` to handle unexpected stops. Known crash: extended captures (hours) can hit `EXC_BAD_ACCESS` in the error delegate — wrap with crash protection / watchdog restart.
