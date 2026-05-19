# Patterns & Recipes

## Keyboard Shortcuts

Three levels of keyboard shortcuts:

### Local (Menu Accelerators)

```ts
Menu.setApplicationMenu(Menu.buildFromTemplate([{
  label: 'File',
  submenu: [{
    label: 'Save',
    accelerator: 'CmdOrCtrl+S',
    click: () => handleSave(),
  }],
}]));
```

Use `CmdOrCtrl` for cross-platform (maps to ⌘ on macOS, Ctrl on Windows/Linux).

### Global (System-Wide)

```ts
import { globalShortcut } from 'electron';
globalShortcut.register('CmdOrCtrl+Shift+Space', () => toggleQuickCapture());
app.on('will-quit', () => globalShortcut.unregisterAll());
```

### Window-Scoped (before-input-event)

Intercept keypresses in main before they reach the renderer:

```ts
win.webContents.on('before-input-event', (event, input) => {
  if (input.control && input.key.toLowerCase() === 'i') {
    event.preventDefault();
    toggleDevTools();
  }
});
```

Or handle in renderer via standard `addEventListener('keydown', ...)`.

## Deep Links (Custom Protocol)

Register your app as the handler for a custom URL scheme (`myapp://`):

```ts
if (process.defaultApp) {
  app.setAsDefaultProtocolClient('myapp', process.execPath, [path.resolve(process.argv[1])]);
} else {
  app.setAsDefaultProtocolClient('myapp');
}

// macOS: handle via open-url event
app.on('open-url', (event, url) => handleDeepLink(url));

// Windows/Linux: handle via second-instance event
const gotLock = app.requestSingleInstanceLock();
if (!gotLock) { app.quit(); }
else {
  app.on('second-instance', (_e, argv) => {
    const url = argv.pop();
    if (url?.startsWith('myapp://')) handleDeepLink(url);
    if (mainWindow?.isMinimized()) mainWindow.restore();
    mainWindow?.focus();
  });
}
```

**Packaging:** For macOS, set `packagerConfig.protocols` in Forge. For Linux, set `mimeType` in maker-deb config.

## Notifications

```ts
import { Notification } from 'electron';

new Notification({
  title: 'Download Complete',
  body: 'report-q4.pdf is ready',
  icon: nativeImage.createFromPath('icon.png'),
  silent: false,
}).show();

// Check support
Notification.isSupported();
```

On macOS: works out of the box. On Windows: app must have a Start Menu shortcut with an AppUserModelID. On Linux: uses `libnotify`.

## Spellchecker

Electron includes Chromium's built-in spellchecker. Configure languages:

```ts
win.webContents.session.setSpellCheckerLanguages(['en-US', 'es']);
```

Handle misspellings in context menu:

```ts
win.webContents.on('context-menu', (_e, params) => {
  if (params.misspelledWord) {
    const menu = Menu.buildFromTemplate(
      params.dictionarySuggestions.map(s => ({
        label: s,
        click: () => win.webContents.replaceMisspelling(s),
      }))
    );
    menu.popup();
  }
});
```

Add custom words: `win.webContents.session.addWordToSpellCheckerDictionary('customWord')`.

## Device Access

Electron provides Chromium's device APIs (Bluetooth, HID, Serial, USB) with programmatic device selection instead of browser popups.

```ts
// Bluetooth
win.webContents.on('select-bluetooth-device', (event, devices, callback) => {
  event.preventDefault();
  const target = devices.find(d => d.deviceName === 'MyDevice');
  callback(target?.deviceId ?? '');
});

// HID / Serial — use session handlers
session.defaultSession.on('select-hid-device', (event, details, callback) => {
  event.preventDefault();
  if (details.deviceList.length > 0) callback(details.deviceList[0].deviceId);
});

session.defaultSession.setDevicePermissionHandler((details) => {
  return details.deviceType === 'hid' && details.origin === 'app://';
});
```

## Offscreen Rendering

Render to a bitmap (for 3D textures, screenshots, video capture):

