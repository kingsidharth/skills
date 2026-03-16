# Development Guide Reference

## Table of Contents
- Calling Rust from the Frontend
- Calling the Frontend from Rust
- State Management
- Embedding Additional Files (Resources)
- Embedding External Binaries (Sidecar)
- App Icons
- Debugging
- Updating Dependencies
- Source Control
- Window Customization
- Window Menu
- System Tray
- Splashscreen
- Node.js as a Sidecar

---

## Calling Rust from the Frontend

Define Rust functions as Tauri commands with the `#[tauri::command]` attribute, then register them in the invoke handler.

### Basic Command

```rust
// src-tauri/src/lib.rs

#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Calling from JavaScript

```typescript
import { invoke } from '@tauri-apps/api/core';

// Invoke the Rust command
const result = await invoke('greet', { name: 'World' });
console.log(result); // "Hello, World!"
```

**Parameter naming:** Rust uses snake_case (`user_name`), JavaScript uses camelCase (`userName`). Tauri handles the conversion automatically.

### Async Commands

```rust
#[tauri::command]
async fn fetch_data(url: String) -> Result<String, String> {
    // Async operations run on a separate thread, won't block the main thread
    reqwest::get(&url)
        .await
        .map_err(|e| e.to_string())?
        .text()
        .await
        .map_err(|e| e.to_string())
}
```

### Error Handling

Commands can return `Result<T, E>` where `E: ToString`. Errors are propagated to JavaScript as rejected promises.

```rust
#[tauri::command]
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}
```

```typescript
try {
  const result = await invoke('divide', { a: 10, b: 0 });
} catch (error) {
  console.error(error); // "Division by zero"
}
```

### Accessing the App Handle

```rust
#[tauri::command]
fn get_app_version(app: tauri::AppHandle) -> String {
    app.package_info().version.to_string()
}
```

### Accessing the Window

```rust
#[tauri::command]
fn set_title(window: tauri::Window, title: String) {
    window.set_title(&title).unwrap();
}
```

Full reference: https://v2.tauri.app/develop/calling-rust/

---

## Calling the Frontend from Rust

Use Tauri's event system to send data from Rust to the frontend.

### Emit to All Windows

```rust
use tauri::Emitter;

app.emit("backend-event", serde_json::json!({ "message": "Hello from Rust!" }))?;
```

### Emit to a Specific Window

```rust
use tauri::Emitter;

let window = app.get_webview_window("main").unwrap();
window.emit("update", payload)?;
```

### Listen in JavaScript

```typescript
import { listen } from '@tauri-apps/api/event';

const unlisten = await listen('backend-event', (event) => {
  console.log(event.payload); // { message: "Hello from Rust!" }
});

// Stop listening when done
unlisten();
```

Full reference: https://v2.tauri.app/develop/calling-frontend/

---

## State Management

Use `tauri::Builder::manage()` to share state across commands. State is accessible via `tauri::State<T>` in command parameters.

### Immutable State

```rust
struct Config {
    api_url: String,
}

#[tauri::command]
fn get_api_url(config: tauri::State<Config>) -> String {
    config.api_url.clone()
}

fn run() {
    tauri::Builder::default()
        .manage(Config { api_url: "https://api.example.com".into() })
        .invoke_handler(tauri::generate_handler![get_api_url])
        .run(tauri::generate_context!())
        .unwrap();
}
```

### Mutable State

Wrap mutable data in `Mutex` or `RwLock`:

```rust
use std::sync::Mutex;

struct Counter(Mutex<i32>);

#[tauri::command]
fn increment(counter: tauri::State<Counter>) -> i32 {
    let mut val = counter.0.lock().unwrap();
    *val += 1;
    *val
}

#[tauri::command]
fn get_count(counter: tauri::State<Counter>) -> i32 {
    *counter.0.lock().unwrap()
}

fn run() {
    tauri::Builder::default()
        .manage(Counter(Mutex::new(0)))
        .invoke_handler(tauri::generate_handler![increment, get_count])
        .run(tauri::generate_context!())
        .unwrap();
}
```

Full reference: https://v2.tauri.app/develop/state-management/

---

## Embedding Additional Files (Resources)

Bundle files with your app that are accessible at runtime.

### Configuration

```json
// tauri.conf.json
{
  "bundle": {
    "resources": [
      "data/config.json",
      "assets/**/*"
    ]
  }
}
```

### Access at Runtime (Rust)

```rust
use tauri::Manager;

