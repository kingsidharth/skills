# Multi-Domain Binding

When running a backend that must accept requests from localhost (same-machine frontend), Tailscale IPs (`100.x.y.z`), and MagicDNS domains (`host.tailnet.ts.net`), the app needs to handle multiple origins.

## Bind to `0.0.0.0` or `127.0.0.1`?

**Bind to `127.0.0.1:PORT`** (recommended with Tailscale Serve):

- Your app only listens on localhost
- Tailscale Serve acts as a reverse proxy, forwarding `https://hostname.ts.net` → `http://127.0.0.1:PORT`
- Frontend on the same machine connects directly to `localhost:PORT`
- No ports exposed on public interfaces

```bash
# App listens on 127.0.0.1:4000
node server.js --host 127.0.0.1 --port 4000

# Tailscale exposes it to the tailnet
tailscale serve --bg 4000
```

**Bind to `0.0.0.0:PORT`** (if you need direct Tailscale IP access without Serve):

- App is reachable on all interfaces including Tailscale's `100.x.y.z`
- Must be combined with UFW to block public interface access
- Less ideal — prefer Tailscale Serve as the ingress point

## Handling multiple hostnames

Your backend will receive requests with different `Host` headers depending on how the client connects:

| Client | Host header |
|---|---|
| Same-machine frontend | `localhost:4000` |
| Tailnet device via Serve | `hostname.tailnet-name.ts.net` |
| Tailnet device via raw IP | `100.x.y.z:4000` |

### CORS configuration

If your frontend makes cross-origin requests to the API:

```typescript
// Express/Fastify example
const ALLOWED_ORIGINS = [
  'http://localhost:3000',           // local frontend dev
  'https://mac-mini.tailnet.ts.net', // tailnet access
];

// Or dynamically:
const TAILSCALE_DOMAIN = process.env.TS_DOMAIN; // set from tailscale status
if (TAILSCALE_DOMAIN) {
  ALLOWED_ORIGINS.push(`https://${TAILSCALE_DOMAIN}`);
}
```

### Discovering the current Tailscale identity at startup

```typescript
import { execSync } from 'child_process';

function getTailscaleInfo() {
  try {
    const status = JSON.parse(
      execSync('tailscale status --json', { encoding: 'utf-8' })
    );
    return {
      ip: status.TailscaleIPs?.[0],         // 100.x.y.z
      dnsName: status.Self?.DNSName?.replace(/\.$/, ''), // host.tailnet.ts.net
      hostname: status.Self?.HostName,       // host
    };
  } catch {
    return null; // Tailscale not running
  }
}

// Use at startup
const tsInfo = getTailscaleInfo();
if (tsInfo) {
  console.log(`Tailscale: ${tsInfo.dnsName} (${tsInfo.ip})`);
  // Register with backend coordinator, update CORS, etc.
}
```

```python
import subprocess, json

def get_tailscale_info():
    try:
        status = json.loads(
            subprocess.check_output(["tailscale", "status", "--json"], text=True)
        )
        dns_name = status.get("Self", {}).get("DNSName", "").rstrip(".")
        return {
            "ip": status.get("TailscaleIPs", [None])[0],
            "dns_name": dns_name,
            "hostname": status.get("Self", {}).get("HostName"),
        }
    except Exception:
        return None
```

## Worker registration pattern

Remote workers connect to the backend API to register themselves:

```python
import os, httpx

ts_info = get_tailscale_info()
backend_url = os.environ.get("BACKEND_URL", "https://mac-mini.tailnet.ts.net")

# Register this worker with the backend
httpx.post(f"{backend_url}/api/workers/register", json={
    "hostname": ts_info["hostname"],
    "tailscale_ip": ts_info["ip"],
    "dns_name": ts_info["dns_name"],
    "capabilities": ["gpu", "image-processing"],
})
```

## Environment variable pattern

Set `BACKEND_URL` based on context:

```bash
# On the same machine as the backend (frontend dev)
export BACKEND_URL=http://localhost:4000

# On a remote worker
export BACKEND_URL=https://mac-mini.tailnet.ts.net

# Or use the Tailscale IP directly
export BACKEND_URL=http://100.64.1.23:4000
```

Your app reads `BACKEND_URL` and doesn't need to know whether it's local, Tailscale Serve, or a raw IP.

## Same-machine frontend + backend

When frontend and backend are on the same machine:

- Frontend dev server: `localhost:3000`
- Backend: `localhost:4000`
- Frontend fetches from `localhost:4000` — no Tailscale involved, no CORS issues
- Tailscale Serve on port 4000 makes the API available to remote workers/devices

The key insight: `tailscale serve` doesn't interfere with localhost access. Both local and Tailscale connections work simultaneously.
