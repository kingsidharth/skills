# Menus, Dock, Tray

## Application menu

macOS requires an application menu. If you don't set one, you get Electron's default ("Electron" as the app name, unhelpful items). Set it in `app.whenReady()`:

```ts
import { app, Menu, shell } from 'electron'

const template: Electron.MenuItemConstructorOptions[] = [
  // First item on macOS must be the app name
  {
    label: app.name,
    submenu: [
      { role: 'about' },
      { type: 'separator' },
      { label: 'Preferences…', accelerator: 'Cmd+,', click: () => openSettings() },
      { type: 'separator' },
      { role: 'services' },
      { type: 'separator' },
      { role: 'hide' },
      { role: 'hideOthers' },
      { role: 'unhide' },
      { type: 'separator' },
      { role: 'quit' },
    ],
  },
  {
    label: 'File',
    submenu: [{ label: 'New', accelerator: 'Cmd+N', click: () => createNewDoc() }],
  },
  { role: 'editMenu' },
  { role: 'viewMenu' },
  { role: 'windowMenu' },
  {
    role: 'help',
    submenu: [{ label: 'Documentation', click: () => shell.openExternal('https://…') }],
  },
]

Menu.setApplicationMenu(Menu.buildFromTemplate(template))
```

Use `role` shortcuts — they produce platform-correct labels, accelerators, and behaviors (including `services`, `startSpeaking`, etc. on macOS).

## Context menu

Right-click menus must be implemented per-window with `webContents.on('context-menu', ...)`. `electron-context-menu` (Sindre Sorhus) gives sane defaults (copy, paste, inspect, lookup) in one line.

## Dock menu

Right-click on dock icon:

```ts
const dockMenu = Menu.buildFromTemplate([
  { label: 'New Window', click: () => createWindow() },
  { label: 'New Private Window', click: () => createPrivateWindow() },
])
app.dock?.setMenu(dockMenu)
```

`app.dock` is only defined on macOS — optional chaining saves cross-platform crashes.

## Dock badge and progress

```ts
app.dock?.setBadge('3')
app.dock?.setBadge('')      // clear

win.setProgressBar(0.4)     // progress bar under dock icon
win.setProgressBar(-1)      // hide
```

Badge is a string, not a number. Common use: unread count.

## Bouncing dock icon

```ts
const bounceId = app.dock?.bounce('critical')  // or 'informational'
app.dock?.cancelBounce(bounceId!)
```

Use sparingly — users find this intrusive.

## Tray (menu bar icon)

```ts
import { Tray, Menu, nativeImage } from 'electron'

const icon = nativeImage.createFromPath(path.join(__dirname, 'trayTemplate.png'))
const tray = new Tray(icon)
tray.setToolTip('MyApp')
tray.setContextMenu(Menu.buildFromTemplate([
  { label: 'Open', click: () => win.show() },
  { label: 'Quit', role: 'quit' },
]))
```

macOS template images: file name must end `Template.png` (or `Template@2x.png`). Image is monochrome with alpha; macOS inverts it for dark menu bars automatically. Make it 22x22pt (44x44px @2x).

Tray click vs right-click: on macOS, any click opens the context menu by default. For click-to-toggle-popover behavior, use `tray.on('click', () => togglePopover())` and leave the context menu off.

## Keeping app alive without windows

```ts
app.on('window-all-closed', () => {
  // On macOS, keep app running (dock/tray/menubar stays)
  if (process.platform !== 'darwin') app.quit()
})

app.on('activate', () => {
  // Dock click: re-create main window if none exists
  if (BrowserWindow.getAllWindows().length === 0) createMainWindow()
})
```

For menu-bar-only apps (Rectangle, Bartender style): skip creating a main window at all, just set up the tray. Hide the dock icon with `app.dock?.hide()`.

## System appearance

`nativeTheme.themeSource` — force light/dark/system. Respect user's system preference unless your app specifically themes itself independently.

```ts
nativeTheme.on('updated', () => {
  tray.setImage(nativeTheme.shouldUseDarkColors ? darkIcon : lightIcon)
})
```

Template images auto-adapt; separate icons are only needed if the icons aren't template-style.
