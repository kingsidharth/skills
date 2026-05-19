# Python API Clients

Two main options for working with the Tailscale API from Python.

## Option 1: `tailscale` (frenck/python-tailscale)

Async client for the Tailscale local daemon API and control plane.

```bash
pip install tailscale
# or
uv add tailscale
```

```python
import asyncio
from tailscale import Tailscale

async def main():
    async with Tailscale(
        tailnet="your-tailnet",
        api_key="tskey-api-...",
    ) as ts:
        devices = await ts.devices()
        for device in devices.values():
            print(f"{device.name}: {device.addresses}")

asyncio.run(main())
```

- **Requires**: Python ≥3.11
- **PyPI**: https://pypi.org/project/tailscale/
- **GitHub**: https://github.com/frenck/python-tailscale

## Option 2: `tailscale-api`

Sync client, supports both API key and OAuth auth:

```bash
pip install tailscale-api
# or
uv add tailscale-api
```

```python
import tailscale_api

tsc = tailscale_api.TailscaleAPIClient()

# Auth with API key
tsc.set_token("tskey-api-...")

# Or auth with OAuth
tsc.set_oauth_client_info("client_id", "tskey-client-...")
tsc.set_token(tsc.get_oauth_token())

for device in tsc.devices():
    print(device.name)
```

- **PyPI**: https://pypi.org/project/tailscale-api/

## Tailscale REST API directly

Both libraries wrap the Tailscale REST API. You can also call it directly:

```python
import httpx

# Using OAuth to get an access token
token_resp = httpx.post(
    "https://api.tailscale.com/api/v2/oauth/token",
    data={
        "client_id": TS_OAUTH_ID,
        "client_secret": TS_OAUTH_SECRET,
        "grant_type": "client_credentials",
    },
)
access_token = token_resp.json()["access_token"]

# List devices
devices = httpx.get(
    "https://api.tailscale.com/api/v2/tailnet/-/devices",
    headers={"Authorization": f"Bearer {access_token}"},
).json()

# Create an auth key
key = httpx.post(
    "https://api.tailscale.com/api/v2/tailnet/-/keys",
    headers={"Authorization": f"Bearer {access_token}"},
    json={
        "capabilities": {
            "devices": {
                "create": {
                    "reusable": True,
                    "ephemeral": True,
                    "preauthorized": True,
                    "tags": ["tag:worker"],
                }
            }
        },
        "expirySeconds": 86400,
    },
).json()
print(key["key"])
```

## Note on `tsnet`

`tsnet` is a Go library for embedding a Tailscale node directly into a Go program (no daemon required). There is no Python equivalent — Python programs must use the system `tailscaled` daemon and interact via the local API or the control plane REST API.

The Go `tsnet.Server` pattern is useful for microservices that each need their own Tailscale identity:

```go
srv := &tsnet.Server{Hostname: "my-service", AuthKey: os.Getenv("TS_AUTHKEY")}
ln, _ := srv.Listen("tcp", ":443")
```

For Python workloads, install Tailscale on the host and use `tailscale serve` to expose local services.
