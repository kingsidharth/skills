# Performance: Unblocking Processes

Two categories of slowness in Electron: slow startup and poor interaction performance.

## Unblocking the Main Process

The main process is the control tower — blocking it freezes the entire app (windows, GPU coordination, OS events).

**Critical rules:**
- Never use `fs.readFileSync`, `child_process.execSync`, or any sync I/O
- Never use `ipcRenderer.sendSync` — blocks the entire renderer until handled
- Use `ipcMain.handle` + `ipcRenderer.invoke` (async) instead
- For CPU-heavy tasks in main: use `worker_threads` or `utilityProcess`

```ts
import { utilityProcess } from 'electron';

// Spawn a separate Node.js process for heavy computation
const child = utilityProcess.fork(path.join(__dirname, 'heavy-task.js'));
child.postMessage({ type: 'start', data: payload });
child.on('message', (result) => { /* handle */ });
```

`utilityProcess` is preferred over hidden `BrowserWindow` for non-UI background work — lower overhead, no Chromium renderer.

## Unblocking the Renderer

The renderer is a browser tab. Same rules as web performance:
- Don't block the event loop with synchronous computation
- Use `requestIdleCallback()` for non-critical work
- Use `requestAnimationFrame()` for visual updates
- Use `startTransition` / `useTransition` (React) to prevent UI freezing during state updates

### Web Workers in Electron

Move heavy computation off the renderer's main thread:

```ts
// renderer
const worker = new Worker(new URL('./processor.worker.ts', import.meta.url));
worker.postMessage(largeDataset);
worker.onmessage = (e) => setResult(e.data);
```

**WebSocket + Worker pattern** (80% UI speed gain case study): move WebSocket connections to a Web Worker. All buffering happens in the worker's isolate; only processed results are posted to the main thread.

Trade-off: each Web Worker spawns a V8 isolate (~10–20 MB overhead). Acceptable for UI responsiveness gains, but monitor total memory.

### Direct Main↔Worker Communication

Normally: Worker → Renderer → Main (blocks renderer during relay). Better: create a `MessageChannel` in preload and pass one port to the worker, the other to main via IPC:

```ts
// preload.ts — create channel at startup
const { port1, port2 } = new MessageChannel();
ipcRenderer.postMessage('worker-port', null, [port2]);
// Pass port1 to web worker
```

This enables direct main↔worker data transfer, bypassing the renderer entirely. Critical for bulk data loading (markdown vaults, file imports).

## Startup Optimization

1. **Bundle everything** — replace `require()` with a bundler. `require()` is synchronous, recursive, and blocks both main and renderer threads. This is the single biggest startup bottleneck.

2. **Defer non-critical imports** — use dynamic `import()` and route-based code splitting:
   ```ts
   const Settings = lazy(() => import('./pages/Settings'));
   ```

3. **App shell architecture** — render a minimal shell immediately, hydrate features progressively

4. **Drop polyfills** — you own the browser (Chromium), target it directly

5. **Disable unused Chromium features** — via command-line switches in `app.commandLine`

6. **V8 Snapshots** — pre-initialize the V8 heap with your dependencies:
   - Use `electron-link` to create a snapshotable JS module
   - Use `mksnapshot` to create the snapshot blob
   - Atom reduced startup by 50% with this technique; VSCode uses it since 2017
   - Limitation: snapshot code must not contain `Date.now()`, `Math.random()`, or I/O calls

## Long-Running App Concerns

Desktop apps stay open for days/weeks. Memory leaks accumulate.

- Clean up event listeners in `useEffect` return functions
- Remove IPC listeners when windows close
- Close file handles, sockets, DB connections
- End child processes when no longer needed
- Pause/slow recurring tasks when window loses focus
- Use `requestIdleCallback()` for deferred background work
- Listen for system events: `powerMonitor.on('suspend')`, `powerMonitor.on('lock-screen')`
- Expire data caches; prune stale entries
- Run your app for a week during development and watch memory in Activity Monitor

> **Ref:** [Performance](https://www.electronjs.org/docs/latest/tutorial/performance) · [Palette Perf Guide](https://palette.dev/blog/improving-performance-of-electron-apps) · [Johnny Le Perf](https://johnnyle.io/read/electron-performance)
