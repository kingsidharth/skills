# Permissions and entitlements

Two layers on macOS: **OS permissions** (the user-facing prompts) and **entitlements** (declared in your signed binary).

Both must align. Missing entitlement → silent denial. Missing usage description → no prompt at all, just a denial.

## Entitlements file

`build/entitlements.mac.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- Required for Electron under hardened runtime -->
  <key>com.apple.security.cs.allow-jit</key><true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
  <key>com.apple.security.cs.disable-library-validation</key><true/>

  <!-- Add only what your app actually uses -->
  <key>com.apple.security.device.microphone</key><true/>
  <key>com.apple.security.device.camera</key><true/>
  <key>com.apple.security.device.audio-input</key><true/>

  <!-- Network -->
  <key>com.apple.security.network.client</key><true/>
  <key>com.apple.security.network.server</key><true/>

  <!-- Allow loading .env etc. in development - drop for App Store -->
  <key>com.apple.security.cs.allow-dyld-environment-variables</key><true/>
</dict>
</plist>
```

`allow-jit` is required (Chromium V8). `allow-unsigned-executable-memory` is required for Node native code in some configs. `disable-library-validation` is required when loading frameworks signed by a different team (e.g., third-party native modules).

## Usage descriptions (Info.plist)

electron-builder injects these via `extendInfo`:

```yaml
# electron-builder.yml
mac:
  extendInfo:
    NSMicrophoneUsageDescription: "MyApp records audio for voice notes."
    NSCameraUsageDescription: "MyApp captures video for tutorials."
    NSAudioCaptureUsageDescription: "MyApp captures system audio when recording."
    NSDesktopFolderUsageDescription: "MyApp saves files you ask it to save to Desktop."
    NSDocumentsFolderUsageDescription: "MyApp saves projects to Documents."
    NSDownloadsFolderUsageDescription: "MyApp saves exports to Downloads."
    NSAppleEventsUsageDescription: "MyApp uses Apple events to integrate with other apps."
```

Strings are user-facing — write them in plain English, mention what the app actually does. Apple rejects vague ones ("for app functionality") at App Store review.

## Querying status from main

```ts
import { systemPreferences } from 'electron'

type Media = 'microphone' | 'camera' | 'screen'
function status(m: Media) {
  return systemPreferences.getMediaAccessStatus(m)
  // 'not-determined' | 'granted' | 'denied' | 'restricted' | 'unknown'
}

async function ensureMic() {
  const s = status('microphone')
  if (s === 'granted') return true
  if (s === 'not-determined') return systemPreferences.askForMediaAccess('microphone')
  // 'denied' or 'restricted' — user must change in System Settings
  shell.openExternal('x-apple.systempreferences:com.apple.preference.security?Privacy_Microphone')
  return false
}
```

`askForMediaAccess` triggers the OS prompt the *first* time per app+permission. After that, the user's answer is sticky — re-prompting is impossible. Direct them to System Settings via the URL scheme.

## Permission flow per feature

| Feature | Entitlement | Info.plist key | Runtime |
|---|---|---|---|
| Microphone | `device.microphone`, `device.audio-input` | `NSMicrophoneUsageDescription` | `askForMediaAccess('microphone')` |
| Camera | `device.camera` | `NSCameraUsageDescription` | `askForMediaAccess('camera')` |
| Screen recording | (none — screen) | (system-managed) | `desktopCapturer.getSources()` triggers prompt |
| System audio | `device.audio-input` | `NSAudioCaptureUsageDescription` | via desktopCapturer |
| Notifications | (none) | (none) | `Notification` API → user prompt |
| Location | (none in entitlements) | `NSLocationUsageDescription` | navigator.geolocation |
| Contacts | `personal-information.addressbook` (MAS only) | `NSContactsUsageDescription` | rare in Electron |
| Accessibility | n/a — not entitled, OS managed | n/a | `systemPreferences.isTrustedAccessibilityClient(true)` |
| Full Disk Access | n/a | n/a | user grants in System Settings |

## Permission request handler

Even with entitlements, Electron's session handler can override:

```ts
const ALLOWED = new Set(['media', 'mediaKeySystem', 'notifications', 'clipboard-read'])

session.defaultSession.setPermissionRequestHandler((wc, permission, cb, details) => {
  if (!isAppOrigin(wc.getURL())) return cb(false)
  cb(ALLOWED.has(permission))
})
```

`media` is the umbrella — Electron does not split mic/camera/screen at this layer. To deny only screen capture, leave the request handler open and reject in `setDisplayMediaRequestHandler`.

## Accessibility (for global shortcuts, window manipulation)

Apps that read/write other apps' UI need Accessibility permission. Electron's `globalShortcut` works without it; tools like Rectangle (window mgmt) need it.

```ts
const trusted = systemPreferences.isTrustedAccessibilityClient(true)  // true prompts
if (!trusted) {
  // App is now in System Settings → Privacy → Accessibility
  // User must toggle, restart often required
}
```

## Hardened runtime gotchas

`hardenedRuntime: true` (electron-builder default for signed mac builds) blocks behaviors that older apps relied on:

- `dlopen` of unsigned libraries → need `disable-library-validation`
- JIT (V8) → need `allow-jit`
- W^X memory pages with arbitrary code → need `allow-unsigned-executable-memory` for some native modules
- Reading env vars at launch → need `allow-dyld-environment-variables` (or stop relying on it)

Symptom of missing entitlement: app immediately quits with crash report mentioning `Library Validation` or `Code signature invalid`. Console.app under "Crash Reports" has the details.

## Reading entitlements at runtime

You can't. Entitlements are baked into the signature. To know what's available, query the actual permission state via `systemPreferences` and OS APIs.
