# Window Management

## Custom Title Bar

Remove the OS title bar and build your own in HTML:

```ts
const win = new BrowserWindow({
  titleBarStyle: 'hidden',
  // Windows/Linux: re-expose native window controls
  ...(process.platform !== 'darwin' ? { titleBarOverlay: true } : {}),
});
```

Make the custom title bar draggable with CSS:

```css
.titlebar {
  app-region: drag;
  height: 32px;
}
.titlebar button {
  app-region: no-drag; /* buttons must be clickable */
}
```

### Customizing titleBarOverlay (Windows/Linux)

```ts
titleBarOverlay: {
  color: '#2f3241',
  symbolColor: '#74b1be',
  height: 60,
}
```

### macOS Traffic Lights

```ts
// Hidden inset (shift down)
{ titleBarStyle: 'hiddenInset' }

// Precise positioning
{ titleBarStyle: 'hidden', trafficLightPosition: { x: 10, y: 10 } }

// Hide on hover
{ titleBarStyle: 'customButtonsOnHover' }

// Programmatic toggle
win.setWindowButtonVisibility(false);
```

## Custom Window Styles

### Frameless Window

```ts
const win = new BrowserWindow({ frame: false });
```

Requires implementing your own close/minimize/maximize buttons and drag regions.

### Transparent Window

```ts
const win = new BrowserWindow({
  transparent: true,
  frame: false,
  backgroundColor: '#00000000',
});
```

### Window Vibrancy (macOS)

```ts
const win = new BrowserWindow({
  vibrancy: 'under-window',
  visualEffectState: 'active',
});
```

### Background Material (Windows)

```ts
const win = new BrowserWindow({
  backgroundMaterial: 'acrylic', // or 'mica', 'tabbed'
});
```

## Custom Window Interactions

### Draggable Regions

Any element with `app-region: drag` becomes a drag handle. Mark interactive children with `app-region: no-drag`.

### Minimum/Maximum Size

```ts
win.setMinimumSize(400, 300);
win.setMaximumSize(1920, 1080);
```

### Always On Top

```ts
win.setAlwaysOnTop(true, 'floating');
```

### Show Without Focus

```ts
const win = new BrowserWindow({ focusable: false });
win.showInactive();
```

### Prevent Close (Confirm Dialog)

```ts
win.on('close', (e) => {
  if (hasUnsavedChanges) {
    e.preventDefault();
    dialog.showMessageBox(win, {
      type: 'question',
      buttons: ['Save', 'Discard', 'Cancel'],
      message: 'Unsaved changes',
    }).then(({ response }) => {
      if (response === 0) save().then(() => win.destroy());
      if (response === 1) win.destroy();
    });
  }
});
```

## Multi-Window Management

```ts
// Track all windows
const windows = new Set<BrowserWindow>();

function createWindow(route: string): BrowserWindow {
  const win = new BrowserWindow({ /* ... */ });
  windows.add(win);
  win.on('closed', () => windows.delete(win));
  win.loadURL(`app://renderer/${route}`);
  return win;
}

// Restore or create
function showOrCreate(route: string) {
  const existing = [...windows].find(w => w.getURL().includes(route));
  if (existing) { existing.focus(); return existing; }
  return createWindow(route);
}
```

## Progress Bar

```ts
// Taskbar/Dock progress (0-1, or -1 to remove)
win.setProgressBar(0.5);
win.setProgressBar(-1); // clear
```

## Navigation History

```ts
// Go back/forward in renderer navigation
win.webContents.navigationHistory.goBack();
win.webContents.navigationHistory.goForward();
win.webContents.navigationHistory.canGoBack();
```

> **Ref:** [Custom Title Bar](https://www.electronjs.org/docs/latest/tutorial/custom-title-bar) · [Custom Window Styles](https://www.electronjs.org/docs/latest/tutorial/custom-window-styles) · [Custom Window Interactions](https://www.electronjs.org/docs/latest/tutorial/custom-window-interactions) · [Progress Bar](https://www.electronjs.org/docs/latest/tutorial/progress-bar) · [Navigation History](https://www.electronjs.org/docs/latest/tutorial/navigation-history)
