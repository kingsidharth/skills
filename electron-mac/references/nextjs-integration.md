# Next.js integration

## Two modes

**Static export (`output: 'export'`)** — Next.js emits static HTML/JS into `out/`. Electron loads it via `file://` or a custom protocol. No Node server at runtime. Works for App Router with client-rendered pages.

**Standalone (`output: 'standalone'`)** — Next.js emits `.next/standalone/` with a minimal Node server. Electron spawns that server on a free port and loads `http://localhost:<port>`. Required if you use Server Components that need to render at runtime, or Route Handlers.

Static is the right default. Standalone roughly doubles startup time and packaged size.

## Static export config

```ts
// src/renderer/next.config.ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  output: 'export',
  distDir: 'out',
  trailingSlash: true,        // home.html -> home/index.html, matches loadFile paths
  images: { unoptimized: true }, // `next/image` optimization needs a server
  assetPrefix: './',          // relative paths for file:// loading
}

export default config
```

App Router `loading.tsx`, `error.tsx`, parallel/intercepting routes all work in static export as long as the page itself is a Client Component or a fully-static Server Component. Dynamic segments require `generateStaticParams`.

## Custom protocol (recommended over file://)

```ts
import { app, protocol, net } from 'electron'
import path from 'node:path'
import { pathToFileURL } from 'node:url'

protocol.registerSchemesAsPrivileged([
  { scheme: 'app', privileges: { standard: true, secure: true, supportFetchAPI: true, corsEnabled: true } },
])

app.whenReady().then(() => {
  protocol.handle('app', (req) => {
    const url = new URL(req.url)
    const file = url.pathname === '/' ? 'index.html' : url.pathname
    const abs = path.join(app.getAppPath(), 'renderer/out', file)
    return net.fetch(pathToFileURL(abs).toString())
  })
  win.loadURL('app://local/')
})
```

Gives you HTTPS-equivalent security context (cookies, service workers, `fetch` same-origin) without the `file://` restrictions.

## Standalone server in Electron

```ts
import { getPort } from 'get-port-please'
import { fork } from 'node:child_process'

const port = await getPort({ portRange: [3000, 3999] })
const server = fork(path.join(app.getAppPath(), 'renderer/server.js'), {
  env: { ...process.env, PORT: String(port), NODE_ENV: 'production' },
})
await waitForPort(port)
win.loadURL(`http://localhost:${port}`)

app.on('before-quit', () => server.kill())
```

Package the standalone output via `asarUnpack` so Next can read its files. See `packaging.md`.

## Dev vs prod loading

```ts
const isDev = !app.isPackaged
if (isDev) {
  win.loadURL('http://localhost:3000')
  win.webContents.openDevTools({ mode: 'detach' })
} else {
  win.loadURL('app://local/')
}
```

Keep dev pointing at `next dev` for HMR. Prod goes through the protocol handler.

## Server Components caveats

In static export, Server Components run at build time. They cannot read user-specific data or call Electron APIs. To use RSC with runtime Electron data, go standalone and call Electron IPC from Client Components as usual.

## App Router + IPC

IPC only works in Client Components. Wrap the preload API in a hook:

```ts
'use client'
export function useStore<T>(key: string) {
  const [value, setValue] = useState<T | null>(null)
  useEffect(() => {
    window.api.store.get<T>(key).then(setValue)
  }, [key])
  return [value, (v: T) => { setValue(v); window.api.store.set(key, v) }] as const
}
```

## Image optimization

`next/image` optimization requires a server. For static export, use `unoptimized: true` and handle resizing yourself (pre-bake at build, or use a native image library in the main process).

## Fonts

`next/font` works in static export — Next bundles the woff2 files into `out/_next/static/media`. Local font files are served from the protocol handler. No Google Fonts network hit at runtime.
