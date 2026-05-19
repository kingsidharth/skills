# Audio

## Microphone capture (renderer)

Standard Web API works inside Electron:

```ts
const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
```

On macOS, you must request permission first via the main process or the call fails:

```ts
import { systemPreferences } from 'electron'

const status = systemPreferences.getMediaAccessStatus('microphone')
if (status !== 'granted') {
  const ok = await systemPreferences.askForMediaAccess('microphone')
  if (!ok) throw new Error('mic denied')
}
```

`getMediaAccessStatus` returns `'not-determined' | 'granted' | 'denied' | 'restricted' | 'unknown'`. `'restricted'` means parental controls or MDM — don't show a "grant access" prompt, send the user to System Settings.

## Entitlements

For the packaged app to get past Gatekeeper with mic access, your `entitlements.mac.plist` must include:

```xml
<key>com.apple.security.device.microphone</key><true/>
<key>com.apple.security.device.audio-input</key><true/>
```

And `Info.plist` (electron-builder generates from `extendInfo`):

```yaml
mac:
  extendInfo:
    NSMicrophoneUsageDescription: "MyApp records audio for voice notes."
    NSCameraUsageDescription: "MyApp captures video for screen tutorials."
```

The `*UsageDescription` strings are required — without them, macOS denies access without prompting.

## Permission handler

Even with entitlements set, Electron's session permission handler can deny `media`:

```ts
session.defaultSession.setPermissionRequestHandler((_wc, permission, cb) => {
  cb(['media', 'mediaKeySystem'].includes(permission))
})
```

`media` covers mic, camera, and screen capture together — Electron does not split them like Chrome.

## Screen and system audio capture

```ts
import { desktopCapturer, session } from 'electron'

session.defaultSession.setDisplayMediaRequestHandler((req, cb) => {
  desktopCapturer.getSources({ types: ['screen', 'window'] }).then(sources => {
    cb({ video: sources[0], audio: 'loopback' })
  })
})

// renderer
const stream = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: true })
```

System audio capture on macOS 14.2+ uses Apple's CoreAudio Tap API (default since Electron 39). It requires:

```xml
<!-- entitlements.mac.plist -->
<key>com.apple.security.device.audio-input</key><true/>
```

```yaml
# electron-builder
mac:
  extendInfo:
    NSAudioCaptureUsageDescription: "MyApp captures system audio for screen recording."
```

Without `NSAudioCaptureUsageDescription`, audio capture silently produces a dead stream — no error.

For older macOS (< 13), system audio requires a kernel-extension-based virtual device (BlackHole, Soundflower). Electron cannot do it natively.

## Falling back to legacy permission system

To force the older Screen & System Audio Recording permission flow on macOS 14.2+:

```ts
app.commandLine.appendSwitch('disable-features', 'MacCatapLoopbackAudioForScreenShare')
```

Use only if CoreAudio Tap causes problems for your specific use case.

## Web Audio for processing

The Web Audio API works normally in renderer. `AudioWorklet` is supported. Wire `MediaStream` into an `AudioContext` for analysis, or to a `MediaRecorder` for capture-to-file.

```ts
const ctx = new AudioContext()
await ctx.audioWorklet.addModule('/worklets/level-meter.js')
const node = new AudioWorkletNode(ctx, 'level-meter')
const source = ctx.createMediaStreamSource(stream)
source.connect(node).connect(ctx.destination)
```

Note: under context isolation, `AudioWorklet` URLs need to come from the same origin as the page. With a custom `app://` protocol, register it as `standard: true` so worklets resolve.

## Recording to disk

`MediaRecorder` produces opus/webm. To save:

```ts
const recorder = new MediaRecorder(stream, { mimeType: 'audio/webm;codecs=opus' })
const chunks: Blob[] = []
recorder.ondataavailable = e => chunks.push(e.data)
recorder.onstop = async () => {
  const blob = new Blob(chunks, { type: 'audio/webm' })
  const buf = await blob.arrayBuffer()
  await window.api.audio.save(new Uint8Array(buf))
}
recorder.start()
```

Send the buffer to main for disk write, or use a transferable to avoid copies on big recordings (see `ipc.md` MessagePort).

For mp3 / m4a, run a native module (lame, fdk-aac via WASM, or `fluent-ffmpeg` shelled out) — Chromium does not encode those by default.

## Device enumeration

```ts
const devices = await navigator.mediaDevices.enumerateDevices()
const inputs = devices.filter(d => d.kind === 'audioinput')
```

Devices appear with empty `label` until permission is granted. Always request access first.
