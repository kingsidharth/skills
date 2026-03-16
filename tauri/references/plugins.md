# Plugins Reference

## Table of Contents
- Plugin Setup Pattern
- Official Plugin Catalog
- HTTP Client — Detailed Usage
- File System — Detailed Usage
- Upload — Detailed Usage
- Store — Detailed Usage
- Updater — Detailed Usage
- Community / Third-Party Plugins

---

## Plugin Setup Pattern

All official Tauri plugins follow a consistent setup pattern. The `tauri add` command handles all steps automatically:

```bash
pnpm tauri add <plugin-name>
```

This performs three operations:
1. Adds the Rust crate to `src-tauri/Cargo.toml`
2. Registers the plugin in `src-tauri/src/lib.rs`
3. Installs the JavaScript package (if applicable)

### Manual Setup (if needed)

```bash
# 1. Rust crate
cd src-tauri
cargo add tauri-plugin-<name>

# 2. JavaScript package
pnpm add @tauri-apps/plugin-<name>
```

```rust
// 3. Register in src-tauri/src/lib.rs
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_<name>::init())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Permissions

Every plugin operation requires an explicit permission in your capability file:

```json
// src-tauri/capabilities/default.json
{
  "permissions": [
    "<plugin>:default",
    "<plugin>:allow-<specific-command>"
  ]
}
```

---

## Official Plugin Catalog

### Desktop + Mobile (All Platforms)

| Plugin | Add Command | JS Package | Primary Use |
|---|---|---|---|
| File System | `tauri add fs` | `@tauri-apps/plugin-fs` | Read/write files, directories, watch changes |
| HTTP Client | `tauri add http` | `@tauri-apps/plugin-http` | HTTP requests (re-exports `reqwest` in Rust) |
| Upload | `tauri add upload` | `@tauri-apps/plugin-upload` | File upload/download with progress |
| Dialog | `tauri add dialog` | `@tauri-apps/plugin-dialog` | Native file open/save, message boxes |
| Clipboard | `tauri add clipboard` | `@tauri-apps/plugin-clipboard` | System clipboard read/write |
| Notifications | `tauri add notification` | `@tauri-apps/plugin-notification` | OS native notifications |
| SQL | `tauri add sql` | `@tauri-apps/plugin-sql` | SQLite, MySQL, PostgreSQL |
| Store | `tauri add store` | `@tauri-apps/plugin-store` | Persistent key-value store |
| Logging | `tauri add logging` | `@tauri-apps/plugin-logging` | Structured logging to file/console |
| OS Info | `tauri add os` | `@tauri-apps/plugin-os` | Platform, arch, hostname, locale |
| Deep Linking | `tauri add deep-linking` | `@tauri-apps/plugin-deep-linking` | Custom URL scheme handling |
| Websocket | `tauri add websocket` | `@tauri-apps/plugin-websocket` | WebSocket client |
| Opener | `tauri add opener` | `@tauri-apps/plugin-opener` | Open URLs/files with default apps |
| Process | `tauri add process` | `@tauri-apps/plugin-process` | Exit/restart the app |

### Desktop Only

| Plugin | Add Command | JS Package | Primary Use |
|---|---|---|---|
| Autostart | `tauri add autostart` | `@tauri-apps/plugin-autostart` | Launch on system login |
| CLI | `tauri add cli` | `@tauri-apps/plugin-cli` | Parse command-line arguments |
| Global Shortcut | `tauri add global-shortcut` | `@tauri-apps/plugin-global-shortcut` | Global keyboard shortcuts |
| Shell | `tauri add shell` | `@tauri-apps/plugin-shell` | Execute shell commands |
| Single Instance | `tauri add single-instance` | `@tauri-apps/plugin-single-instance` | Prevent multiple app instances |
| Updater | `tauri add updater` | `@tauri-apps/plugin-updater` | In-app auto-update |
| Window State | `tauri add window-state` | `@tauri-apps/plugin-window-state` | Persist/restore window size+position |
| Positioner | `tauri add positioner` | `@tauri-apps/plugin-positioner` | Position relative to tray, screen |
| Stronghold | `tauri add stronghold` | `@tauri-apps/plugin-stronghold` | Encrypted secrets vault |
| Persisted Scope | `tauri add persisted-scope` | `@tauri-apps/plugin-persisted-scope` | Persist file access scope |
| Localhost | `tauri add localhost` | `@tauri-apps/plugin-localhost` | Serve frontend on localhost port |

### Mobile Only

| Plugin | Add Command | JS Package | Primary Use |
|---|---|---|---|
| NFC | `tauri add nfc` | `@tauri-apps/plugin-nfc` | Near Field Communication |
| Barcode Scanner | `tauri add barcode-scanner` | `@tauri-apps/plugin-barcode-scanner` | Camera barcode scanning |
| Biometric | `tauri add biometric` | `@tauri-apps/plugin-biometric` | Fingerprint/face auth |
| Geolocation | `tauri add geolocation` | `@tauri-apps/plugin-geolocation` | GPS/location services |
| Haptics | `tauri add haptics` | `@tauri-apps/plugin-haptics` | Haptic feedback |

---

## HTTP Client — Detailed Usage

The HTTP plugin provides a `fetch` API in JavaScript that mirrors the Web Fetch API, and re-exports `reqwest` in Rust.

### JavaScript

```typescript
import { fetch } from '@tauri-apps/plugin-http';

// GET request
const response = await fetch('https://api.example.com/data', {
  method: 'GET',
});
console.log(response.status);      // 200
console.log(response.statusText);  // "OK"
const data = await response.json();

