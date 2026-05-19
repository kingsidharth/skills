# Tailscale Serve & Funnel

## Serve vs Funnel

- **Serve**: exposes a local service to your **tailnet only** (private)
- **Funnel**: exposes a local service to the **entire internet** (public)

Both act as a reverse proxy from a Tailscale-managed HTTPS endpoint to a local `127.0.0.1` port.

## Prerequisites

- MagicDNS enabled (on by default for new tailnets)
- HTTPS certificates enabled in admin console → DNS

## Tailscale Serve

### Basic usage

```bash
# Proxy tailnet HTTPS → localhost:4000
tailscale serve 4000

# Run in background (persistent)
tailscale serve --bg 4000

# Explicit forms (equivalent)
tailscale serve localhost:4000
tailscale serve http://127.0.0.1:4000

# Serve on a specific HTTPS port
tailscale serve --https=8443 4000

# Serve at a subpath
tailscale serve --set-path=/api --bg 4000

# Serve over plain HTTP (no TLS termination)
tailscale serve --http=80 4000
```

Result: `https://my-node.tailnet-name.ts.net` → proxies to `http://127.0.0.1:4000`

HTTP servers are also accessible via short MagicDNS names like `http://my-node`.

### Multiple mount points

```bash
tailscale serve --set-path=/ --bg 3000        # frontend
tailscale serve --set-path=/api --bg 4000     # backend API
tailscale serve --set-path=/docs --bg /var/www/docs  # static files
```

### TCP forwarding

```bash
# Forward raw TCP (e.g., PostgreSQL)
tailscale serve --tcp=5432 tcp://localhost:5432

# TLS-terminated TCP
tailscale serve --tls-terminated-tcp=443 tcp://localhost:8080
```

### Status and management

```bash
tailscale serve status          # list active serves
tailscale serve status --json   # machine-readable
tailscale serve off             # turn off all serves
tailscale serve --https=443 off # turn off specific serve
```

### Self-signed backend

```bash
# If your local service uses self-signed HTTPS
tailscale serve https+insecure://localhost:8443
```

## Tailscale Funnel

Same syntax but exposes to the public internet. Only ports 443, 8443, 10000 are allowed.

```bash
tailscale funnel 3000
tailscale funnel --bg 3000
```

Result: `https://my-node.tailnet-name.ts.net` is accessible from the public internet.

### ACL requirement for Funnel

Your policy file must grant Funnel access:

```jsonc
{
  "nodeAttrs": [{
    "target": ["autogroup:member"],
    "attr": ["funnel"]
  }]
}
```

## Serve for Tailscale Services

When using Tailscale Services (TailVIPs), use the `--service` flag:

```bash
tailscale serve --service=100.100.100.50 --bg 4000
```

This binds to the Service's virtual IP rather than the device's own IP.

## Key details

- Only `http://127.0.0.1` is supported as proxy target (not `0.0.0.0` or LAN IPs)
- Tailscale daemon terminates TLS — your backend sees plain HTTP
- HTTPS certs are auto-provisioned via Let's Encrypt
- On macOS App Store version: can proxy ports but cannot serve files/directories (use open-source variant for file serving)
- `--bg` runs as a background process that persists across reboots
- PROXY protocol support (`--proxy-protocol=2`) passes original client IP to backend