let resource_path = app.path().resolve("data/config.json", tauri::path::BaseDirectory::Resource)?;
let content = std::fs::read_to_string(resource_path)?;
```

### Access at Runtime (JavaScript)

```typescript
import { resolveResource } from '@tauri-apps/api/path';
import { readTextFile } from '@tauri-apps/plugin-fs';

const resourcePath = await resolveResource('data/config.json');
const content = await readTextFile(resourcePath);
```

Full reference: https://v2.tauri.app/develop/resources/

---

## Embedding External Binaries (Sidecar)

Ship external executables alongside your Tauri app.

### Configuration

```json
// tauri.conf.json
{
  "bundle": {
    "externalBin": [
      "binaries/my-tool"
    ]
  }
}
```

The binary must be named with the target triple suffix: `my-tool-x86_64-unknown-linux-gnu`, `my-tool-aarch64-apple-darwin`, etc. Tauri resolves the correct binary at runtime.

### Spawn from Rust

```rust
use tauri_plugin_shell::ShellExt;

let sidecar = app.shell().sidecar("my-tool").unwrap();
let (mut rx, _child) = sidecar.spawn().unwrap();
```

Full reference: https://v2.tauri.app/develop/sidecar/

---

## App Icons

Generate all required platform icons from a single source image (minimum 1024×1024 PNG):

```bash
pnpm tauri icon ./app-icon.png
```

This generates icons for macOS (`.icns`), Windows (`.ico`), Linux (`.png` at various sizes), iOS, and Android in `src-tauri/icons/`.

Full reference: https://v2.tauri.app/develop/icons/

---

## Debugging

### Desktop WebView DevTools

- **Windows/Linux:** Right-click → "Inspect" or `Ctrl + Shift + I`
- **macOS:** `Cmd + Option + I`

### iOS Debugging

1. In Safari: Settings → Advanced → "Show features for web developers"
2. For physical devices: enable Web Inspector in Settings → Safari → Advanced
3. Open Safari → Develop menu → select your device → click localhost

### Android Debugging

1. Enable Developer Mode on device (Settings → About → tap Build Number 7 times)
2. Enable USB Debugging in Developer Options
3. In Chrome on your computer: navigate to `chrome://inspect`
4. Your device/emulator appears in remote devices list → click "inspect"

### IDE Debugging (Rust)

- **VS Code:** https://v2.tauri.app/develop/debug/vscode/
- **JetBrains IDEs:** https://v2.tauri.app/develop/debug/rustrover/
- **Neovim:** https://v2.tauri.app/develop/debug/neovim/

### CrabNebula DevTools

A dedicated debugging tool for Tauri apps: https://v2.tauri.app/develop/debug/crabnebula-devtools/

### Hot-Reload Control

Tauri watches `src-tauri/` and dependent crates for changes. Control with:
- `--no-watch` flag to disable
- `.taurignore` file (same syntax as `.gitignore`) in `src-tauri/` or workspace root

```
# .taurignore example
build/
src/generated/*.rs
deny.toml
```

---

## Updating Dependencies

**Rust:** Edit versions in `src-tauri/Cargo.toml`, then `cargo update` in `src-tauri/`.

**JavaScript:** Use your package manager's update command (`pnpm update`, `bun update`, etc.).

Full reference: https://v2.tauri.app/develop/updating-dependencies/

---

## Source Control

**Commit:** `Cargo.lock`, `pnpm-lock.yaml` (or equivalent), `src-tauri/capabilities/`, `src-tauri/icons/`

**Don't commit:** `src-tauri/target/`, `node_modules/`, `src-tauri/gen/` (auto-generated)

---

## Window Customization

Control window decorations, transparency, size, position, always-on-top, and more via `tauri.conf.json` or the Rust/JS API.

Full reference: https://v2.tauri.app/learn/window-customization/

## Window Menu

Create native application menus (File, Edit, View, etc.) with keyboard shortcuts.

Full reference: https://v2.tauri.app/learn/window-menu/

## System Tray

Create system tray icons with context menus. Desktop platforms only.

Full reference: https://v2.tauri.app/learn/system-tray/

## Splashscreen

Show a loading screen while the Rust backend initializes. Implemented as a separate window that closes once the main window is ready.

Full reference: https://v2.tauri.app/learn/splashscreen/

## Node.js as a Sidecar

Ship a Node.js binary alongside your Tauri app for leveraging existing Node.js libraries.

Full reference: https://v2.tauri.app/learn/sidecar-nodejs/