// POST request
const postResponse = await fetch('https://api.example.com/submit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' }),
});
```

### Rust

```rust
use tauri_plugin_http::reqwest;

let res = reqwest::get("https://api.example.com/data").await?;
println!("{}", res.status());
let body = res.text().await?;
```

### Permissions

Configure allowed URLs in capabilities:

```json
{
  "permissions": [
    {
      "identifier": "http:default",
      "allow": [{ "url": "https://api.example.com/*" }],
      "deny": [{ "url": "https://private.example.com/*" }]
    }
  ]
}
```

### Forbidden Headers

By default, forbidden request headers (per the Fetch spec) are ignored. To use them:

```toml
# src-tauri/Cargo.toml
[dependencies]
tauri-plugin-http = { version = "2", features = ["unsafe-headers"] }
```

---

## File System — Detailed Usage

Access the file system from JavaScript. In Rust, use `std::fs` or `tokio::fs` directly.

### JavaScript

```typescript
import {
  exists, readTextFile, writeTextFile, readDir,
  mkdir, remove, rename, copyFile,
  BaseDirectory
} from '@tauri-apps/plugin-fs';

// Check if file exists
const fileExists = await exists('config.json', { baseDir: BaseDirectory.AppData });

// Read
const content = await readTextFile('config.json', { baseDir: BaseDirectory.AppData });

// Write
await writeTextFile('config.json', JSON.stringify(data), {
  baseDir: BaseDirectory.AppData,
});

// List directory
const entries = await readDir('documents', { baseDir: BaseDirectory.Home });

// Create directory
await mkdir('my-app/data', { baseDir: BaseDirectory.AppData, recursive: true });

// Watch for changes
import { watch } from '@tauri-apps/plugin-fs';
const stopWatching = await watch('data/', (event) => {
  console.log('File changed:', event);
}, { baseDir: BaseDirectory.AppData });
```

### Rust (use `FsExt` for scope management)

```rust
use tauri_plugin_fs::FsExt;

app.fs_scope().allow_directory("/path/to/dir", false);
```

### Security

- Path traversal is prevented — `../` is not allowed
- Access is restricted by scope configuration
- On Android/iOS, access defaults to the application folder only

### Android Extra Setup

Add to `gen/android/app/src/main/AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

### iOS Extra Setup

Create `src-tauri/gen/apple/PrivacyInfo.xcprivacy` with the `NSPrivacyAccessedAPICategoryFileTimestamp` key and `C617.1` reason.

---

## Upload — Detailed Usage

Upload files to remote servers over HTTP with progress tracking. Also supports downloads.

### JavaScript

```typescript
import { upload, download } from '@tauri-apps/plugin-upload';

// Upload with progress
await upload('https://example.com/upload', '/path/to/file', (progress, total) => {
  console.log(`Uploaded ${progress} of ${total} bytes`);
});

// Download with progress
await download('https://example.com/file.zip', '/path/to/save', (progress, total) => {
  console.log(`Downloaded ${progress} of ${total} bytes`);
});
```

---

## Store — Detailed Usage

Persistent key-value store — like localStorage but native and cross-platform.

### JavaScript

```typescript
import { Store } from '@tauri-apps/plugin-store';

const store = await Store.load('settings.json');

// Set
await store.set('theme', 'dark');
await store.set('user', { name: 'Alice', id: 42 });

// Get
const theme = await store.get('theme');

// Save to disk (auto-saves on app close, but can be manual)
await store.save();

// Listen for changes
await store.onKeyChange('theme', (value) => {
  console.log('Theme changed to:', value);
});
```

---

## Updater — Detailed Usage

In-app auto-update mechanism for desktop apps. Generates signed update bundles.

### Setup

```bash
pnpm tauri add updater
```

```rust
// Desktop-only registration
.setup(|app| {
    #[cfg(desktop)]
    app.handle().plugin(tauri_plugin_updater::Builder::new().build());
    Ok(())
})
```

### Configuration

```json
// tauri.conf.json
{
  "bundle": {
    "createUpdaterArtifacts": "v1Compatible"
  },
  "plugins": {
    "updater": {
      "pubkey": "YOUR_PUBLIC_KEY",
      "endpoints": [
        "https://releases.myapp.com/{{target}}-{{arch}}/{{current_version}}"
      ]
    }
  }
}
```

Dynamic URL variables: `{{current_version}}`, `{{target}}` (linux/windows/darwin), `{{arch}}` (x86_64/i686/aarch64/armv7).

### Generate Signing Keys

```bash
pnpm tauri signer generate -w ~/.tauri/myapp.key
```

Set the environment variable `TAURI_SIGNING_PRIVATE_KEY` to the generated key for builds.

Full reference: https://v2.tauri.app/plugin/updater/

---

## Community / Third-Party Plugins

| Plugin | Repository | Description |
|---|---|---|
| iOS Photos | https://github.com/Gbyte-Group/tauri-plugin-ios-photos | Access iOS photo library |
| macOS Menubar App | https://github.com/ahkohd/tauri-macos-menubar-app-example | Menubar/tray-only macOS app (v2 branch, 362 stars) |
| MCP Server | https://github.com/hypothesi/mcp-server-tauri | Model Context Protocol server for AI-assisted Tauri development |

### Writing Custom Plugins

Tauri supports developing your own plugins with Rust backend logic and JavaScript bindings. For mobile plugins, you can include native Swift (iOS) and Kotlin (Android) code.

Full reference: https://v2.tauri.app/develop/plugins/ and https://v2.tauri.app/develop/plugins/develop-mobile/
