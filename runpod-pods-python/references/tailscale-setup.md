# Tailscale Inside RunPod Pods

Private networking between RunPod pods and your local/cloud machines via Tailscale.

## Why

- SSH into pods without public IP or port mapping
- Access pod services (Jupyter, APIs) over private Tailscale IP
- Secure pod-to-pod communication across data centers (alternative to RunPod global networking)
- Access home/office resources from pods

## Prerequisites

- Tailscale account with auth key (Settings > Keys > Generate auth key)
- Reusable + ephemeral key recommended for pods (auto-cleanup on disconnect)

## Setup in Docker Entrypoint

Add to your Dockerfile or startup script. Works in RunPod's default Ubuntu-based images.

### Option 1: Startup Script (recommended for custom templates)

```bash
#!/bin/bash
# /workspace/start-tailscale.sh

# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start in userspace networking mode (no TUN device needed in containers)
tailscaled --tun=userspace-networking --state=/workspace/tailscale-state/ &
sleep 2

# Authenticate — uses TAILSCALE_AUTHKEY env var
tailscale up \
  --authkey="${TAILSCALE_AUTHKEY}" \
  --hostname="runpod-${RUNPOD_POD_ID:-unknown}"

echo "Tailscale IP: $(tailscale ip -4)"
```

### Option 2: Inline in Docker CMD

For one-off pods without custom templates:

```python
# In your pod creation env vars:
"env": {
    "TAILSCALE_AUTHKEY": "{{ RUNPOD_SECRET_tailscale_key }}"
}

# In dockerStartCmd:
"dockerStartCmd": [
    "bash", "-c",
    "curl -fsSL https://tailscale.com/install.sh | sh && "
    "tailscaled --tun=userspace-networking --state=/workspace/tailscale-state/ & "
    "sleep 2 && "
    "tailscale up --authkey=$TAILSCALE_AUTHKEY --hostname=runpod-$RUNPOD_POD_ID && "
    "python /app/main.py"
]
```

### Option 3: Bake into Docker Image

```dockerfile
FROM runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04

RUN curl -fsSL https://tailscale.com/install.sh | sh

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/bash
# entrypoint.sh
tailscaled --tun=userspace-networking --state=/workspace/tailscale-state/ &
sleep 2
tailscale up --authkey="${TAILSCALE_AUTHKEY}" --hostname="runpod-${RUNPOD_POD_ID}"
exec "$@"
```

## Key Flags

| Flag | Purpose |
|---|---|
| `--tun=userspace-networking` | Required in containers without /dev/net/tun |
| `--state=/workspace/tailscale-state/` | Persist state across restarts (save to volume) |
| `--authkey` | Auto-authenticate without interactive login |
| `--hostname` | Human-readable name in Tailscale admin |
| `--advertise-exit-node` | Route all traffic through pod (optional) |
| `--accept-routes` | Accept subnet routes from other nodes |

## Storing the Auth Key

Use RunPod Secrets (console > Secrets > Create):
- Secret name: `tailscale_key`
- Secret value: `tskey-auth-xxxxx`

Reference in template env vars: `TAILSCALE_AUTHKEY={{ RUNPOD_SECRET_tailscale_key }}`

## Connecting

Once Tailscale is running on the pod:

```bash
# From your local machine (with Tailscale installed)
ssh root@runpod-<POD_ID>    # uses Tailscale hostname
# or
ssh root@100.x.y.z          # uses Tailscale IP
```

No port mapping needed. No public IP needed.

## Gotchas

- **Userspace networking is slower than kernel mode** — fine for SSH/API, not ideal for bulk data transfer. Use `runpodctl send/receive` or rsync over public IP for large transfers.
- **State directory**: Use `/workspace/tailscale-state/` to survive restarts. Without persistent state, each restart registers a new node.
- **Ephemeral keys**: Recommended. Nodes auto-deregister when disconnected, keeping your Tailscale admin clean.
- **Pod termination**: If using ephemeral keys, node auto-removes. With non-ephemeral keys, manually remove stale nodes from Tailscale admin.
- **DNS**: Tailscale MagicDNS lets you use `runpod-PODID` as hostname directly.
