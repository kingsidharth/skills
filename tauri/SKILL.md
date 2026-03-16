---
name: tauri-apps
description: "Build cross-platform desktop and mobile applications with Tauri v2, Rust backend, and web frontends (React/Next.js, Vue, Svelte, etc.). Use this skill whenever the user wants to create, scaffold, develop, debug, or distribute a Tauri application. Also trigger when the user mentions Tauri plugins, Tauri commands, Tauri capabilities/permissions, calling Rust from JavaScript, IPC between frontend and backend, native desktop/mobile app packaging, or building apps with pnpm/bun + Rust. Covers project setup, Next.js SSG integration, plugin installation, security capabilities, code signing, and distribution to macOS App Store, Google Play, Windows Installer, AppImage, and more. Even if the user just says 'build a desktop app' or 'package my web app as native', consider whether Tauri is the right tool and suggest this skill."
---

# Tauri v2 — Application Development Skill

Build cross-platform desktop and mobile applications with Tauri v2. Tauri uses a Rust backend with a web frontend rendered via the OS's native WebView. Unlike Electron, it doesn't bundle Chromium — resulting in binaries as small as 600KB.

Targets: macOS, Windows, Linux, iOS, Android — from a single codebase.

Official docs: https://v2.tauri.app

## When to Read Reference Files

This SKILL.md covers the workflow and key decisions. For detailed API patterns, plugin setup, security configuration, and distribution steps, read the appropriate reference file:

| Scenario | Reference File |
|---|---|
| Setting up a new project with Next.js, Vite, or other frontend | `references/project-setup.md` |
| Developing: Rust commands, IPC, state, icons, debugging | `references/development-guide.md` |
| Adding plugins (HTTP, FS, Upload, Store, etc.) | `references/plugins.md` |
| Security: capabilities, permissions, scopes, CSP | `references/security.md` |
| Building, signing, and distributing the app | `references/distribution.md` |

## Architecture Overview

Tauri uses a multi-process model:

**Core Process (Rust)** — the app's entry point with full OS access. Creates windows, system tray, notifications. Routes all IPC. Manages global state (settings, DB connections). Written in Rust for memory safety and performance.

**WebView Processes** — each window is a separate process using the OS's native WebView engine (Edge WebView2 on Windows, WKWebView on macOS, webkitgtk on Linux). Your HTML/CSS/JS runs here. WebView libraries are dynamically linked at runtime, not bundled — this is why Tauri apps are tiny.

**IPC flows through the Core process.** The frontend calls Rust functions via `invoke()`, and Rust can emit events to the frontend. Two patterns exist:
- **Brownfield** (default) — works with existing frontend projects, no extra config
- **Isolation** — sandboxed iframe intercepts all IPC for security when loading untrusted third-party code

## Project Structure

```
my-app/
├── src/                    # Frontend source (React, Vue, Svelte, etc.)
├── src-tauri/
│   ├── Cargo.toml          # Rust dependencies
│   ├── Cargo.lock          # Commit this
│   ├── tauri.conf.json     # Main Tauri config
│   ├── capabilities/       # Security capability files
│   │   └── default.json
│   ├── src/
│   │   └── lib.rs          # Rust entry point, plugin registration, commands
│   ├── icons/              # App icons (generated via `tauri icon`)
│   └── gen/                # Auto-generated (mobile projects, schemas)
├── package.json
└── pnpm-lock.yaml          # Or bun.lockb, yarn.lock, etc.
```

## Core Workflow

### 1. Scaffold

```bash
pnpm create tauri-app       # Interactive: picks frontend, package manager, template
# Or: bun create tauri-app / cargo create-tauri-app
```

For existing projects, add Tauri manually:
```bash
pnpm add -D @tauri-apps/cli
pnpm tauri init
```

### 2. Configure

Edit `src-tauri/tauri.conf.json`:
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

Platform-specific overrides auto-merge: `tauri.macos.conf.json`, `tauri.linux.conf.json`, `tauri.windows.conf.json`, `tauri.android.conf.json`, `tauri.ios.conf.json`.

### 3. Develop

```bash
pnpm tauri dev              # Desktop with hot-reload
pnpm tauri android dev      # Android emulator/device
pnpm tauri ios dev          # iOS simulator/device
```

Rust code in `src-tauri/` auto-rebuilds on change. Use `.taurignore` (like `.gitignore`) to exclude files from watch.

### 4. Add Capabilities

Every Tauri API access requires an explicit permission. Define in `src-tauri/capabilities/default.json`:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "main-capability",
  "windows": ["main"],
  "permissions": [
    "core:path:default",
    "core:event:default",
    "core:window:default",
    "core:app:default"
  ]
}
```

Read `references/security.md` for platform-specific capabilities, remote API access, and scope configuration.

### 5. Add Plugins

```bash
pnpm tauri add http         # Adds Rust crate + JS package + registers in lib.rs
pnpm tauri add fs
pnpm tauri add store
```

Then add permissions to your capability file. Read `references/plugins.md` for the full plugin catalog and usage patterns.

### 6. Build & Distribute

```bash
pnpm tauri build            # Desktop: produces platform-specific installer
pnpm tauri android build    # Android: APK/AAB
pnpm tauri ios build        # iOS: IPA
```

Most platforms require code signing. Read `references/distribution.md` for per-platform signing, App Store submission, and CI/CD pipeline setup.

## Key Principles

- **Defer logic to Rust** — keep the frontend thin, put business logic and secrets in the Core process
- **Minimum permissions** — only grant what each window needs via capabilities
- **SSG for meta-frameworks** — Next.js must use `output: 'export'`; Tauri doesn't support SSR
- **Commit lockfiles** — both `Cargo.lock` and your JS lockfile (`pnpm-lock.yaml`, etc.)
- **Don't commit** `src-tauri/target/` or `node_modules/`

## App Size Optimization

Minimal Tauri app: ~600KB. Typical real-world: 3–10MB.

```toml
# src-tauri/Cargo.toml — release profile
[profile.release]
codegen-units = 1
lto = true
opt-level = "z"    # "s" for balance, "3" for speed
strip = true
panic = "abort"
```

Frontend: enable tree-shaking, minify JS, disable source maps in production, optimize images (WebP), audit dependencies with Bundlephobia.

## Quick Command Reference

| Command | Purpose |
|---|---|
| `pnpm tauri dev` | Desktop development with hot-reload |
| `pnpm tauri build` | Production build + bundling |
| `pnpm tauri build --no-bundle` | Build without bundling (split step) |
| `pnpm tauri bundle --bundles app,dmg` | Bundle specific formats |
| `pnpm tauri android dev` | Android development |
| `pnpm tauri ios dev` | iOS development |
| `pnpm tauri add <plugin>` | Add a plugin (Rust + JS + registration) |
| `pnpm tauri icon ./icon.png` | Generate all platform icons from source |
| `pnpm tauri info` | Show environment info for debugging |
| `pnpm tauri migrate` | Migrate from v1 to v2 |

Replace `pnpm` with `bun`, `yarn`, `npm`, `deno task`, or `cargo` as needed.
