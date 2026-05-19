# Memory optimization

Electron's baseline footprint is ~150 MB. Below that is hard. Goal: don't grow over time, don't grow per window, don't pin large buffers.

## Leak patterns

**Event listeners not cleaned up** — the most common leak. Every IPC subscription, DOM listener, timer, and observer must have a matching cleanup.

```ts
// React: always return cleanup
useEffect(() => {
  const off = window.api.on.update(handle)
  return off
}, [])

// Vanilla: store the disposer
const stop = setInterval(tick, 1000)
window.addEventListener('unload', () => clearInterval(stop))
```

`removeAllListeners()` is a code smell — it suggests you don't track listener ownership. Track each one.

**IPC handlers registered per window** — `ipcMain.handle('foo', ...)` is global. Registering it inside `createWindow()` makes the second window throw "Attempted to register a second handler". Register once at startup.

**Closures over large data** — a single `onMessage` callback that captures a huge object keeps that object alive forever. Move state out of the closure.

**Hidden BrowserWindows used as workers** — they keep a full Chromium renderer alive (~50 MB+). Replace with `UtilityProcess` or `Worker` thread.

**Native module handles** — DB connections, file watchers, child processes. Close them in `app.on('before-quit')` and on per-feature teardown.

## V8 heap

Default V8 heap limit is ~4 GB (with pointer compression). Apps that consistently hit it should stream rather than load whole. If you genuinely need more heap (rare for desktop apps), `--max-old-space-size` via `app.commandLine.appendSwitch('js-flags', '--max-old-space-size=8192')`.

Pointer compression saves ~40% heap and improves GC; it's enabled by default in Electron 14+.

## Window pooling vs creating

Creating a BrowserWindow takes 100-300 ms and 50+ MB. For frequently-opened auxiliary windows (settings, quick-view), keep them around hidden:

```ts
let settingsWin: BrowserWindow | null = null

function showSettings() {
  if (settingsWin && !settingsWin.isDestroyed()) {
    settingsWin.show()
    return
  }
  settingsWin = new BrowserWindow({ show: false /* ... */ })
  settingsWin.on('close', e => {
    e.preventDefault()
    settingsWin?.hide()  // hide instead of destroy
  })
  settingsWin.loadURL(/* ... */)
  settingsWin.show()
}
```

Trade-off: a hidden window still keeps its renderer alive. Only pool if open-frequency justifies it.

## Pause work in background windows

When the main window loses focus or is occluded, throttle:

```ts
win.on('blur', () => win.webContents.send('paused'))
win.on('focus', () => win.webContents.send('resumed'))

// renderer
window.api.on.paused(() => clearInterval(pollHandle))
```

For animations, `requestAnimationFrame` already throttles to ~1 fps when occluded. For your own polling loops, listen to `visibilitychange` or to the IPC signal.

## Defer initialization

Loading every dependency at startup adds 50-200 ms to first paint. Defer:

```ts
// Bad: import everything at top of main
import './updater'
import './tray'
import './metrics'

// Better: dynamic import after first window appears
app.whenReady().then(async () => {
  await createMainWindow()
  // First window is visible — now load the rest
  void import('./updater').then(m => m.start())
  void import('./tray').then(m => m.create())
  setTimeout(() => import('./metrics').then(m => m.init()), 5000)
})
```

Splash screens are unnecessary if you can show a window in <500 ms. Splash always feels slower than the same wait without one.

## Reduce module weight

Profile with `--cpu-prof`. Look for surprisingly heavy `require` chains. Common offenders:

- `moment` → use `date-fns` or `Intl.DateTimeFormat`
- `lodash` → cherry-pick (`lodash/debounce`) or use native
- `aws-sdk` v2 → v3 with per-service imports
- `ffi` for one C call → native binding via NAPI-RS

Anything over 200 KB compressed is suspicious. Tree-shake at build time. Check renderer bundle with `next-bundle-analyzer`.

## Renderer DOM and lists

Don't render 10k DOM nodes. Virtualize with `react-window` or `@tanstack/react-virtual`. The latter is more flexible and well-maintained.

```ts
const rowVirtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 48,
  overscan: 5,
})
```

Even 500 visible rows of complex JSX (rich text, icons, callbacks per row) drag scroll. Profile with React DevTools' Profiler tab.

## Image and media memory

A 4K image as PNG decoded is ~33 MB in canvas memory. Resize before display. Use `srcset` or thumbnail variants for grids. Revoke `URL.createObjectURL` blobs after use.

## Diagnostic tools

- DevTools → Memory tab → Heap snapshot. Compare before/after a suspected leak.
- DevTools → Performance → record. Spot main-thread blocks.
- Main process: `process.memoryUsage()` and `process.getHeapStatistics()` — snapshot periodically, log to file, watch for growth.
- `chrome://process-internals` (open in your dev window) shows per-process memory live.

## App went idle, memory still high

Chromium doesn't aggressively reclaim. To force shrink: `app.releaseMemory()` (where available) or trigger GC in the renderer with `--js-flags=--expose-gc` and `globalThis.gc()` after big releases. Generally not worth it — let GC run naturally.
