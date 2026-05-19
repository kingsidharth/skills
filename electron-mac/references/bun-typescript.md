# Bun, TypeScript, tsup

## Bun's role

Bun is the package manager and script runner. It is not the runtime inside the packaged app — Electron ships with Node. Avoid `Bun.*` APIs (`Bun.file`, `Bun.serve`, `Bun.password`) in main-process code; they'll fail once packaged.

Safe to use Bun for: installing deps, running dev scripts, tests, build tooling, CI.

## tsup for main + preload

`tsup` bundles TypeScript to JS with ESM or CJS output, picks up `tsconfig.main.json`, and has watch mode.

```ts
// tsup.config.ts
import { defineConfig } from 'tsup'

export default defineConfig([
  {
    entry: ['src/main/index.ts'],
    outDir: 'dist/main',
    format: 'esm',              // Electron 28+ supports ESM main
    target: 'node22',
    platform: 'node',
    sourcemap: true,
    clean: true,
    external: ['electron'],
    skipNodeModulesBundle: true,  // leave node_modules resolved at runtime
  },
  {
    entry: ['src/preload/index.ts'],
    outDir: 'dist/preload',
    format: 'cjs',              // sandboxed preload requires CJS
    target: 'node22',
    platform: 'node',
    sourcemap: true,
    external: ['electron'],
  },
])
```

Sandboxed preloads must be CJS — the sandbox does not support ESM.

Don't bundle `node_modules` into main — native modules must remain as `.node` files that `@electron/rebuild` can rebuild. Bundling breaks the rebuild step.

## tsconfig split

```jsonc
// tsconfig.base.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "moduleResolution": "bundler"
  }
}
```

```jsonc
// tsconfig.main.json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "module": "nodenext",
    "target": "es2022",
    "types": ["node"],
    "lib": ["es2023"],
    "outDir": "dist/main"
  },
  "include": ["src/main", "src/preload", "src/shared"]
}
```

```jsonc
// src/renderer/tsconfig.json  (Next.js)
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "target": "es2020",
    "lib": ["es2020", "dom", "dom.iterable"],
    "jsx": "preserve",
    "plugins": [{ "name": "next" }]
  }
}
```

Keep renderer out of the main tsconfig — it'll pull DOM types into the main process and confuse `require` resolution.

## Shared types

Put shared IPC types in `src/shared/`. Never share runtime code between main and renderer — a file imported from both will be bundled twice and referential equality won't hold across the bridge.

## Tailwind in renderer

Standard Next.js + Tailwind setup. Inside Electron, `-webkit-app-region: drag` is non-standard CSS, so add it as an arbitrary value:

```jsx
<div className="[-webkit-app-region:drag] h-10" />
<button className="[-webkit-app-region:no-drag]" />
```

Or drop a small layer in `globals.css` with `.drag` / `.no-drag` classes.

## Scripts

```json
{
  "scripts": {
    "dev": "concurrently -k \"bun run dev:renderer\" \"bun run dev:main\"",
    "dev:renderer": "cd src/renderer && next dev -p 3000",
    "dev:main": "tsup --watch --onSuccess \"electron .\"",
    "typecheck": "tsc -p tsconfig.main.json --noEmit && tsc -p src/renderer/tsconfig.json --noEmit",
    "build": "bun run build:renderer && bun run build:main",
    "build:renderer": "cd src/renderer && next build",
    "build:main": "tsup",
    "dist": "bun run build && electron-builder --mac",
    "postinstall": "electron-builder install-app-deps"
  }
}
```

## Gotchas

- `verbatimModuleSyntax: true` means every type-only import must use `import type`. Worth it — the build fails fast instead of producing silent side-effect imports in main.
- Electron's ESM requires `"type": "module"` in `package.json` AND ESM-compatible preload if the preload is not sandboxed. Sandboxed preload is always CJS regardless.
- Source maps in production — ship them if you run Sentry; otherwise strip to shrink bundle.
