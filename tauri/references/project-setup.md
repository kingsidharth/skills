# Project Setup Reference

## Table of Contents
- Scaffolding a New Project
- Package Manager Selection
- Next.js Configuration
- Vite Configuration
- Mobile Dev Server Setup
- Monorepo Patterns
- Configuration File Deep Dive

---

## Scaffolding a New Project

The `create-tauri-app` tool supports all major package managers and frontend frameworks:

```bash
# pnpm (recommended for most projects)
pnpm create tauri-app

# bun (fastest, macOS/Linux only)
bun create tauri-app

# npm
npm create tauri-app@latest

# yarn
yarn create tauri-app

# cargo (no Node.js needed)
cargo install create-tauri-app --locked
cargo create-tauri-app
```

The interactive prompts ask for:
1. **Project name** and **bundle identifier** (e.g., `com.myapp.app`)
2. **Frontend language**: TypeScript / JavaScript or Rust (Leptos, Yew, etc.)
3. **Package manager**: pnpm, yarn, npm, bun
4. **UI template**: Vanilla, React, Vue, Svelte, Solid, Angular, Preact
5. **Flavor**: TypeScript or JavaScript

After scaffolding:
```bash
cd my-app
pnpm install
pnpm tauri dev
```

The first `tauri dev` run compiles all Rust dependencies — this can take several minutes. Subsequent builds only recompile your code.

### Adding Tauri to an Existing Project

```bash
# Install the Tauri CLI as a dev dependency
pnpm add -D @tauri-apps/cli

# Initialize Tauri in your project
pnpm tauri init
```

This creates the `src-tauri/` directory. You'll need to answer:
- **App name** and **window title**
- **Web assets path** relative to `src-tauri/tauri.conf.json` (e.g., `../out` for Next.js, `../dist` for Vite)
- **Dev server URL** (e.g., `http://localhost:3000` for Next.js, `http://localhost:5173` for Vite)
- **Dev command** (e.g., `pnpm dev`)
- **Build command** (e.g., `pnpm build`)

---

## Package Manager Selection

| Manager | Speed | Windows | Notes |
|---|---|---|---|
| **pnpm** | Fast | Yes | Content-addressable storage, strict dependency isolation. Best all-rounder. |
| **bun** | Fastest | No* | Uses its own JS runtime (not Node.js). macOS + Linux only. |
| **yarn** | Medium | Yes | Classic choice, good monorepo support (workspaces). |
| **npm** | Baseline | Yes | Ships with Node.js, no extra install needed. |

*Bun Windows workaround: Build frontend on Linux/macOS runner in CI, transfer artifacts to Windows runner for Tauri build step.

All Tauri CLI commands work identically across package managers — just swap the prefix:

| pnpm | bun | npm | yarn | cargo |
|---|---|---|---|---|
| `pnpm tauri dev` | `bun tauri dev` | `npm run tauri dev` | `yarn tauri dev` | `cargo tauri dev` |
| `pnpm tauri build` | `bun tauri build` | `npm run tauri build` | `yarn tauri build` | `cargo tauri build` |
| `pnpm tauri add fs` | `bun tauri add fs` | `npm run tauri add fs` | `yarn tauri add fs` | `cargo tauri add fs` |

---

## Next.js Configuration

Tauri requires **static site generation (SSG)**. Next.js must use `output: 'export'` — server-side rendering is not supported since Tauri acts as a static file host, not a Node.js server.

### next.config.mjs

```javascript
const isProd = process.env.NODE_ENV === 'production';
const internalHost = process.env.TAURI_DEV_HOST || 'localhost';

/** @type {import('next').NextConfig} */
const nextConfig = {
  // SSG mode — required for Tauri
  output: 'export',

  // Disable server-side image optimization (requires SSR)
  images: { unoptimized: true },

  // Asset prefix for dev server (needed for hot-reload to work)
  assetPrefix: isProd ? undefined : `http://${internalHost}:3000`,
};

