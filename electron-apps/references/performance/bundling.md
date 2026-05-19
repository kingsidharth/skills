# Bundling & Code Splitting

## Why Bundle

`require()` is synchronous and recursive. Each call: resolves module path → reads file from disk → compiles → executes. This cascades through the dependency tree and blocks both main and renderer.

A bundler (Vite, esbuild, webpack) collapses your dependency tree into one or few files, eliminating thousands of `require()` calls. Non-negotiable for production Electron apps.

## Strategy: One Bundle Per Process

- `main.bundle.js` — main process entry
- `preload.bundle.js` — preload script
- `renderer.bundle.js` — renderer entry (with code splitting for routes)

Each process gets its own optimized bundle. The preload bundle should be tiny — it only exposes `contextBridge` APIs.

## Code Splitting

Use dynamic `import()` for route-based splitting:

```tsx
const Home = lazy(() => import('./pages/Home'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<AppShell />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

The bundler splits each lazy route into its own chunk, loaded on demand.

## Tree Shaking

- Target the exact Chromium version you ship — no polyfills needed
- Set `sideEffects: false` in `package.json` where applicable
- Use ESM imports — CJS `require()` is harder to tree-shake

## Using Bun

### Bun as Package Manager Only (Node Runtime)

```bash
# Install dependencies with bun (faster than npm/yarn)
bun install

# Run with Node.js runtime (Electron requires Node)
npx electron .
```

`bun install` is significantly faster than npm/yarn for dependency resolution and installation. Compatible with `node_modules` layout that Electron expects.

### Bun as Bundler

```bash
# Bundle main process
bun build src/main.ts --outdir=dist --target=node --external electron

# Bundle preload
bun build src/preload.ts --outdir=dist --target=node --external electron

# Bundle renderer
bun build src/renderer.ts --outdir=dist --target=browser --splitting
```

Bun's bundler is fast but less mature than Vite/esbuild for complex scenarios. For Electron Forge integration, Vite plugin is more battle-tested.

**Key constraint:** Electron's main process runs on Node.js (not Bun's runtime). Use Bun for dependency management and bundling, but the runtime is always Node.

### Practical Setup

```json
{
  "scripts": {
    "install": "bun install",
    "build:main": "bun build src/main.ts --outdir=dist --target=node --external electron",
    "build:renderer": "bun build src/renderer/index.tsx --outdir=dist/renderer --target=browser --splitting",
    "start": "electron dist/main.js",
    "dev": "concurrently \"bun run build:main --watch\" \"bun run build:renderer --watch\" \"electron .\""
  }
}
```

## CSP Implications

Some bundler output modes emit `eval()`-like code (Webpack `devtool: 'eval'`). This conflicts with strict CSP. Configure:

```ts
// webpack
devtool: 'source-map', // not 'eval' or 'cheap-eval-source-map'
```

```ts
// vite
build: { sourcemap: true } // uses separate .map files
```

> **Ref:** [Palette Perf Guide: Bundlers](https://palette.dev/blog/improving-performance-of-electron-apps) · [Electron Performance: Bundling](https://www.electronjs.org/docs/latest/tutorial/performance)
