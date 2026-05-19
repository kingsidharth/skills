# Process Isolation

## Context Isolation

Ensures preload scripts and Electron internals run in a separate JS context from web content. Prevents prototype pollution — `Array.prototype.push` in renderer can't affect preload.

```ts
new BrowserWindow({
  webPreferences: {
    contextIsolation: true,  // default since Electron 12
  },
});
```

The renderer's `window` and the preload's `window` are different objects. If you `window.hello = 'wave'` in preload, `window.hello` is `undefined` in the renderer.

Only expose what's needed via `contextBridge.exposeInMainWorld()`. Never expose raw `ipcRenderer`.

**TypeScript declarations** for context bridge — augment the `Window` interface:

```ts
// global.d.ts
export interface ElectronAPI {
  loadPreferences: () => Promise<Prefs>;
}
declare global {
  interface Window { electronAPI: ElectronAPI; }
}
```

## Process Sandboxing

Chromium's OS-level sandbox restricts what renderer processes can access. Same isolation as Chrome tabs.

```ts
new BrowserWindow({
  webPreferences: { sandbox: true },
});
```

Enable on **all** renderers. Loading or processing untrusted content in an unsandboxed process (including main) is not advised.

**Disabling `contextIsolation` also disables sandboxing**, regardless of the `sandbox` flag.

## Secure Defaults (Electron 20+)

```ts
new BrowserWindow({
  webPreferences: {
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: true,
    webSecurity: true,
    allowRunningInsecureContent: false,
  },
});
```

If you find yourself changing these, interrogate the reason aggressively.

> **Ref:** [Context Isolation](https://www.electronjs.org/docs/latest/tutorial/context-isolation) · [Process Sandboxing](https://www.electronjs.org/docs/latest/tutorial/sandbox) · [Security](https://www.electronjs.org/docs/latest/tutorial/security)
