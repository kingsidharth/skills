# Performance

## Startup time budget

Cold start: target <2s to first useful paint. Anything beyond 3s feels broken.

Cold-start phases (typical):
- 200-400 ms — Electron/Chromium init, can't reduce
- 50-300 ms — main process initialization (your code)
- 100-400 ms — first BrowserWindow creation
- 100-1500 ms — renderer load, parse, paint

Reduce what you control: main-process init and renderer first paint.

## Main process startup

Top of `main/index.ts` should be the bare minimum. Move feature code behind `app.whenReady()`, and even within ready, defer non-critical work:

```ts
import { app, BrowserWindow } from 'electron'
import path from 'node:path'

app.whenReady().then(async () => {
  const win = await createMainWindow()  // critical path
  win.once('ready-to-show', () => {
    win.show()  // user sees something
    requestIdle(() => import('./updater').then(m => m.start()))
    requestIdle(() => import('./analytics').then(m => m.init()))
  })
})

function requestIdle(fn: () => void) {
  setTimeout(fn, 0)  // or use process.nextTick / an idle scheduler
}
```

`ready-to-show` fires when the renderer has rendered enough to display without flash. Always wait for it before `win.show()`:

```ts
const win = new BrowserWindow({ show: false /* … */ })
win.once('ready-to-show', () => win.show())
win.loadURL(/* … */)
```

Without this, users see white flash → content. With it, the window appears already painted.

## Renderer first paint

Static export with a `app://` protocol gives the fastest cold start — no server, just file reads. Standalone Next.js server adds ~300-700 ms for `next start` to begin responding.

For App Router static export, ensure your `app/page.tsx` doesn't dynamic-import heavy components. `loading.tsx` skeletons help perceived speed.

```tsx
// Bad: blocks first paint
import HeavyChart from './HeavyChart'
export default function Page() { return <HeavyChart /> }

// Good: stream behind a skeleton
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
})
```

## Main thread (renderer)

The single biggest cause of jank: long tasks on the main thread. Anything >50 ms blocks input.

- Long lists → virtualize
- Heavy computation → Web Worker (yes, in Electron renderer)
- Big JSON.parse → stream parse, or do it in main process and stream chunks back
- Synchronous IPC → never; always `invoke` (async)

```ts
const worker = new Worker(new URL('./hash.worker.ts', import.meta.url), { type: 'module' })
worker.postMessage(data)
worker.onmessage = e => setResult(e.data)
```

Webpack/Next.js handles `new Worker(new URL(...))` in App Router. For Tauri-style hot paths, prefer a native module via `UtilityProcess` over Worker — no V8 isolate startup, direct memory access.

## UtilityProcess for CPU work

```ts
import { utilityProcess } from 'electron'
const child = utilityProcess.fork(path.join(__dirname, 'workers/hash.js'))
child.postMessage({ file })
child.on('message', msg => { /* result */ })
```

Lighter than a hidden BrowserWindow (no Chromium), heavier than a Worker (separate Node process). Right tool when:
- Renderer Web Workers can't do it (no Node APIs)
- You don't want to block the main process
- Crash isolation matters

## Animation perf

Use CSS transforms and opacity. Avoid animating `width`, `height`, `top`, `left`, or `box-shadow` — they trigger layout/paint. `will-change: transform` to promote to GPU layer when needed; remove after animation to free GPU memory.

For 60+ fps, your frame budget is 16.6 ms. JS work + style + layout + paint must fit. Profile in DevTools' Performance tab.

## Lazy-load routes

In Next.js App Router, route segments are code-split automatically. Don't fight it — don't `import` route components from each other.

For features behind feature flags or premium gates, dynamic-import on activation:

```ts
async function openProEditor() {
  const { ProEditor } = await import('./pro/ProEditor')
  setEditor(<ProEditor />)
}
```

## Image performance

- Pre-resize at build, ship multiple sizes, use `<picture>` or `srcset`.
- WebP / AVIF over PNG/JPG.
- Lazy-load offscreen images: `<img loading="lazy">`.
- Don't decode images on the main thread for large galleries — `createImageBitmap` in a worker, then transfer.

## Caching network responses

Use the renderer's HTTP cache. For app-level data caching, IndexedDB or `caches` API. Don't roll your own JSON cache — the browser already does it.

For Service Workers in Electron: supported on `app://` (registered as `secure: true, standard: true, supportFetchAPI: true`), not on `file://`. Reason enough to use a custom protocol.

## Profiling production

Add an opt-in switch:

```ts
if (process.env.PROFILE) {
  app.commandLine.appendSwitch('--cpu-prof')
  app.commandLine.appendSwitch('--cpu-prof-dir', logsDir)
}
```

`.cpuprofile` files load in Chrome DevTools (Performance → Load profile) or VS Code (`vscode-js-profile-flame`).

For renderer, ship `--enable-blink-features=DocumentTransitionAPI` etc. when needed; otherwise leave Chromium defaults.

## Perceived perf tricks

- Optimistic UI updates — render the new state before the IPC round-trip finishes; reconcile on response.
- Skeleton screens > spinners > nothing. Spinners signal "I'm working"; skeletons signal "almost ready".
- Pre-warm everything you can: prefetch the next likely route, decode the next image.
- Show real numbers, not deceptively round ones. Users notice "100%" stuck on a fake bar.
