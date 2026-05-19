# Infrastructure

## Headless / daemon mode (`llmster`)

Run LM Studio without the GUI. Two options:

### Option 1: `llmster` (recommended)

LM Studio core packaged as a standalone daemon. Works on Linux servers, cloud, GPU rigs.

```bash
# Install
curl -fsSL https://lmstudio.ai/install.sh | bash   # Linux/Mac
irm https://lmstudio.ai/install.ps1 | iex           # Windows

# Start
lms daemon up

# Stop
lms daemon down

# Check status
lms daemon status

# Update
lms daemon update
```

### Option 2: Desktop app headless

Enable "Run on login" in app settings → app minimizes to system tray, server continues.

### JIT model loading

When enabled (default for REST endpoints):
- `/v1/models` returns all downloaded models, not just loaded ones
- Inference endpoints auto-load the requested model
- JIT-loaded models auto-unload after idle TTL

## LM Link

Secure, E2E-encrypted remote model access across devices via Tailscale.

Use case: run large models on a powerful desktop, access from a laptop anywhere.

```bash
lms link enable
lms link status
lms link set-device-name "my-gpu-rig"
lms link set-preferred-device "device-name"
lms link disable
```

Works with CLI, REST API, and integrations (Claude Code, Codex, etc.).

## CLI (`lms`)

Ships with LM Studio. MIT licensed.

| Command | Purpose |
|---------|---------|
| `lms chat` | Interactive terminal chat |
| `lms get <model>` | Download a model |
| `lms ls` | List downloaded models |
| `lms ps` | List loaded models |
| `lms load [--gpu=max\|auto\|0-1] [--context-length=N]` | Load model into memory |
| `lms unload [--all]` | Unload model(s) |
| `lms server start/stop/status` | Control the API server |
| `lms daemon up/down/status/update` | Manage headless daemon |
| `lms link enable/disable/status` | Manage LM Link |
| `lms dev` | Plugin development mode |
| `lms push` / `lms clone` | Publish/clone plugins |
| `lms create node-typescript` | Scaffold a new SDK project |

## Project setup

```bash
lms create node-typescript    # interactive scaffold
# or add to existing project:
npm install @lmstudio/sdk
```
