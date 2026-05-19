# IPC

## Patterns

Three patterns, pick by direction and response shape:

| Direction | Response? | API |
|---|---|---|
| Renderer → Main | Yes (awaited) | `ipcRenderer.invoke` / `ipcMain.handle` |
| Renderer → Main | No | `ipcRenderer.send` / `ipcMain.on` |
| Main → Renderer | No | `webContents.send` / `ipcRenderer.on` |
| Two-way stream | Yes | `MessageChannelMain` / `MessagePort` |

`invoke`/`handle` is the default. It's promise-based, has automatic error propagation, and matches how most renderer code wants to call out.

Use `send`/`on` for fire-and-forget events (analytics, logging).

Use `MessagePort` for high-throughput streams (telemetry, incremental data). It bypasses serialization of the IPC channel name per message and supports structured clone including `ArrayBuffer` transfer without copy.

## The contextBridge API shape

```ts
// preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('api', {
  store: {
    get: (key: string) => ipcRenderer.invoke('store:get', key),
    set: (key: string, value: unknown) => ipcRenderer.invoke('store:set', key, value),
  },
  dialog: {
    pickFile: () => ipcRenderer.invoke('dialog:pick-file'),
  },
  on: {
    updateAvailable: (cb: (v: string) => void) => {
      const listener = (_: unknown, version: string) => cb(version)
      ipcRenderer.on('update:available', listener)
      return () => ipcRenderer.off('update:available', listener)
    },
  },
})
```

Expose a *namespaced* object, not a flat set of functions. Makes it easier to evolve and to type. Return an `off` function from every event subscription so React effects can clean up — see `memory-optimization.md`.

## Typing across the bridge

```ts
// types/global.d.ts
export interface AppApi {
  store: {
    get<T = unknown>(key: string): Promise<T | null>
    set(key: string, value: unknown): Promise<void>
  }
  dialog: { pickFile(): Promise<string | null> }
  on: { updateAvailable(cb: (v: string) => void): () => void }
}
declare global {
  interface Window { api: AppApi }
}
```

Share the same type between preload and renderer by having both import from `src/shared/api.ts` (TS-only, no runtime).

## Main-side handler

```ts
// main/ipc.ts
import { ipcMain } from 'electron'

ipcMain.handle('store:get', async (_event, key: string) => {
  return store.get(key)
})
```

Validate every argument. The renderer is your threat model — treat preload IPC surface like you'd treat a public HTTP API. A buggy or compromised renderer can pass anything.

```ts
ipcMain.handle('fs:read', async (_e, p: unknown) => {
  if (typeof p !== 'string') throw new Error('invalid')
  const abs = path.resolve(USER_DATA, p)
  if (!abs.startsWith(USER_DATA)) throw new Error('path traversal')
  return fs.readFile(abs, 'utf8')
})
```

## Don't do this

- `ipcRenderer.sendSync` — blocks the renderer and the main process. Use `invoke`.
- Expose `ipcRenderer` itself via contextBridge — this is equivalent to disabling isolation because any channel is now reachable.
- Serialize huge blobs through IPC. For files, send a path; for buffers, use `MessagePort` with transferable `ArrayBuffer`.
- Handlers that silently catch and swallow — errors should propagate to the renderer's `await` so it can surface them.
