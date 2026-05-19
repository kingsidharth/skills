# Screenshot Capture

## Native Swift: SCScreenshotManager (macOS 14+)

Replaces the deprecated `CGWindowListCreateImage` (obsoleted in macOS 15). Part of ScreenCaptureKit.

```swift
import ScreenCaptureKit

// Get shareable content
let content = try await SCShareableContent.excludingDesktopWindows(false, onScreenWindowsOnly: true)
guard let display = content.displays.first else { return }

// Create filter (full display)
let filter = SCContentFilter(display: display, excludingApplications: [], exceptingWindows: [])

// Configure
let config = SCStreamConfiguration()
config.width = display.width * 2  // Retina
config.height = display.height * 2

// Capture
let image = try await SCScreenshotManager.captureSampleBuffer(contentFilter: filter, configuration: config)
// or: let cgImage = try await SCScreenshotManager.captureImage(contentFilter: filter, configuration: config)
```

### Window-Specific Screenshots

```swift
if let targetWindow = content.windows.first(where: { $0.title == "My Window" }) {
    let filter = SCContentFilter(desktopIndependentWindow: targetWindow)
    let image = try await SCScreenshotManager.captureImage(contentFilter: filter, configuration: config)
}
```

### Permission

Requires Screen Recording permission. Using `SCContentSharingPicker` (the system picker UI) bypasses the need for pre-granted permission — the user's selection implicitly grants access for that window/display.

### Legacy: CGWindowListCreateImage

Still functional on macOS 14 but **obsoleted in macOS 15 SDK** — won't compile against the macOS 15+ SDK. Migrate to `SCScreenshotManager`.

## Electron: desktopCapturer

Electron provides `desktopCapturer.getSources()` for enumerating screens and windows, then uses `navigator.mediaDevices.getUserMedia()` to capture frames.

### Single Screenshot via desktopCapturer

```typescript
// Main process
const { desktopCapturer } = require('electron')

const sources = await desktopCapturer.getSources({
  types: ['screen'],
  thumbnailSize: { width: 1920, height: 1080 }
})

// sources[0].thumbnail is a NativeImage
const pngBuffer = sources[0].thumbnail.toPNG()
fs.writeFileSync('screenshot.png', pngBuffer)
```

### Continuous Capture via MediaStream

```typescript
// Renderer process
const stream = await navigator.mediaDevices.getUserMedia({
  video: {
    mandatory: {
      chromeMediaSource: 'desktop',
      chromeMediaSourceId: sourceId  // from desktopCapturer.getSources
    }
  }
})

const video = document.createElement('video')
video.srcObject = stream
await video.play()

// Capture frame to canvas
const canvas = document.createElement('canvas')
canvas.width = video.videoWidth
canvas.height = video.videoHeight
canvas.getContext('2d').drawImage(video, 0, 0)
const dataUrl = canvas.toDataURL('image/png')
```

### Electron Permission

Requires `NSScreenCaptureUsageDescription` in the app's Info.plist. The parent process (terminal, IDE) needs this key during development.

## macOS `screencapture` CLI

Built-in macOS command. Useful as a fallback or for simple automation:

```bash
# Full screen to file
screencapture screenshot.png

# Specific window (interactive selection)
screencapture -w screenshot.png

# Clipboard
screencapture -c

# Specific display
screencapture -D 1 screenshot.png

# Timed (5 second delay)
screencapture -T 5 screenshot.png

# No shadow on window captures
screencapture -o -w screenshot.png
```

Can be spawned from Swift (`Process`), Node.js (`child_process`), or any host. Doesn't require Screen Recording permission when run from Terminal (inherits Terminal's permission). App-bundled usage requires the app to have Screen Recording permission.

## Choosing an Approach

- **Native Swift app**: Use `SCScreenshotManager` — modern, Retina-aware, filter-capable
- **Electron app**: Use `desktopCapturer` thumbnails for one-shot, MediaStream for continuous
- **Quick automation / scripts**: Use `screencapture` CLI — zero code, pipe-friendly
- **Cross-platform Electron + periodic screenshots**: MediaStream approach with interval-based canvas capture
