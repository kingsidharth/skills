# Sidebar layout

macOS sidebar conventions: translucent blur, inset traffic lights, content that scrolls independently, and optional collapse.

## Minimum viable sidebar

```jsx
<div className="flex h-screen bg-transparent">
  <aside className="
    w-64 [-webkit-app-region:drag]
    bg-white/0 border-r border-black/5 dark:border-white/10
  ">
    <div className="h-14" /> {/* room for traffic lights */}
    <nav className="[-webkit-app-region:no-drag] px-2">
      {/* links */}
    </nav>
  </aside>
  <main className="flex-1 bg-white dark:bg-neutral-900 overflow-auto">
    {children}
  </main>
</div>
```

Key moves:

- Sidebar uses `bg-transparent` (or `bg-white/0`) so the window's `vibrancy: 'sidebar'` material shows through.
- Main content is opaque — vibrancy is restricted to the sidebar area.
- `[-webkit-app-region:drag]` on the sidebar makes it a drag handle; every interactive child restores `no-drag`.
- Leave ~56px of empty space at the top of the sidebar for traffic lights (when `trafficLightPosition: { x: 16, y: 16 }`, clear area is roughly 80×30px top-left).

## Window setup for this layout

```ts
new BrowserWindow({
  titleBarStyle: 'hiddenInset',
  trafficLightPosition: { x: 16, y: 16 },
  vibrancy: 'sidebar',
  visualEffectState: 'active',
  backgroundColor: '#00000000',
  width: 1200,
  height: 800,
})
```

## Collapsible sidebar

Keep the collapsed state in the main process (persist to `electron-store`) so window restoration matches. Apps like Linear and Notion tween width from 256 → 48px with traffic light position adjusting on collapse:

```ts
win.setWindowButtonVisibility(!collapsed)
// or animate traffic light position
win.setTrafficLightPosition({ x: collapsed ? 12 : 16, y: collapsed ? 12 : 16 })
```

## Responsive behavior

Electron windows can be resized by the user. Define clear breakpoints:

- `<900px` — sidebar overlays instead of displacing content; close-on-click-outside
- `900–1200px` — standard 64px or 220px sidebar
- `>1200px` — wider sidebar allowed, or reveal a secondary column

Use `win.on('resize', ...)` only if you need native-side logic. For CSS, a container query on the root (`@container (max-width: 900px)`) is better than `window.matchMedia` because it survives multi-display moves cleanly.

## Title bar overlay (cross-platform)

If shipping to Windows/Linux too, `titleBarOverlay: true` on those platforms surfaces the native window controls at a known position. On macOS it's ignored — your traffic lights come from `titleBarStyle`.

```ts
new BrowserWindow({
  titleBarStyle: 'hidden',
  ...(process.platform !== 'darwin' ? { titleBarOverlay: { color: '#00000000', symbolColor: '#ffffff', height: 44 } } : {}),
})
```

## Common bugs

- **Traffic lights overlap sidebar content** — increase top padding or move lights with `trafficLightPosition`.
- **Vibrancy flickers on resize** — caused by the renderer painting an opaque background during resize. Set `backgroundColor: '#00000000'` on the BrowserWindow.
- **Can't drag the window** — you forgot `-webkit-app-region: drag` somewhere reachable. The title bar area is not automatically draggable when `titleBarStyle` is `hidden` or `hiddenInset`.
- **Clicks pass through transparent sidebar** — only an issue with `transparent: true` on BrowserWindow (different from vibrancy). Avoid that combo.
- **Sidebar doesn't blur the content behind the window** — vibrancy only blurs *behind the window*, not behind main content. If you want blur between sidebar and content, use `backdrop-filter: blur(...)` on a CSS element, not vibrancy.
