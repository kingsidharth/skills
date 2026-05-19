---
name: electron-apps
description: Electron desktop app development across macOS and Windows. Including security, performance, using code beyond Javascript & Typescript, native APIs, storage, debugging and troubleshooting issues, storage, and electron best practices. Use when working with electron code, troubleshooting, testing, preparing for release.
---

# Electron Apps

Build production-grade desktop applications with Electron. TypeScript throughout, security-first, performance-aware.

## Quick Start

```bash
npx create-electron-app@latest my-app --template=vite-typescript
cd my-app && bun install && npm run start
```

For project layout, IPC contracts, and secure window creation, see [Project Setup](references/project-setup/structure.md).

## Essential Patterns

### Secure BrowserWindow

Every window starts here. These are Electron 20+ defaults — never weaken them without a specific reason:

```ts
const win = new BrowserWindow({
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    contextIsolation: true,    // preload ≠ renderer context
    nodeIntegration: false,    // no require() in renderer
    sandbox: true,             // Chromium OS-level sandbox
  },
  show: false,                 // prevent white flash
});
win.once('ready-to-show', () => win.show());
```

### Module Loading Strategy

`require()` is synchronous and recursive — the single biggest startup bottleneck. Bundle everything; load progressively.

```ts
// ❌ Eager — blocks main thread on startup
import heavyModule from 'heavy-module';

// ✅ Deferred — load when actually needed
const getHeavy = () => import('heavy-module');

// ✅ Route-level code splitting in renderer
const Settings = lazy(() => import('./pages/Settings'));
```

One bundle per process: `main.bundle.js`, `preload.bundle.js`, `renderer.bundle.js` (with chunk splitting). Details in [Bundling](references/performance/bundling.md).

### Unblocking the Main Process

The main process is the control tower. Blocking it freezes every window.

```ts
// ❌ Sync I/O
const data = fs.readFileSync('large.json', 'utf-8');

// ✅ Async I/O
const data = await fs.promises.readFile('large.json', 'utf-8');

// ❌ Sync IPC (blocks entire renderer until main responds)
const result = ipcRenderer.sendSync('query', args);

// ✅ Async IPC
const result = await ipcRenderer.invoke('query', args);

// ✅ Heavy computation → separate process
const child = utilityProcess.fork(path.join(__dirname, 'worker.js'));
child.postMessage(payload);
```

See [Unblocking](references/performance/unblocking.md) for Web Workers, Worker↔Main direct channels, and `utilityProcess`.

### Cache-First Rendering

Desktop apps should feel instant on repeat launches. Render from local cache, refresh in background:

```ts
// TanStack Query + IndexedDB persister
const queryClient = new QueryClient({
  defaultOptions: { queries: { gcTime: 1000 * 60 * 60 * 24 } },
});

// Wrap app with PersistQueryClientProvider
// → First paint from cache (0-100ms), network refresh in background
```

For optimistic updates, prefetching, and pre-warmed startup (Linear-style), see [Cache-First](references/performance/cache-first.md).

### IPC Contract

Define channel names once, share between main and preload:

```ts
// src/shared/ipc-channels.ts
export const IPC = {
  GET_VERSION: 'get-version',
  SAVE_FILE: 'save-file',
} as const;
```

For the full invoke/handle pattern, listener cleanup, and MessagePort for high-throughput, see [IPC Patterns](references/core/ipc-patterns.md).

---

## Reference Map

### Core Concepts
- [Process Model](references/core/process-model.md) — main process, renderer, preload scripts, TypeScript declarations
- [IPC Patterns](references/core/ipc-patterns.md) — invoke/handle, one-way, main→renderer, renderer↔renderer, MessagePort, listener cleanup

### Security
- [Process Isolation](references/security/process-isolation.md) — context isolation, sandboxing, secure defaults
- [Content Security Policy](references/security/csp.md) — meta tag setup, bundler compatibility, common gotchas
- [IPC Security & Safe Storage](references/security/ipc-and-storage.md) — input validation, navigation hardening, permissions, safeStorage, file:// avoidance, auditing checklist

### Performance
- [Unblocking Processes](references/performance/unblocking.md) — main process rules, renderer optimization, Web Workers, Worker↔Main direct channels, startup optimization, V8 snapshots, long-running app concerns
- [Bundling & Code Splitting](references/performance/bundling.md) — require() problem, one-bundle-per-process, code splitting, tree shaking, Bun (pkg mgr vs bundler), CSP implications
- [Native Code & WASM](references/performance/native-code.md) — WebAssembly, NAPI-RS (Rust 10x case study), Go bindings, Python sidecars, comparison table
- [Cache-First & Perceived Performance](references/performance/cache-first.md) — pre-warmed startup, IndexedDB+TanStack persistence, optimistic updates, prefetch, idle-time work
- [Instrumentation & Profiling](references/performance/instrumentation.md) — Chrome DevTools, contentTracing API, CPU instruction counting, production monitoring (Slack/VSCode patterns), component-level CPU costs, React profiling tools

### Windows & UI
- [Window Management](references/windows/window-management.md) — custom title bars, traffic lights, frameless/transparent windows, drag regions, vibrancy, progress bars, multi-window patterns, navigation history, prevent-close dialogs

### Patterns & Recipes
- [Recipes](references/patterns/recipes.md) — keyboard shortcuts (local/global/window), deep links (custom protocol), notifications, spellchecker, device access (Bluetooth/HID/Serial), offscreen rendering, multithreading options, background server pattern, live reloading, Windows taskbar, environment variables

### Debugging
- [Debugging](references/debugging/debugging.md) — main process inspector, renderer DevTools, REPL, DevTools extensions, native addon debugging (Xcode/lldb), automated testing with Playwright, common debug techniques

### Native & Platform
- [Platform Integration](references/native/platform-integration.md) — device access (screen, camera, mic, clipboard), power monitoring, startup registration, storage paradigms, network interception, safe key storage, macOS (Swift sidecars, permissions, dock), Windows (taskbar, jump lists), native UI (menus, tray, notifications, theme)

### Electron Forge
- [Config & Plugins](references/forge/config-and-plugins.md) — build lifecycle, TypeScript config, packagerConfig, rebuildConfig, plugin system (Vite/Fuses), hooks, build identifiers, CLI commands
- [Makers & Distribution](references/forge/makers-and-distribution.md) — all makers (DMG/Squirrel/deb/AppX/WiX), publishers (GitHub/S3), code signing (macOS/Windows), notarization, auto-updates, CI/CD, custom makers/publishers

### Project Setup
- [Structure & Setup](references/project-setup/structure.md) — recommended directory layout, Forge+Vite+TS quick start, Bun integration, IPC channel contracts, tsconfig, essential dependencies, secure BrowserWindow factory

---

## Decision Quick-Reference

| Need | Solution |
|------|----------|
| Background task (non-UI) | `utilityProcess` |
| Background task (renderer-bound) | Web Worker |
| Heavy computation | NAPI-RS (Rust) or WASM |
| Data persistence | IndexedDB (cache), SQLite (structured), `safeStorage` (secrets), `electron-store` (config) |
| IPC pattern | `invoke`/`handle` (default), `send`/`on` (fire-forget), `MessagePort` (high-throughput) |
| Custom title bar | `titleBarStyle: 'hidden'` + HTML/CSS with `app-region: drag` |
| System-wide shortcut | `globalShortcut.register()` |
| Deep link | `app.setAsDefaultProtocolClient()` |
| Testing | Playwright (E2E), Jest (unit), `contentTracing` (perf regression) |
