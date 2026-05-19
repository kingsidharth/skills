# Security

## webPreferences baseline

Every window:

```ts
new BrowserWindow({
  webPreferences: {
    preload: path.join(__dirname, '../preload/index.js'),
    contextIsolation: true,   // default since 12
    nodeIntegration: false,   // default since 5
    sandbox: true,            // default since 20 (if contextIsolation true)
    webSecurity: true,        // default true
    allowRunningInsecureContent: false,
  },
})
```

All defaults are secure. The failure mode is explicitly flipping one off because some library's install guide says to — don't.

## Context isolation

With isolation, the preload's `window` is a different object from the renderer's `window`. Assigning `window.foo = …` in preload silently fails to expose anything. Use `contextBridge.exposeInMainWorld('foo', …)` instead.

The bridge copies data through V8's structured clone. You cannot pass functions with closures, class instances with private fields, or objects with custom prototypes. Pass plain data, and for callbacks, the bridge wraps them specially.

## Sandboxed preload

A sandboxed preload runs with the OS sandbox enabled. It *cannot* use most Node built-ins (`fs`, `child_process`). It *can* use `ipcRenderer`, `contextBridge`, and a handful of Electron-specific APIs.

If a preload needs Node — for example, to import a native module or read a bundled config — it has to be non-sandboxed, which means `sandbox: false` on the window. Prefer keeping the preload sandboxed and doing the Node work in the main process behind an IPC handler.

## Content Security Policy

Set a CSP. Without one, a rogue dependency or XSS can exfiltrate data.

```ts
session.defaultSession.webRequest.onHeadersReceived((details, cb) => {
  cb({
    responseHeaders: {
      ...details.responseHeaders,
      'Content-Security-Policy': [
        "default-src 'self' app:; " +
        "script-src 'self' 'wasm-unsafe-eval'; " +
        "style-src 'self' 'unsafe-inline'; " +
        "connect-src 'self' https://api.yourservice.com; " +
        "img-src 'self' data: blob:;",
      ],
    },
  })
})
```

`'unsafe-inline'` for styles is nearly unavoidable with CSS-in-JS and Tailwind at dev time. Tighten for production builds if possible.

## Permission handler

Electron auto-approves requests like notifications and media by default. Replace the handler:

```ts
session.defaultSession.setPermissionRequestHandler((_wc, permission, cb) => {
  const allow = ['media', 'notifications', 'clipboard-read'].includes(permission)
  cb(allow)
})
```

`media` is the composite permission covering microphone, camera, and screen capture — Electron groups all three, unlike Chrome. If you need only mic, there's no finer-grained Electron permission, but you can check actual device use via `systemPreferences.getMediaAccessStatus()`.

## Navigation and windowOpen

Lock down navigation:

```ts
app.on('web-contents-created', (_e, contents) => {
  contents.on('will-navigate', (event, url) => {
    if (new URL(url).origin !== 'http://localhost:3000') event.preventDefault()
  })
  contents.setWindowOpenHandler(({ url }) => {
    shell.openExternal(url)
    return { action: 'deny' }
  })
})
```

External links go through `shell.openExternal` to the system browser, not a new renderer.

## Never

- Load untrusted remote content in a BrowserWindow. Use an `<iframe>` with `sandbox` attribute or, better, a separate browser.
- Set `webSecurity: false` — disables same-origin policy for the entire window.
- Use `remote` module — removed in recent Electron, but if you see it referenced anywhere, migrate to IPC.
- Pass `ipcRenderer` directly through `contextBridge` — that defeats the point.
