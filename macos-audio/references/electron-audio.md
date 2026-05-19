# Electron System Audio Capture

Two approaches for capturing system audio in Electron on macOS. Both work without third-party audio drivers (BlackHole, Soundflower).

## Approach 1: Chromium Built-In (macOS 13.2+)

Leverages Chromium's internal ScreenCaptureKit/CoreAudio integration. Zero additional binaries.

### Setup

```typescript
// main.ts
import { session } from 'electron'

session.defaultSession.setDisplayMediaRequestHandler((request, callback) => {
  callback({ video: request.video, audio: 'loopback' })
})
```

```typescript
// renderer.ts — request the stream
const stream = await navigator.mediaDevices.getDisplayMedia({
  video: true,  // required on macOS, even if you only want audio
  audio: true
})

// Extract audio track
const audioTrack = stream.getAudioTracks()[0]
// Remove video track if not needed
stream.getVideoTracks().forEach(t => t.stop())
```

### Extracting PCM in Renderer

MediaStream audio must be processed via Web Audio API in the renderer:

```typescript
const audioContext = new AudioContext({ sampleRate: 16000 })
const source = audioContext.createMediaStreamSource(stream)

await audioContext.audioWorklet.addModule('pcm-processor.js')
const worklet = new AudioWorkletNode(audioContext, 'pcm-processor')

worklet.port.onmessage = (e) => {
  const pcmData: Float32Array = e.data
  // Send to main process via IPC for ASR/storage
  window.electronAPI.sendAudioChunk(pcmData)
}

source.connect(worklet)
worklet.connect(audioContext.destination)
```

### Trade-Offs

- **Permission**: Requires "Screen & System Audio Recording" — triggers the purple screen recording indicator in Control Centre, even for audio-only
- **App restart**: May be required after granting permission on macOS 15+
- **Post-mixer audio**: Recording level follows system volume (turn volume to 0 = silence in recording)
- **macOS version matrix**: Different Chromium flags needed for different macOS versions. Reliable from 13.2+
- **Renderer-only**: Audio processing locked to renderer process; need IPC to get data to main

### Version-Specific Flags

For macOS 14.2+ with Electron 39+, Chromium defaults to Core Audio Taps. For older macOS versions or to force legacy behavior:

```typescript
// Force legacy ScreenCaptureKit path (pre-14.2)
app.commandLine.appendSwitch('disable-features', 'MacCatapLoopbackAudioForScreenShare')
```

### electron-audio-loopback Package

Community package wrapping the above into a cleaner API:

```typescript
import { getLoopbackAudioMediaStream } from 'electron-audio-loopback'

const stream = await getLoopbackAudioMediaStream()
// Returns MediaStream with only audio tracks
```

Options: `forceCoreAudioTap`, `loopbackWithMute`, `sessionOverride`.

## Approach 2: AudioTee.js (macOS 14.2+)

Spawns a native Swift binary (Core Audio Taps) as a child process. Audio streams to main process via stdout.

### Setup

```bash
npm install audiotee
```

```typescript
// main.ts
import { AudioTee } from 'audiotee'

const audiotee = new AudioTee({
  sampleRate: 16000,       // Resamples + converts to 16-bit
  chunkDurationMs: 100     // 100ms chunks
})

audiotee.on('data', (chunk) => {
  // chunk.data is a Buffer containing 16-bit 16kHz mono PCM
  sendToASR(chunk.data)
})

await audiotee.start()
// ... later
await audiotee.stop()
```

### Trade-Offs

- **Permission**: Only "System Audio Recording" — no purple indicator, no screen recording prompt, customizable prompt via `NSAudioCaptureUsageDescription`
- **No restart required** after granting permission
- **Pre-mixer audio**: Volume-independent capture
- **Main process**: Audio data arrives in Node.js main process directly — no IPC overhead
- **Packaging**: Must bundle the Swift binary (~600KB). Requires entitlements for audio capture + library validation. See [packaging guide](https://stronglytyped.uk/articles/packaging-shipping-electron-apps-audiotee)
- **macOS only**: No Windows support yet (WASAPI port in progress)

### Packaging for Distribution

Add to Electron builder config:

```json
{
  "extraResources": [{
    "from": "node_modules/audiotee/bin",
    "to": "audiotee"
  }],
  "mac": {
    "entitlements": "build/entitlements.plist"
  }
}
```

Entitlements must include:
```xml
<key>com.apple.security.cs.disable-library-validation</key>
<true/>
```

## Which to Choose

| Factor | Chromium Built-In | AudioTee.js |
|---|---|---|
| Extra binaries | None | ~600KB Swift binary |
| Permission UX | Screen & Audio + restart | Audio Only, no restart |
| Audio in main process | No (IPC from renderer) | Yes (direct) |
| Volume-independent | No | Yes |
| Windows support | Yes | No (yet) |
| Min macOS | 13.2 | 14.2 |

**Recommendation**: AudioTee.js for macOS-only or macOS-primary apps (better UX, simpler audio pipeline). Chromium built-in for cross-platform or when packaging simplicity matters most.
