# File system

All filesystem access happens in the main process. The renderer requests via IPC. With sandboxing on, even the renderer's own `fetch` to `file://` is blocked.

## Standard paths

```ts
app.getPath('userData')   // ~/Library/Application Support/MyApp
app.getPath('home')       // ~
app.getPath('appData')    // ~/Library/Application Support
app.getPath('temp')       // os tmp
app.getPath('downloads')  // ~/Downloads
app.getPath('documents')  // ~/Documents
app.getPath('desktop')    // ~/Desktop
app.getPath('logs')       // ~/Library/Logs/MyApp
```

`userData` is the canonical place for app data — survives updates, sandboxed correctly under Mac App Store, never collides with another app. The path includes your `app.name`, so set that early via `app.setName(...)`.

## Reading and writing

```ts
import { app } from 'electron'
import fs from 'node:fs/promises'
import path from 'node:path'

const USER_DATA = app.getPath('userData')

ipcMain.handle('fs:read', async (_e, relPath: string) => {
  const abs = path.resolve(USER_DATA, relPath)
  if (!abs.startsWith(USER_DATA)) throw new Error('outside userData')
  return fs.readFile(abs, 'utf8')
})
```

Always resolve to absolute and check the prefix. Path traversal is the most common bug here.

## Dialogs

```ts
import { dialog } from 'electron'

const { canceled, filePaths } = await dialog.showOpenDialog(win, {
  title: 'Pick a project',
  properties: ['openDirectory', 'createDirectory'],
})

const { canceled, filePath } = await dialog.showSaveDialog(win, {
  defaultPath: 'Untitled.json',
  filters: [{ name: 'JSON', extensions: ['json'] }],
})
```

Pass the `BrowserWindow` as the first argument so the dialog appears as a sheet attached to the window rather than a free-floating modal.

`properties` for open: `'openFile' | 'openDirectory' | 'multiSelections' | 'showHiddenFiles' | 'createDirectory' | 'promptToCreate' | 'noResolveAliases' | 'treatPackageAsDirectory' | 'dontAddToRecent'`.

## Drag and drop into the renderer

The renderer can use the standard HTML5 drag-and-drop API. Dropped files have a `.path` attribute on the `File` object (Electron extension):

```ts
function onDrop(e: DragEvent) {
  e.preventDefault()
  for (const file of e.dataTransfer?.files ?? []) {
    const path = (file as File & { path: string }).path
    window.api.fs.import(path)
  }
}
```

Send the path to main; don't try to read the file in the renderer — under sandbox, you can't.

## Drag from the renderer to other apps

```ts
ipcMain.on('drag-out', (event, filePath: string) => {
  event.sender.startDrag({
    file: filePath,
    icon: nativeImage.createFromPath(filePath).resize({ width: 64 }),
  })
})
```

Renderer initiates: `window.api.startDrag(localPath)`; main calls `webContents.startDrag`.

## File watchers

Use `chokidar` (battle-tested) over raw `fs.watch` (platform inconsistencies, missed events). Watch in main, notify renderer over IPC. Be careful with watcher count — macOS has FSEvents quotas; watching `~` recursively will hit them.

## Custom protocols

Register a protocol to serve files from `userData` to the renderer with proper content-types and CSP:

```ts
protocol.handle('user', async (req) => {
  const url = new URL(req.url)
  const abs = path.join(USER_DATA, url.pathname)
  if (!abs.startsWith(USER_DATA)) return new Response('forbidden', { status: 403 })
  return net.fetch(pathToFileURL(abs).toString())
})
```

Register the scheme as `secure: true, standard: true, supportFetchAPI: true` for `fetch` to work cross-origin in the renderer.

## Showing in Finder

```ts
import { shell } from 'electron'
shell.showItemInFinder(path)   // open Finder, select the item
shell.openPath(path)           // open with default app
shell.trashItem(path)          // move to Trash (async)
```

`trashItem` is the safe way to "delete" — it's reversible from the user's POV. Don't `fs.unlink` user-facing files.

## Mac App Store sandbox

Under MAS sandbox, you can only read paths the user has granted via dialog/drag. Calls to `fs.readFile` on unrelated paths fail silently. Use `app.getPath('userData')` (always permitted) for app-internal data and security-scoped bookmarks for user-chosen paths if persistence across launches is needed.
