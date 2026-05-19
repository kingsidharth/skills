# Project Setup & Structure

## Recommended Structure

```
my-electron-app/
├── forge.config.ts              # Electron Forge config (typed)
├── tsconfig.json
├── package.json
├── src/
│   ├── main/
│   │   ├── index.ts             # Main process entry
│   │   ├── ipc-handlers.ts      # All ipcMain.handle registrations
│   │   ├── windows.ts           # BrowserWindow creation/management
│   │   ├── menu.ts              # Application menu
│   │   ├── updater.ts           # Auto-update logic
│   │   └── native/              # Native module wrappers
│   │       └── storage.ts
│   ├── preload/
│   │   ├── index.ts             # Preload entry — contextBridge only
│   │   └── api.ts               # Type-safe API surface definition
│   ├── renderer/
│   │   ├── index.html
│   │   ├── index.tsx            # React entry
│   │   ├── App.tsx
│   │   ├── pages/               # Route-based code splitting targets
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── stores/              # State management
│   │   └── workers/             # Web Workers
│   │       └── processor.worker.ts
│   ├── shared/
│   │   ├── types.ts             # Shared types (IPC channels, data shapes)
│   │   ├── constants.ts
│   │   └── ipc-channels.ts      # Channel name constants (single source of truth)
│   └── global.d.ts              # Window interface augmentation
├── resources/                   # Icons, platform assets
│   ├── icon.icns
│   ├── icon.ico
│   └── icon.png
├── native/                      # NAPI-RS / Rust addons (optional)
│   ├── Cargo.toml
│   └── src/lib.rs
└── entitlements.plist            # macOS signing entitlements
```

**Key principles:**
- Separate directories per process type (main, preload, renderer)
- Shared types in `shared/` — never import Electron APIs here
- IPC channel names as constants, not string literals scattered across files
- Preload is thin — delegates everything to main via IPC

## Quick Start with Forge + Vite + TypeScript

```bash
npx create-electron-app@latest my-app --template=vite-typescript
cd my-app
```

This scaffolds a working app with Vite bundling, TypeScript, and HMR for the renderer.

### Using Bun for Dependencies

```bash
# Use bun for fast installs, node for runtime
bun install
npm run start   # Forge uses Node.js runtime
```

Add to `package.json`:

```json
{
  "packageManager": "bun@1.x",
  "scripts": {
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make",
    "publish": "electron-forge publish"
  }
}
```

## IPC Channel Contract

Define channels as a single source of truth:

```ts
// src/shared/ipc-channels.ts
export const IPC = {
  GET_VERSION: 'get-version',
  READ_CONFIG: 'read-config',
  SAVE_FILE: 'save-file',
  THEME_CHANGED: 'theme-changed',
} as const;

export type IpcChannel = typeof IPC[keyof typeof IPC];
```

Use in both main and preload — prevents typos.

## TypeScript Config

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "paths": {
      "@shared/*": ["./src/shared/*"]
    }
  },
  "include": ["src/**/*"]
}
```

## Essential Development Dependencies

```json
{
  "devDependencies": {
    "electron": "^33.0.0",
    "@electron-forge/cli": "^7.0.0",
    "@electron-forge/plugin-vite": "^7.0.0",
    "@electron-forge/maker-squirrel": "^7.0.0",
    "@electron-forge/maker-dmg": "^7.0.0",
    "@electron-forge/maker-deb": "^7.0.0",
    "@electron-forge/publisher-github": "^7.0.0",
    "@electron/rebuild": "^3.0.0",
    "electron-devtools-installer": "^3.0.0",
    "typescript": "^5.0.0",
    "vite": "^6.0.0"
  }
}
```

## Secure BrowserWindow Factory

```ts
// src/main/windows.ts
import { BrowserWindow } from 'electron';
import path from 'node:path';

export function createMainWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, '../preload/index.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
      webSecurity: true,
    },
    titleBarStyle: 'hiddenInset', // macOS native title bar
    show: false, // show after ready-to-show to avoid flash
  });

  win.once('ready-to-show', () => win.show());

  // Security: restrict navigation
  win.webContents.on('will-navigate', (event, url) => {
    if (!url.startsWith('app://')) event.preventDefault();
  });

  return win;
}
```

> **Ref:** [Electron Quick Start](https://www.electronjs.org/docs/latest/tutorial/quick-start) · [Forge Getting Started](https://www.electronforge.io/) · [Forge Vite Template](https://www.electronforge.io/templates/vite-+-typescript)
