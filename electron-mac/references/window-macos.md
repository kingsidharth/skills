# Window (macOS)

## Title bar styles

| Value | Behavior |
|---|---|
| `default` | Standard macOS title bar |
| `hidden` | Title bar removed, traffic lights visible, content fills window |
| `hiddenInset` | Same as `hidden` but traffic lights inset further from edge |
| `customButtonsOnHover` | (Deprecated, don't use) |

`hiddenInset` is the standard for modern macOS apps (Linear, Notion, Raycast). Keeps traffic lights but gives you full-height content.

```ts
new BrowserWindow({
  titleBarStyle: 'hiddenInset',
  trafficLightPosition: { x: 16, y: 16 },  // align with your sidebar gutter
  width: 1200,
  height: 800,
  minWidth: 600,
  minHeight: 400,
})
```

## Traffic lights

`trafficLightPosition` moves the close/minimize/zoom cluster. Position `{ x: 16, y: 16 }` matches a typical 16-24px sidebar padding. Use `win.setWindowButtonVisibility(false)` to hide them (rare — only for splash screens or very custom chrome).

For multi-window apps, position per window. Settings modals usually want default position; main window wants a custom inset.

## Vibrancy (blur/translucency)

| Material | When to use |
|---|---|
| `sidebar` | Sidebar/panel background — the canonical macOS translucent sidebar |
| `hud` | Floating overlays |
| `popover` | Popover panels |
| `window` | Full-window frosted background |
| `under-window` | Low-intensity background |
| `fullscreen-ui` | UI bars in fullscreen |
| `content` | Generic content area |
| `menu` / `selection` / `titlebar` | Matches those native surfaces |

```ts
new BrowserWindow({
  vibrancy: 'sidebar',
  visualEffectState: 'active',  // stays lit even when window loses focus
  backgroundColor: '#00000000', // transparent to let vibrancy through
})
```

`visualEffectState` values: `followWindow` (default — dims when inactive), `active`, `inactive`. `active` gives Raycast/Arc-style always-vibrant feel.

Vibrancy only affects pixels the renderer hasn't painted over. Sidebar element needs `background: transparent` (or semi-transparent rgba) to show the blur. A full opaque `body { background: white }` defeats it.

## Frameless with custom controls

```ts
new BrowserWindow({ frame: false, titleBarStyle: 'hidden' })
```

Frameless removes both the title bar AND the traffic lights. You're responsible for rebuilding close/minimize/zoom. Usually overkill — `titleBarStyle: 'hidden'` keeps the lights and you just hide your own chrome.

## Drag regions

macOS needs to know which parts of the window are draggable (move the window) vs. interactive (click targets).

```css
.titlebar     { -webkit-app-region: drag; }
.titlebar button, .titlebar input { -webkit-app-region: no-drag; }
```

Default: nothing is draggable. If you hide the title bar you must mark a drag region or the user can't move the window. Don't apply `drag` to an element that has children with their own click handlers — event ordering gets weird.

## Rounded corners

`roundedCorners: true` (default on macOS) — follows system. `false` gives square-cornered frameless windows, rare.

## Background color and transparency

```ts
new BrowserWindow({
  transparent: true,           // window background fully transparent
  backgroundColor: '#00000000',
  hasShadow: false,            // usually paired with transparent
})
```

Transparent windows don't play well with vibrancy (vibrancy *is* the transparency). Use one or the other. Transparent is for fully custom shapes (HUDs, picker overlays); vibrancy is for frosted standard surfaces.

## Dark mode

```ts
import { nativeTheme } from 'electron'

nativeTheme.themeSource = 'system'  // 'light' | 'dark' | 'system'
nativeTheme.on('updated', () => {
  win.webContents.send('theme-updated', nativeTheme.shouldUseDarkColors)
})
```

Vibrancy materials automatically adapt to system appearance. Your CSS should read `prefers-color-scheme` to match.

## Fullscreen

`titleBarStyle: 'hiddenInset'` + fullscreen hides the traffic lights automatically. Listen to `enter-full-screen` / `leave-full-screen` on the window and tell the renderer to adjust its title-bar drag region.

## Setting window level

`win.setAlwaysOnTop(true, 'floating')` — levels: `normal`, `floating`, `torn-off-menu`, `modal-panel`, `main-menu`, `status`, `pop-up-menu`, `screen-saver`. Use `floating` for heads-up widgets.
