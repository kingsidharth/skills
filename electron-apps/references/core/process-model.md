# Process Model

Electron inherits Chromium's multi-process architecture. Every app has exactly one **main process** and one or more **renderer processes**, bridged by **preload scripts**.

## Main Process

- Entry point of the app (Node.js environment)
- Full access to Node.js APIs, OS, file system, native modules
- Creates and manages `BrowserWindow` instances
- Controls app lifecycle via `app` module
- Handles IPC from renderers, system tray, menus, dialogs
- **Never block this process** — it's the control tower for windows, GPU coordination, and OS events

## Renderer Process

- Each `BrowserWindow` spawns its own renderer
- Runs web content (HTML/CSS/JS) in a Chromium context
- **No direct Node.js access** (when properly configured)
- Destroyed when its parent `BrowserWindow` closes
- Same security model as a browser tab

## Preload Scripts

- Execute in the renderer context **before** web content loads
- Have access to a limited Node.js API subset
- Primary role: expose a curated API surface to the renderer via `contextBridge`
- Attached via `webPreferences.preload` in `BrowserWindow` constructor

```ts
const win = new BrowserWindow({
  webPreferences: {
    preload: path.join(__dirname, 'preload.ts'),
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: true,
  },
});
```

## TypeScript Declarations for Context Bridge

When exposing APIs via `contextBridge`, augment the `Window` interface:

```ts
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('api', {
  getVersion: () => ipcRenderer.invoke('get-version'),
  onMenuAction: (cb: (action: string) => void) =>
    ipcRenderer.on('menu-action', (_e, action) => cb(action)),
});

// global.d.ts
export interface ElectronAPI {
  getVersion: () => Promise<string>;
  onMenuAction: (cb: (action: string) => void) => void;
}

declare global {
  interface Window {
    api: ElectronAPI;
  }
}
```

## Process Type Imports

```ts
import { app, BrowserWindow } from 'electron';          // main process
import { contextBridge, ipcRenderer } from 'electron';   // preload
// electron/main, electron/renderer, electron/common for types
```

> **Ref:** [Process Model](https://www.electronjs.org/docs/latest/tutorial/process-model) · [Using Preload Scripts](https://www.electronjs.org/docs/latest/tutorial/tutorial-preload)
