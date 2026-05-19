# Persistence & Lifecycle

Handling screen lock, display sleep, app termination, and session recovery for long-running audio capture.

## Screen Lock & Display Sleep

### ScreenCaptureKit

- **Display sleep**: SCStream continues delivering audio buffers. Video frames stop or contain blank content.
- **Screen lock (Ctrl+Cmd+Q)**: Audio capture continues. The stream is not interrupted by the lock screen.
- **User switch (Fast User Switching)**: Stream stops. You receive `stream(_:didStopWithError:)`. Must restart capture when the user switches back.
- **Lid close on MacBook**: If set to sleep, the stream stops. If configured to stay awake (external display, `caffeinate`, or power settings), audio continues.

### Core Audio Taps

- **Display sleep**: Tap continues. Audio flows as long as apps are producing sound.
- **Screen lock**: Tap continues.
- **System sleep**: Tap stops. The IO proc callback stops firing. Must re-establish on wake.
- **Lid close**: Same as system sleep unless prevented.

### AVAudioEngine

- **Display sleep / lock**: Engine continues running. Mic capture continues.
- **System sleep**: Engine stops. Must restart on wake.
- **Audio route change** (headphones plugged/unplugged): Engine may need reconfiguration. Listen for `AVAudioSession.routeChangeNotification` (iOS) or monitor `kAudioDevicePropertyDeviceHasChanged` on macOS.

## Preventing Sleep

For continuous recording, prevent system sleep:

### ProcessInfo (Swift)

```swift
// Prevent sleep while recording
let activity = ProcessInfo.processInfo.beginActivity(
    options: [.userInitiated, .idleSystemSleepDisabled],
    reason: "Audio recording in progress"
)

// When done:
ProcessInfo.processInfo.endActivity(activity)
```

### IOKit Power Assertion (Lower Level)

```swift
import IOKit.pwr_mgt

var assertionID: IOPMAssertionID = 0
IOPMAssertionCreateWithName(
    kIOPMAssertPreventUserIdleSystemSleep as CFString,
    IOPMAssertionLevel(kIOPMAssertionLevelOn),
    "Audio recording active" as CFString,
    &assertionID
)

// Release when done:
IOPMAssertionRelease(assertionID)
```

### caffeinate (CLI / Electron)

```bash
# Prevent sleep for child process lifetime
caffeinate -i -w $PID
```

From Electron/Node.js:
```typescript
const { spawn } = require('child_process')
const caffeinate = spawn('caffeinate', ['-i', '-w', String(process.pid)])
// Kill when recording stops
caffeinate.kill()
```

## App Termination & Restart Recovery

### Graceful Shutdown

Register for termination signals to flush buffers:

```swift
// Swift
NotificationCenter.default.addObserver(
    forName: NSApplication.willTerminateNotification,
    object: nil, queue: .main
) { _ in
    // Flush AVAssetWriter
    // Stop SCStream
    // Destroy Core Audio Taps
}

// Handle SIGTERM/SIGINT
signal(SIGTERM) { _ in cleanup() }
signal(SIGINT) { _ in cleanup() }
```

```typescript
// Electron
app.on('before-quit', async (e) => {
  e.preventDefault()
  await stopRecording()
  await flushBuffers()
  app.exit(0)
})

process.on('SIGTERM', async () => { await cleanup(); process.exit(0) })
```

### State Persistence for Recovery

Save recording state to disk periodically:

```swift
struct RecordingState: Codable {
    var isRecording: Bool
    var startTime: Date
    var outputFilePath: String
    var lastChunkTimestamp: TimeInterval
}

// Save every N seconds or on each chunk
func persistState(_ state: RecordingState) {
    let data = try? JSONEncoder().encode(state)
    data?.write(to: stateFileURL, options: .atomic)
}
```

On app launch, check for interrupted recording state and offer to resume or recover the partial file.

### Segmented Recording

Instead of one long file, write audio in segments (e.g., 60-second chunks):
- If the app crashes, you lose at most one segment
- Segments can be concatenated post-hoc
- Easier to manage file I/O and memory

```swift
// Rotate file every 60 seconds
Timer.scheduledTimer(withTimeInterval: 60, repeats: true) { _ in
    finishCurrentSegment()
    startNewSegment()
}
```

## Watching for Audio Route Changes

Monitor when the user plugs/unplugs headphones, switches to Bluetooth, etc.:

```swift
// macOS: Watch default output device changes
let deviceID = AudioObjectID(kAudioObjectSystemObject)
var address = AudioObjectPropertyAddress(
    mSelector: kAudioHardwarePropertyDefaultOutputDevice,
    mScope: kAudioObjectPropertyScopeGlobal,
    mElement: kAudioObjectPropertyElementMain
)

AudioObjectAddPropertyListenerBlock(deviceID, &address, .main) { _, _ in
    // Default output device changed
    // Core Audio Taps may need recreation
    // ScreenCaptureKit continues on its own
}
```

## Electron-Specific: Main Process Crash Recovery

Use Electron's `crashReporter` and `app.requestSingleInstanceLock()`:

```typescript
const gotLock = app.requestSingleInstanceLock()
if (!gotLock) {
  // Another instance running, or recovering from crash
  app.quit()
}

app.on('second-instance', () => {
  // Focus existing window
})
```

Combine with file-based state to detect and recover interrupted sessions on relaunch.

## Login Item / Auto-Start

For apps that should resume recording on boot:

```swift
// Swift (macOS 13+)
import ServiceManagement
try SMAppService.mainApp.register()  // Add to login items
```

```typescript
// Electron
app.setLoginItemSettings({
  openAtLogin: true,
  openAsHidden: true  // Start in background
})
```
