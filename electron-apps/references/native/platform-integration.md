# Native Modules & Platform Integration

## Cross-Platform APIs

### Device Access

```ts
// Screen capture
const sources = await desktopCapturer.getSources({ types: ['window', 'screen'] });

// Clipboard
import { clipboard } from 'electron';
clipboard.writeText('Hello');
const text = clipboard.readText();

// Camera / Microphone — uses Chromium's getUserMedia
// Configure permission handler in main:
win.webContents.session.setPermissionRequestHandler(
  (_wc, permission, callback) => {
    callback(['media', 'mediaKeySystem'].includes(permission));
  }
);

// System audio capture (macOS 13+): use desktopCapturer with audio flag
// Renderer:
navigator.mediaDevices.getUserMedia({ audio: { mandatory: { chromeMediaSource: 'desktop' } } });
```

### Power & System

```ts
import { powerMonitor, powerSaveBlocker } from 'electron';

powerMonitor.on('suspend', () => { /* pause background tasks */ });
powerMonitor.on('resume', () => { /* resume sync */ });
powerMonitor.on('lock-screen', () => { /* stop sensitive ops */ });
powerMonitor.on('on-battery', () => { /* reduce polling frequency */ });

// Prevent sleep during long operations
const id = powerSaveBlocker.start('prevent-display-sleep');
// ... long operation ...
powerSaveBlocker.stop(id);
```

### Startup

```ts
app.setLoginItemSettings({
  openAtLogin: true,
  openAsHidden: true, // macOS: open without showing window
});
```

### Storage Paradigms

| Approach | Use case | API |
|----------|----------|-----|
| `electron-store` | Small config/preferences | JSON file with atomic writes |
| SQLite (better-sqlite3) | Structured data, queries | Sync API in main, invoke from renderer |
| IndexedDB | Renderer-local cache | Browser API |
| `safeStorage` | Secrets, tokens | OS keychain/keyring encryption |
| `app.getPath('userData')` | Base directory for all app data | OS-specific path |

### Network Interception

```ts
// Intercept and modify requests
session.defaultSession.webRequest.onBeforeRequest(
  { urls: ['*://*.example.com/*'] },
  (details, callback) => {
    callback({ cancel: false, redirectURL: details.url.replace('http:', 'https:') });
  }
);

// Inspect responses
session.defaultSession.webRequest.onCompleted((details) => {
  log(`${details.method} ${details.url} → ${details.statusCode}`);
});
```

### Safe Key Storage

```ts
import { safeStorage } from 'electron';

// Check availability (not available on all Linux distros)
if (safeStorage.isEncryptionAvailable()) {
  const encrypted = safeStorage.encryptString(apiKey);
  fs.writeFileSync(keyPath, encrypted);

  const decrypted = safeStorage.decryptString(fs.readFileSync(keyPath));
}
```

Backed by: macOS Keychain, Windows DPAPI, Linux libsecret/kwallet.

## macOS Specifics

### Swift Sidecars

Run native Swift processes alongside Electron for platform-specific capabilities:

```ts
// Spawn Swift binary as sidecar
const sidecar = utilityProcess.fork(path.join(__dirname, 'macos-sidecar'));
// Or use child_process for non-Node binaries
const swift = spawn(path.join(resourcesPath, 'MacHelper'), ['--arg']);
```

Use cases: MLX inference, Vision framework OCR, native share sheets, Touch Bar integration.

### macOS Permissions

```ts
// Check permission status
const status = systemPreferences.getMediaAccessStatus('camera');
// 'not-determined' | 'granted' | 'denied' | 'restricted' | 'unknown'

// Request permission
const granted = await systemPreferences.askForMediaAccess('microphone');
```

### App Icon & Dock

```ts
app.dock.setBadge('3');        // dock badge
app.dock.setIcon(nativeImage); // custom dock icon
app.dock.bounce('critical');   // attention bounce
```

## Windows Specifics

### Windows-Specific Features

```ts
// Taskbar progress
win.setProgressBar(0.5); // 0-1
win.setProgressBar(-1);  // remove

// Thumbnail toolbar buttons
win.setThumbarButtons([{
  tooltip: 'Play',
  icon: nativeImage.createFromPath('play.png'),
  click: () => { /* play */ },
}]);

// Jump list
app.setJumpList([{
  type: 'custom',
  name: 'Recent',
  items: [{ type: 'file', path: 'C:\\recent.txt' }],
}]);
```

### Windows Safe Storage

Uses DPAPI (Data Protection API) via `safeStorage` — same API as macOS, different backend. No additional setup required.

## Native UI Components

### Application Menu

```ts
import { Menu } from 'electron';

const template: MenuItemConstructorOptions[] = [
  {
    label: 'File',
    submenu: [
      { label: 'New', accelerator: 'CmdOrCtrl+N', click: handleNew },
      { type: 'separator' },
      { role: 'quit' },
    ],
  },
  { role: 'editMenu' },
  { role: 'viewMenu' },
];

Menu.setApplicationMenu(Menu.buildFromTemplate(template));
```

### Context Menu

```ts
win.webContents.on('context-menu', (_event, params) => {
  const menu = Menu.buildFromTemplate([
    { label: 'Copy', role: 'copy', enabled: params.editFlags.canCopy },
    { label: 'Paste', role: 'paste' },
  ]);
  menu.popup();
});
```

### System Tray

```ts
const tray = new Tray(nativeImage.createFromPath('icon.png'));
tray.setToolTip('My App');
tray.setContextMenu(Menu.buildFromTemplate([
  { label: 'Show', click: () => win.show() },
  { label: 'Quit', click: () => app.quit() },
]));
```

### Notifications

```ts
new Notification({
  title: 'Update Available',
  body: 'Version 2.0 is ready to install.',
  icon: nativeImage.createFromPath('icon.png'),
}).show();
```

### Native Appearance

```ts
import { nativeTheme } from 'electron';

nativeTheme.on('updated', () => {
  const isDark = nativeTheme.shouldUseDarkColors;
  win.webContents.send('theme-changed', isDark);
});

// Set theme
nativeTheme.themeSource = 'system'; // 'dark' | 'light' | 'system'
```

> **Ref:** [desktopCapturer](https://www.electronjs.org/docs/latest/api/desktop-capturer) · [powerMonitor](https://www.electronjs.org/docs/latest/api/power-monitor) · [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage) · [systemPreferences](https://www.electronjs.org/docs/latest/api/system-preferences) · [Tray](https://www.electronjs.org/docs/latest/api/tray)
