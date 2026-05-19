# Native modules

Native modules (`.node` files) are compiled C/C++/Rust code loadable from Node. Electron has a different ABI than system Node (different V8 version, BoringSSL instead of OpenSSL), so every native module must be rebuilt against Electron's headers.

## The rebuild step

Every install or Electron upgrade:

```bash
bunx @electron/rebuild
```

Or via electron-builder:

```bash
bunx electron-builder install-app-deps
```

Both find native modules in `node_modules`, download Electron's headers, and rebuild. Run on `postinstall` so CI doesn't forget.

```json
{ "scripts": { "postinstall": "electron-builder install-app-deps" } }
```

Symptom of skipping: app crashes on load with `Error: The module … was compiled against a different Node.js version using NODE_MODULE_VERSION X. This version of Node.js requires NODE_MODULE_VERSION Y.`

## Prebuilds

Modules using `prebuild` or `node-gyp-build` ship precompiled binaries for common platforms. If your Electron version + arch matches an available prebuild, no rebuild needed. If not, source build kicks in — requires Xcode CLT on macOS.

For CI, install Xcode Command Line Tools (`xcode-select --install`) on macOS runners. GitHub Actions' `macos-14`/`macos-15` images already have them.

## N-API vs NAN

Prefer N-API modules. N-API is ABI-stable across Node versions, so a single binary works across many Electron versions without recompiling. NAN-based modules (older `nan` package) need rebuild per ABI.

When picking native deps: check if it uses N-API. If not and you have alternatives, switch.

## V8 memory cage (Electron 21+)

Electron enables V8 sandboxed pointers. Implication for native modules: `ArrayBuffer` cannot point to memory allocated outside V8's heap. Modules that pass external buffers to JS will fail with `napi_create_external_buffer` errors.

Affected: older bindings to `libuv`-allocated buffers, raw `mmap` outputs, some FFI patterns. Fixes:

1. Copy external buffer into a V8-allocated buffer (`napi_create_buffer_copy`).
2. Allocate via `napi_create_buffer` and write into the V8-managed memory.

If you maintain the module, fix it. If not, file an issue or pin Electron < 21 (not recommended — security regressions).

## Universal builds (arm64 + x64)

Apple silicon and Intel users need different binaries unless you ship a universal build:

```bash
electron-builder --mac --universal
```

For native modules, `electron-builder` runs the build twice (once per arch) and `lipo`s them together. This requires both arches' prebuilds or a working toolchain for both. On an M-series Mac, building x64 needs Rosetta and the x64 Node toolchain — `electron-builder install-app-deps --arch x64` to test.

Alternatively, ship two DMGs (`MyApp-x64.dmg`, `MyApp-arm64.dmg`). Smaller downloads, no Rosetta complexity, but two artifacts to maintain.

## Common native deps

| Module | Notes |
|---|---|
| `better-sqlite3` | Solid, N-API. Recommended. |
| `sharp` | Image processing. Heavy install (~100 MB). Bundle in `asarUnpack`. |
| `keytar` | Keychain access. Often replaced by `safeStorage` (no native dep). |
| `node-pty` | Pseudoterminal — terminal apps. |
| `bufferutil`, `utf-8-validate` | ws speedups. Optional. |
| `serialport` | Hardware. |
| `node-mac-permissions` | macOS-specific permission queries beyond what Electron exposes. |

## Writing your own (NAPI-RS for Rust)

NAPI-RS is the cleanest Rust → Node binding. Compiles per-target via `napi build --target aarch64-apple-darwin` etc. Ships prebuilds via npm. No node-gyp.

```toml
# Cargo.toml
[package]
name = "myapp-native"
[lib]
crate-type = ["cdylib"]
[dependencies]
napi = { version = "2", default-features = false, features = ["napi9"] }
napi-derive = "2"
```

```rust
use napi_derive::napi;
#[napi]
pub fn fast_hash(input: String) -> u64 { /* … */ }
```

For C++/Rust ad-hoc work, `node-addon-api` (C++) and `napi-rs` (Rust) cover most ground.

## ASAR and native modules

Native `.node` files can't go inside an `asar` archive — they must be unpacked to be loaded. electron-builder handles this automatically for known modules, but custom paths need `asarUnpack`:

```yaml
# electron-builder.yml
asarUnpack:
  - "**/*.node"
  - "node_modules/sharp/**"
```

Bundle size impact: unpacked deps live alongside the asar, sometimes doubling distribution size. Trim aggressively — `electron-builder` only includes deps in `dependencies`, not `devDependencies`.
