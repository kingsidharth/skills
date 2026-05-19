# Server, CLI, Headless & LM Link

## CLI (`lms`)

Ships with LM Studio — no separate install. Run `lms --help`.

| Command | Purpose |
|---|---|
| `lms get <model>` | Download a model |
| `lms ls` | List downloaded models |
| `lms ps` | List loaded models |
| `lms load [--gpu=max\|auto\|0-1] [--context-length=N]` | Load model |
| `lms unload [--all]` | Unload model(s) |
| `lms chat` | Interactive terminal chat |
| `lms server start` / `stop` / `status` | Control API server |
| `lms daemon up` / `down` / `status` / `update` | Manage headless daemon |
| `lms link enable` / `disable` / `status` | Manage LM Link |
| `lms log stream` | Tail server logs |

## Headless / daemon mode

Two options:
1. **llmster** (recommended) — standalone daemon, no GUI
2. **Desktop app** — minimize to tray, serve in background

### llmster install

```bash
# Linux / Mac
curl -fsSL https://lmstudio.ai/install.sh | bash
# Windows
irm https://lmstudio.ai/install.ps1 | iex
```

Start: `lms daemon up`

### JIT model loading

When enabled, REST `/v1/models` returns all downloaded models. Inference endpoints auto-load on first request. JIT-loaded models auto-unload after idle timeout.

## Server settings

Configurable via app settings or CLI:
- **Port** — server listen port
- **Require Authentication** — API token via `Authorization` header
- **Serve on Local Network** — bind to LAN IP instead of localhost
- **Allow per-request MCPs** — ephemeral MCP server connections per request
- **Enable CORS** — cross-origin access
- **JIT Model Loading** — load on demand
- **Auto Unload Unused JIT Models** — reclaim memory
- **Only Keep Last JIT Loaded Model** — minimize RAM

## LM Link

End-to-end encrypted remote access to local models across devices, powered by Tailscale.

Use case: run large models on a powerful desktop, use them from a laptop anywhere.

Setup:
1. Open LM Link panel in LM Studio app (above Settings gear)
2. Log in — a link is auto-provisioned
3. Add another device from the LM Link panel

CLI: `lms link enable`, `lms link set-preferred-device <name>`

Works with REST API, CLI, and integrations (Claude Code, Codex, etc.).
