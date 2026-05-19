# Forge: Config & Plugins

Electron Forge is the official build pipeline: package → make → publish.

## Build Lifecycle

```
start → [plugin hooks] → electron .
package → [resolve deps] → [rebuild native modules] → [copy to output] → [asar]
make → [package] → [run makers per platform/arch]
publish → [make] → [run publishers]
```

## Configuration (TypeScript)

As of Forge v7.8.1, `forge.config.ts` loads natively via `jiti`:

```ts
import type { ForgeConfig } from '@electron-forge/shared-types';

const config: ForgeConfig = {
  packagerConfig: {
    asar: true,
    name: 'My App',
    appBundleId: 'com.co.myapp',
    icon: './resources/icon',         // no extension — auto-picks .icns/.ico/.png
    ignore: [/\.map$/, /\.ts$/],
    extraResource: ['./native/lib.node'],
    osxSign: {},
    osxNotarize: { /* ... */ },
    win32metadata: { CompanyName: 'My Company', FileDescription: 'My App' },
  },
  rebuildConfig: {
    force: true,
    onlyModules: ['better-sqlite3'],
  },
  makers: [/* see makers.md */],
  publishers: [/* see makers.md */],
  plugins: [/* see below */],
  hooks: {/* see below */},
};
export default config;
```

## Plugin System

Plugins hook into any lifecycle point:

```ts
import { VitePlugin } from '@electron-forge/plugin-vite';
import { FusesPlugin } from '@electron-forge/plugin-fuses';
import { FuseV1Options } from '@electron/fuses';

plugins: [
  new VitePlugin({
    build: [
      { entry: 'src/main.ts', config: 'vite.main.config.ts' },
      { entry: 'src/preload.ts', config: 'vite.preload.config.ts' },
    ],
    renderer: [{ name: 'main_window', config: 'vite.renderer.config.ts' }],
  }),

  new FusesPlugin({
    [FuseV1Options.RunAsNode]: false,
    [FuseV1Options.EnableCookieEncryption]: true,
    [FuseV1Options.EnableNodeOptionsEnvironmentVariable]: false,
    [FuseV1Options.EnableNodeCliInspectArguments]: false,
  }),
],
```

**Available plugins:** `plugin-vite` (recommended), `plugin-webpack`, `plugin-electronegativity` (security scan), `plugin-fuses`, `plugin-auto-unpack-natives`.

## Hooks

```ts
hooks: {
  generateAssets: async () => { /* generate icons, compile native code */ },
  prePackage: async (config, platform, arch) => { /* pre-build */ },
  postMake: async (config, makeResults) => { return makeResults; },
  readPackageJson: async (config, pkg) => { delete pkg.devDependencies; return pkg; },
},
```

## Build Identifiers

```ts
import { utils } from '@electron-forge/core';

buildIdentifier: process.env.IS_BETA ? 'beta' : 'production',
packagerConfig: {
  appBundleId: utils.fromBuildIdentifier({
    beta: 'com.co.myapp.beta',
    production: 'com.co.myapp',
  }),
},
```

## CLI

```bash
npx electron-forge start                    # dev with HMR
npx electron-forge start -- --inspect       # dev with debugger
npx electron-forge package                  # package only
npx electron-forge package --arch=arm64     # specific arch
npx electron-forge make                     # package + make distributables
npx electron-forge make --platform=win32    # specific platform
npx electron-forge publish                  # make + publish
```

> **Ref:** [Forge Config](https://www.electronforge.io/config/configuration) · [Forge TS Config](https://www.electronforge.io/config/typescript-configuration) · [Forge Plugins](https://www.electronforge.io/config/plugins)