export default nextConfig;
```

### tauri.conf.json

```json
{
  "build": {
    "beforeDevCommand": "pnpm dev",
    "beforeBuildCommand": "pnpm build",
    "devUrl": "http://localhost:3000",
    "frontendDist": "../out"
  }
}
```

### Key Constraints

- `output: 'export'` is mandatory — SSR, API routes, middleware are not available
- `images.unoptimized: true` is required since `next/image` optimization needs a server
- The `out/` directory (Next.js static export output) is the `frontendDist`
- `assetPrefix` must point to the dev server URL in development for HMR to work
- `TAURI_DEV_HOST` environment variable is provided by Tauri CLI for mobile development

### Calling Tauri from Next.js

```typescript
'use client';
import { useState, useEffect } from 'react';
import { invoke } from '@tauri-apps/api/core';

export default function Greet() {
  const [greeting, setGreeting] = useState('');

  useEffect(() => {
    invoke('greet', { name: 'World' })
      .then((result) => setGreeting(result as string))
      .catch(console.error);
  }, []);

  return <div>{greeting}</div>;
}
```

Tauri APIs are only available in the browser context (WebView), not during SSG build. Always use `'use client'` directives and guard Tauri imports behind `typeof window !== 'undefined'` checks if needed.

---

## Vite Configuration

Vite is the recommended bundler for SPA frameworks (React, Vue, Svelte, Solid). It works with Tauri out of the box.

### vite.config.ts (with mobile support)

```typescript
import { defineConfig } from 'vite';

const host = process.env.TAURI_DEV_HOST;

export default defineConfig({
  clearScreen: false,
  server: {
    host: host || false,
    port: 1420,
    strictPort: true,
    hmr: host
      ? { protocol: 'ws', host, port: 1421 }
      : undefined,
  },
});
```

### tauri.conf.json for Vite

```json
{
  "build": {
    "beforeDevCommand": "pnpm dev",
    "beforeBuildCommand": "pnpm build",
    "devUrl": "http://localhost:1420",
    "frontendDist": "../dist"
  }
}
```

---

## Mobile Dev Server Setup

For mobile development (especially physical iOS devices), the dev server must listen on the device-accessible address provided by `TAURI_DEV_HOST`.

The Vite config above handles this automatically. For Next.js, the `assetPrefix` config handles it.

### iOS Physical Device

Two approaches:
1. **Default** — Tauri exposes the dev server on the local network (triggers iOS network permission prompt on first run)
2. **Device TUN address** — More secure, requires Xcode. Connect device in Xcode → Window → Devices and Simulators, then run `pnpm tauri ios dev --force-ip-prompt` and select the IPv6 address ending with `::2`

### Android

Works with emulators automatically. For physical devices: enable Developer Mode (Settings → About → tap Build Number 7 times), then enable USB Debugging in Developer Options.

---

## Monorepo Patterns

For projects sharing code between web (Next.js) and native (Tauri) apps, TurboRepo is a popular choice:

```
my-monorepo/
├── apps/
│   ├── web/           # Next.js app for web deployment
│   └── native/        # Tauri app for desktop/mobile
├── packages/
│   ├── ui/            # Shared React components (Tailwind, Shadcn, etc.)
│   ├── typescript-config/
│   └── eslint-config/
├── turbo.json
└── package.json       # Workspace root
```

Key commands in the monorepo root:
```bash
pnpm dev                    # Start all dev servers
pnpm tauri dev              # Start Tauri desktop app
pnpm tauri android dev      # Start Tauri Android app
pnpm tauri ios dev          # Start Tauri iOS app
```

The `packages/ui` directory contains shared components that render identically across web and native platforms.

---

## Configuration File Deep Dive

### tauri.conf.json

The main configuration file. Supports JSON (default), JSON5 (with `config-json5` feature), or TOML (with `config-toml` feature).

Key sections:
- `build` — dev/build commands, dev server URL, frontend dist path
- `bundle` — app name, identifier, icons, targets, signing config
- `app` — windows, security (capabilities), plugins
- `version` — app version (preferred over Cargo.toml's version)

### Runtime Config Extensions

Use `--config` flag to merge additional config via JSON Merge Patch (RFC 7396):

```bash
# Different build flavor
pnpm tauri build --config src-tauri/tauri.beta.conf.json

# Inline override
pnpm tauri build --config '{"bundle":{"identifier":"com.myapp.beta"}}'
```

### Platform-Specific Configs

These are auto-merged with the base config:
- `tauri.macos.conf.json`
- `tauri.linux.conf.json`
- `tauri.windows.conf.json`
- `tauri.android.conf.json`
- `tauri.ios.conf.json`
