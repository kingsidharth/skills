---
name: electron-mac
description: Build Electron apps for macOS with Next.js/React/TypeScript/Bun. Covers project setup, main/renderer/preload architecture, IPC, macOS window styling, vibrancy sidebar, traffic lights, native file system, dialogs, audio capture, desktopCapturer, storage (electron-store, SQLite), native modules (N-API, node-gyp, @electron/rebuild), memory optimization (V8 cage, leaks, window pooling), performance (startup, main-thread, virtualization), permissions and entitlements, notifications, dock/tray/menus, deep links, auto-update (electron-updater, Squirrel.Mac), electron-builder packaging, DMG, code signing, notarytool notarization, universal arm64+x64 builds, Mac App Store, context isolation and contextBridge security. Use when building, debugging, packaging, or distributing any Electron app targeting macOS.
---

# Electron for macOS

Current stack: Electron 40 (Chromium M144, Node 22), Bun as package manager, TypeScript, Next.js App Router with static export, `src/` layout, `electron-builder` for packaging.

## Routing

| Topic | File |
|---|---|
| Project scaffold, Bun + TS + Next.js layout, dev/build scripts | `references/project-setup.md` |
| Main/renderer/preload model, process lifecycle, UtilityProcess | `references/architecture.md` |
| IPC patterns (`invoke`/`handle`, `send`/`on`, `MessagePort`) | `references/ipc.md` |
| contextBridge, context isolation, sandbox, CSP, webPreferences | `references/security.md` |
| Next.js App Router integration, static export vs standalone, dev port handshake | `references/nextjs-integration.md` |
| Bun/TypeScript/tsup config for main+preload, Tailwind in renderer | `references/bun-typescript.md` |
| macOS window: `titleBarStyle`, `trafficLightPosition`, vibrancy, `visualEffectState`, frameless, rounded corners | `references/window-macos.md` |
| Sidebar layouts, `-webkit-app-region: drag`, title-bar overlay, responsive shell | `references/sidebar-layout.md` |
| Native menus, Dock, Tray, system appearance (dark/light) | `references/menus-dock-tray.md` |
| File system access, `dialog`, drag-and-drop, file:// protocol, custom protocols | `references/file-system.md` |
| Audio capture (`getUserMedia`, `desktopCapturer`, CoreAudio Tap), permissions | `references/audio.md` |
| Storage: `electron-store`, `better-sqlite3`, `app.getPath('userData')`, migration | `references/storage.md` |
| Native modules: N-API, `node-gyp`, `@electron/rebuild`, V8 memory cage, prebuilds | `references/native-modules.md` |
| Memory: leak patterns, window pooling, V8 heap, background window pausing | `references/memory-optimization.md` |
| Startup, main-thread, renderer rendering, virtualization, `requestIdleCallback` | `references/performance.md` |
| macOS entitlements, hardened runtime, `systemPreferences.askForMediaAccess` | `references/permissions-entitlements.md` |
| Deep links (custom URL schemes), single-instance lock, file-open events | `references/protocols-deep-links.md` |
| `electron-updater`, Squirrel.Mac, feed URL, GitHub/S3/generic providers | `references/auto-update.md` |
| `electron-builder` mac config, DMG, ZIP, universal builds, ASAR, `asarUnpack` | `references/packaging.md` |
| Code signing, `notarytool`, `@electron/notarize`, keychain profiles, afterSign | `references/code-signing-notarization.md` |
| Mac App Store submission, sandboxing entitlements, MAS-specific builds | `references/mac-app-store.md` |

## Quick-start defaults

Start from Electron Forge's Next.js + TypeScript template only if the renderer is trivial. For a Next.js App Router renderer with Bun, scaffold manually per `project-setup.md`. Keep `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` unless a specific preload pattern requires otherwise — then narrow the exception to a single window.

Default macOS window: `titleBarStyle: 'hiddenInset'`, `trafficLightPosition: { x: 16, y: 16 }`, `vibrancy: 'sidebar'`, `visualEffectState: 'active'`, `frame: false` only if rebuilding controls yourself.

Use `@electron/rebuild` after every install that touches native deps. Run via `postinstall` so CI never misses it.

Always sign and notarize macOS builds before distribution — unsigned DMGs are blocked by Gatekeeper regardless of how the user acquired them.

## Cross-cutting rules

Never load remote content in a window with `nodeIntegration: true`. Never disable `webSecurity`. Every preload must use `contextBridge.exposeInMainWorld` — direct `window.x = …` assignment is silently dropped under context isolation.

Treat the main process as a hot path. Anything blocking in the main process freezes every window. Offload CPU work to `UtilityProcess`, worker threads in the renderer, or a native module.

For renderer state, prefer in-memory + IPC round-trips to the main process over storing in renderer-accessible files. Writes to the config file race if multiple renderers do it.
