# Protocols and deep links

## Single-instance lock

By default, launching the app twice spawns two processes. For deep-link handling and most apps, you want one process — second launch focuses the existing window.

```ts
const gotLock = app.requestSingleInstanceLock()
if (!gotLock) {
  app.quit()
} else {
  app.on('second-instance', (_e, argv, _cwd) => {
    // Existing instance handles the launch
    if (mainWindow) {
      if (mainWindow.isMinimized()) mainWindow.restore()
      mainWindow.focus()
    }
    // argv contains the deep link if launched via URL on Windows/Linux
    handleDeepLinkArgs(argv)
  })
}
```

On macOS, deep-link URLs come through `open-url`, not `argv`. Handle both.

## Custom URL scheme (deep links)

Register your scheme in main BEFORE `app.whenReady()`:

```ts
if (process.defaultApp) {
  if (process.argv.length >= 2) {
    app.setAsDefaultProtocolClient('myapp', process.execPath, [path.resolve(process.argv[1])])
  }
} else {
  app.setAsDefaultProtocolClient('myapp')
}

// macOS deep-link entry point
app.on('open-url', (event, url) => {
  event.preventDefault()
  handleDeepLink(url)
})
```

`process.defaultApp` is true when running via `electron .` (dev). The dev branch tells macOS where to send the URL during development.

For electron-builder to register the scheme in the bundle, declare it:

```yaml
mac:
  extendInfo:
    CFBundleURLTypes:
      - CFBundleURLName: MyApp
        CFBundleURLSchemes: ["myapp"]
```

Test: `open myapp://login?token=abc` from Terminal — your app should receive `'myapp://login?token=abc'` in `open-url`.

## Universal Links (https://)

Custom schemes are insecure (anyone can register `myapp://`). For production OAuth, use Universal Links — your app handles `https://yourdomain.com/auth/...` URLs alongside Safari, claimed via `apple-app-site-association` JSON hosted at your domain.

Setup:
1. Host `https://yourdomain.com/.well-known/apple-app-site-association` with the right JSON for your team ID + bundle ID.
2. Add `com.apple.developer.associated-domains` entitlement: `applinks:yourdomain.com`.
3. Handle in app via `open-url` (same as custom scheme).

Universal Links require the app to be installed via signed installer or App Store; they don't work for ad-hoc Electron in dev. For OAuth in dev, fall back to a custom scheme or a localhost loopback.

## Custom protocols for app content

Different from URL schemes. `protocol.handle()` lets the renderer fetch your-scheme://... like a normal HTTP origin, served by your handler.

Use cases:
- Static-export Next.js loaded via `app://` (better than `file://`)
- Serving user-uploaded files from `userData/blobs/` to the renderer
- Serving generated PDFs/images on-demand

```ts
import { app, protocol, net } from 'electron'

// Must be called before app.whenReady() resolves
protocol.registerSchemesAsPrivileged([
  { scheme: 'app',  privileges: { standard: true, secure: true, supportFetchAPI: true, corsEnabled: true } },
  { scheme: 'user', privileges: { standard: true, secure: true, supportFetchAPI: true, stream: true } },
])

app.whenReady().then(() => {
  protocol.handle('app', async (req) => {
    const url = new URL(req.url)
    const file = url.pathname === '/' ? '/index.html' : url.pathname
    const abs = path.join(app.getAppPath(), 'renderer/out', file)
    return net.fetch(pathToFileURL(abs).toString())
  })

  protocol.handle('user', async (req) => {
    const id = new URL(req.url).pathname.slice(1)
    const abs = path.join(USER_DATA, 'blobs', id)
    if (!abs.startsWith(USER_DATA)) return new Response('forbidden', { status: 403 })
    return net.fetch(pathToFileURL(abs).toString())
  })
})
```

`standard: true` — schema acts like http (origins, cookies, navigation). Without it, many web APIs (Service Workers, IndexedDB) refuse to work.

`secure: true` — counts as HTTPS for mixed-content rules.

`stream: true` — necessary for video/audio with seek; otherwise range requests fail.

## File-open events (macOS)

When the user double-clicks a file associated with your app, or drags a file onto your dock icon:

```ts
app.on('open-file', (event, filePath) => {
  event.preventDefault()
  if (mainWindow) {
    mainWindow.webContents.send('open-file', filePath)
  } else {
    pendingOpen = filePath
  }
})
```

Fires before `ready` if the app launched in response to the open. Buffer the path until `ready` if needed.

Declare associated file types in electron-builder:

```yaml
mac:
  extendInfo:
    CFBundleDocumentTypes:
      - CFBundleTypeName: MyApp Document
        CFBundleTypeRole: Editor
        LSItemContentTypes: [com.example.myapp.document]
        LSHandlerRank: Owner
    UTExportedTypeDeclarations:
      - UTTypeIdentifier: com.example.myapp.document
        UTTypeDescription: MyApp Document
        UTTypeConformsTo: [public.data]
        UTTypeTagSpecification:
          public.filename-extension: [myapp]
```

## Drag-and-drop external

Renderer drop handling — see `file-system.md`. From main side, `webContents.startDrag` initiates a drag *out* to other apps.
