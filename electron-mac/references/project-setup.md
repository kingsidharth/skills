# Project setup

## Layout

```
app/
├── src/
│   ├── main/           # Main process (Electron)
│   │   ├── index.ts
│   │   ├── windows.ts
│   │   ├── ipc.ts
│   │   └── menu.ts
│   ├── preload/
│   │   └── index.ts
│   └── renderer/       # Next.js App Router
│       ├── app/
│       ├── components/
│       └── next.config.ts
├── build/              # Icons, entitlements, certs (not bundled)
│   ├── icon.icns
│   ├── entitlements.mac.plist
│   └── entitlements.mas.plist
├── electron-builder.yml
├── tsconfig.main.json
├── tsconfig.renderer.json
├── package.json
└── bun.lockb
```

Main process is CommonJS or ESM depending on Electron version (40+ supports ESM). Keep `src/main` and `src/preload` as a separate TS project from `src/renderer` — they target Node, not a browser.

## package.json essentials

```json
{
  "main": "dist/main/index.js",
  "type": "module",
  "scripts": {
    "dev": "concurrently -k \"bun run dev:renderer\" \"bun run dev:main\"",
    "dev:renderer": "cd src/renderer && next dev -p 3000",
    "dev:main": "tsup --watch src/main src/preload --onSuccess \"electron .\"",
    "build:renderer": "cd src/renderer && next build",
    "build:main": "tsup src/main src/preload",
    "build": "bun run build:renderer && bun run build:main",
    "dist": "bun run build && electron-builder --mac --universal",
    "postinstall": "electron-builder install-app-deps"
  }
}
```

`postinstall` runs `@electron/rebuild` against native deps, ensuring N-API modules load under Electron's Node ABI.

## Two build modes

Static export is the default for desktop-first apps: `output: 'export'` in `next.config.ts`, no server needed, loaded via `file://` or a custom `app://` protocol.

Standalone is for apps that genuinely need Server Components or Route Handlers at runtime — bundle `.next/standalone`, spawn Node with a free port (via `get-port-please`), load `http://localhost:<port>`. Heavier and slower to start; avoid unless RSC is required.

See `references/nextjs-integration.md`.

## Bun

Bun installs dependencies. Electron itself still uses Node inside the packaged binary — Bun is not the runtime at runtime. Don't rely on Bun-specific APIs (`Bun.file`, `Bun.serve`) in main-process code; they won't exist in the shipped app.

## TypeScript

Two `tsconfig` files:
- `tsconfig.main.json` — `module: nodenext`, `target: es2022`, `types: ["node", "electron"]`
- `tsconfig.renderer.json` — extends Next.js defaults, DOM lib, no Node types

A shared `tsconfig.base.json` with `strict: true`, `noUncheckedIndexedAccess: true`, `verbatimModuleSyntax: true` is fine. Preload gets its own build — it runs in a renderer process but has `require`.

## Dev workflow

`electron .` launches the main process pointing at `http://localhost:3000` when `process.env.NODE_ENV === 'development'`, else at the built renderer. Use `app.isPackaged` rather than `NODE_ENV` to detect production at runtime — `NODE_ENV` is not reliable in packaged apps.

For hot reload of the main process, `tsup --watch` rebuilds, kills the Electron child, and respawns. `electron-vite` and `@electron-forge/plugin-vite` offer integrated HMR if Vite is acceptable.