```ts
const win = new BrowserWindow({
  webPreferences: { offscreen: true },
});
win.webContents.on('paint', (_e, dirty, image) => {
  fs.writeFileSync('screenshot.png', image.toPNG());
});
win.webContents.setFrameRate(30);
```

GPU shared texture mode (`webPreferences.offscreen.useSharedTexture: true`) avoids CPU↔GPU copies for real-time compositing.

## Multithreading

Electron exposes multiple threading options:

| Option | Context | Node.js? | Use case |
|--------|---------|----------|----------|
| Web Workers | Renderer | No | CPU work off renderer main thread |
| `worker_threads` | Main | Yes | CPU work off main process thread |
| `utilityProcess` | Standalone | Yes | Heavy background tasks, sidecars |
| Hidden BrowserWindow | Renderer | Via preload | When you need DOM + Node |

Prefer `utilityProcess` over hidden windows for non-UI work — lower overhead.

## Background Server Pattern

Run a local server in a background process for data-heavy apps. All data loads from local storage (SQLite), not network:

- Data loads instantly — no network wait
- Little caching needed in renderer — reduces memory bloat
- Renderer stays responsive — heavy work happens in background
- Dev experience bonus: run server in a hidden window during development for Chrome DevTools profiling/debugging

```ts
// Production: spawn as utilityProcess
const server = utilityProcess.fork(path.join(__dirname, 'server.js'));

// Development: load in hidden BrowserWindow for DevTools access
const serverWin = new BrowserWindow({ show: false, webPreferences: { nodeIntegration: true } });
serverWin.loadFile('server.html');
```

In dev, Cmd+R in the server window restarts the server without restarting the UI. Profile startup, inspect state via console.

## Live Reloading in Development

Use Vite plugin with Forge for HMR in renderer. For main process changes, use `--watch` mode:

```json
{
  "scripts": {
    "dev": "electron-forge start"
  }
}
```

Forge's Vite plugin handles HMR for the renderer automatically. Main process restarts on file change.

For manual setups, use `electron-reloader` or `nodemon`:

```ts
// In main.ts (dev only)
if (process.env.NODE_ENV === 'development') {
  require('electron-reloader')(module, { watchRenderer: false });
}
```

## Windows Taskbar

```ts
// Thumbnail toolbar buttons
win.setThumbarButtons([{
  tooltip: 'Play',
  icon: nativeImage.createFromPath('play.png'),
  click: () => handlePlay(),
}]);

// Jump list
app.setJumpList([{
  type: 'custom',
  name: 'Recent Projects',
  items: [{ type: 'task', title: 'Open Last', program: process.execPath, args: '--last' }],
}]);

// Overlay icon (badge)
win.setOverlayIcon(nativeImage.createFromPath('badge.png'), '3 notifications');
```

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `ELECTRON_RUN_AS_NODE` | Run as plain Node.js |
| `ELECTRON_NO_ASAR` | Disable ASAR support |
| `ELECTRON_ENABLE_LOGGING` | Print Chromium logs to console |
| `NODE_OPTIONS` | Passed to Node.js (e.g. `--max-old-space-size`) |

> **Ref:** [Keyboard Shortcuts](https://www.electronjs.org/docs/latest/tutorial/keyboard-shortcuts) · [Deep Links](https://www.electronjs.org/docs/latest/tutorial/launch-app-from-url-in-another-app) · [Notifications](https://www.electronjs.org/docs/latest/tutorial/notifications) · [SpellChecker](https://www.electronjs.org/docs/latest/tutorial/spellchecker) · [Devices](https://www.electronjs.org/docs/latest/tutorial/devices) · [Offscreen](https://www.electronjs.org/docs/latest/tutorial/offscreen-rendering) · [Multithreading](https://www.electronjs.org/docs/latest/tutorial/multithreading) · [Taskbar](https://www.electronjs.org/docs/latest/tutorial/windows-taskbar) · [Background Server](https://archive.jlongster.com/secret-of-good-electron-apps)
