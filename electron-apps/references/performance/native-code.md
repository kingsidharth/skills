# Native Code, WebAssembly & Non-JS Performance

When JavaScript is the bottleneck, Electron lets you pair web tech with any native code.

## When to Reach for Non-JS

- CPU-bound computation (hashing, encryption, compression, image processing)
- Operations where deterministic performance matters (no JIT warmup variance)
- Latency-critical hot paths where V8 optimization isn't sufficient

## WebAssembly

WASM executables are ahead-of-time optimized — no JIT warmup, deterministic performance.

**Real-world usage:**
- Notion: SQLite compiled to WASM for document storage/queries
- Figma: entire rendering engine in WASM
- Signal: cryptography library in WASM
- 1Password: encryption via native Node modules

```ts
// Load WASM module
const wasmModule = await WebAssembly.instantiateStreaming(
  fetch('./processor.wasm'),
  importObject
);
const { processData } = wasmModule.instance.exports;
```

For Rust → WASM, use `wasm-bindgen` or `wasm-pack`:

```bash
wasm-pack build --target web
```

## Native Node Addons with Rust (NAPI-RS)

NAPI-RS bridges Rust code into Node.js as native addons. Case study: CRC32 calculation dropped from 800ms to 75ms per file (10x improvement) using `crc32fast` crate through NAPI-RS.

```rust
// src/lib.rs
use napi_derive::napi;
use crc32fast::Hasher;
use std::fs::File;
use std::io::Read;

#[napi]
pub fn calculate_crc32(path: String) -> u32 {
    let mut file = File::open(&path).unwrap();
    let mut hasher = Hasher::new();
    let mut buffer = [0u8; 8192];
    loop {
        let n = file.read(&mut buffer).unwrap();
        if n == 0 { break; }
        hasher.update(&buffer[..n]);
    }
    hasher.finalize()
}
```

Multi-file with threading (Rust OS threads, not Node event loop):

```rust
#[napi]
pub fn calculate_crc32_batch(paths: Vec<String>) -> Vec<u32> {
    paths.into_par_iter() // rayon parallel iterator
        .map(|p| calculate_crc32(p))
        .collect()
}
```

Result: 10 × 200MB files processed in 150ms (vs 16,700ms in pure JS).

**ASAR note:** native `.node` modules must be unpacked from the ASAR archive. Most bundlers (Forge, electron-builder) handle this, but verify with `asarUnpack` config if your native module fails to load.

## Go Bindings

Use `cgo` to build a shared library, then load via `ffi-napi` or build a Node addon:

```bash
# Build Go shared library
CGO_ENABLED=1 go build -buildmode=c-shared -o libprocessor.so ./cmd/processor
```

Load in Node via `ffi-napi`:
```ts
import ffi from 'ffi-napi';
const lib = ffi.Library('./libprocessor', {
  'ProcessData': ['int', ['string']],
});
```

## Python Integration

For ML inference or data processing, spawn Python as a sidecar:

```ts
import { utilityProcess } from 'electron';

// Or use child_process for Python
const python = spawn('python3', ['scripts/inference.py'], {
  stdio: ['pipe', 'pipe', 'pipe'],
});
python.stdin.write(JSON.stringify(input));
python.stdout.on('data', (data) => { /* parse result */ });
```

For macOS ML: use Swift sidecars with MLX (see [native/macos.md](../native/macos.md)).

## Performance Comparison Summary

| Approach | Overhead | Speed | Complexity |
|----------|----------|-------|------------|
| Pure JS | None | Baseline | Low |
| Worker Threads | ~10MB per thread | Same as JS (parallel) | Medium |
| WebAssembly | Module load time | 2-10x JS | Medium |
| NAPI-RS (Rust) | Build toolchain | 10-100x JS | High |
| Node-API (C/C++) | Build toolchain | 10-100x JS | Very High |
| Sidecar process | Process spawn | Varies | Medium |

> **Ref:** [Brainhub NAPI-RS Guide](https://brainhub.eu/library/electron-app-performance) · [NAPI-RS Docs](https://napi.rs/) · [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen)
