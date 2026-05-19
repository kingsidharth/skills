# Architecture

## Three process types

**Main** — single Node.js process. Owns `app`, `BrowserWindow`, `Menu`, `Tray`, `dialog`, `session`, the filesystem, and every native module. Entry point declared by `"main"` in `package.json`.

**Renderer** — one Chromium process per `BrowserWindow` (plus one per `<webview>` or `WebContentsView`). Runs your web UI. No `require`, no `process`, no `fs` — by default and by design.

**Preload** — a script injected into a renderer before page load. Runs in the renderer's process but in an isolated world with access to a limited set of Node and Electron APIs (`contextBridge`, `ipcRenderer`). The only sanctioned bridge between main and renderer.

Additionally, `utilityProcess.fork()` spawns a standalone Node process for CPU-heavy work — no DOM, no Chromium overhead, faster to start than a hidden BrowserWindow.

## Lifecycle

```
app.whenReady()
  → create main BrowserWindow
  → attach preload
  → load renderer URL

app.on('window-all-closed')
  → on macOS: DO NOT quit (user expects dock-based lifecycle)
  → on other platforms: app.quit()

app.on('activate')
  → on macOS: if no windows, create one (clicking dock icon)
```

macOS convention: the app keeps running with no windows open. Re-create a window on `activate`. Only `app.quit()` when the user explicitly chooses Quit.

## When to use which process

Use the **main process** for: filesystem I/O that touches user data, native modules, menus, dialogs, tray, auto-update, OS integrations, anything that must survive window closure.

Use the **renderer** for: all UI, animations, layout, DOM, local state, Web APIs (Web Audio, `fetch`, IndexedDB). Treat it as a normal web app.

Use **preload** only to expose a thin, typed API surface. Every function exposed is an attack surface — keep it minimal.

Use **UtilityProcess** for: heavy JSON parsing, large file hashing, ML inference via native bindings, transcoding. Anything that would block the main loop for more than a few ms.

## Window architecture patterns

Single-window apps keep state in the renderer and use IPC only for native capabilities.

Multi-window apps (e.g., a main window + settings window) should keep authoritative state in the main process and sync to each renderer via IPC. Otherwise the windows disagree.

Long-running background behavior (polling, sync loops) belongs in the main process or a UtilityProcess, not a hidden BrowserWindow — a BrowserWindow pulls in a full Chromium renderer and hundreds of MB.

## Loading the renderer

`win.loadURL('http://localhost:3000')` in dev, `win.loadFile(path.join(__dirname, '../renderer/index.html'))` in production for static-exported Next.js. For a standalone Next.js server, `loadURL('http://localhost:${port}')` with the port negotiated at startup.

A custom `app://` protocol via `protocol.handle()` is cleaner than `file://` for static exports — it avoids CORS oddities and gives you a stable origin for cookies/storage.

## Where code runs — common pitfalls

- `process.platform` is available in all three contexts, but `process.versions.electron` is only set where Electron APIs are.
- `__dirname` in the built main process points at your output directory, not source. Resolve paths from `app.getAppPath()`.
- `require('electron')` from the renderer throws under context isolation. Always go through preload.
- The preload runs *before* the page, so it can't read the DOM. Use `window.addEventListener('DOMContentLoaded', ...)` if you need DOM access from preload.
