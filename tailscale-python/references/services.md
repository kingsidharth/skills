# Tailscale Services (TailVIPs)

Tailscale Services assign stable virtual IPs (TailVIPs) and MagicDNS names to logical resources, decoupled from any specific device. Services enable HA routing, service discovery, and fine-grained ACLs per service.

**Status**: GA as of early 2026. Available on all plans.

## Concepts

- **Service**: a named resource with a TailVIP, MagicDNS name, endpoint definitions, and one or more hosts
- **TailVIP**: a virtual Tailscale IP pair (v4 + v6) that belongs to the service, not to any device
- **Host**: a Tailscale device that advertises endpoints for a service
- **Endpoint**: a port mapping from the service's virtual IP to a local destination

## Defining a Service

Create via admin console (Services page) or the Tailscale API:

1. Name and description
2. Endpoint ports (e.g., `tcp:443`, `tcp:5432`)
3. Optional tags for identity and ACL grouping

## Hosting a Service

### CLI method

```bash
# Advertise a service endpoint via tailscale serve
tailscale serve --service=<tailvip> --bg <local-target>

# Example: proxy HTTPS to local API on port 4000
tailscale serve --service=100.100.100.50 --bg 4000
```

### Declarative config file

```json
{
  "version": "0.0.1",
  "services": {
    "svc:webapp": {
      "endpoints": {
        "tcp:443": "http://localhost:8096"
      }
    },
    "svc:db": {
      "endpoints": {
        "tcp:3306": "tcp://db:3306",
        "tcp:443": "https://prod:8443"
      }
    }
  }
}
```

Apply config:

```bash
tailscale serve set-config /path/to/services.json
```

Reload on change without restart:

```bash
tailscale serve set-config /path/to/services.json
```

### Get current config

```bash
tailscale serve get-config
```

## Endpoint types

| Layer | Protocol prefix | Use case |
|---|---|---|
| Layer 7 (app) | `http://`, `https://` | Web servers, APIs — Tailscale terminates TLS |
| Layer 4 (transport) | `tcp://`, `tls-terminated-tcp://` | Databases, raw TCP — no packet modification |
| Layer 3 (network) | `--tun` flag | Full control via iptables, Linux only |

## High availability

Multiple hosts can advertise the same service. Tailscale uses Regional Routing to steer clients to the nearest healthy host:

```bash
# On host A (us-east)
tailscale serve --service=100.100.100.50 --bg 4000

# On host B (eu-west)
tailscale serve --service=100.100.100.50 --bg 4000
```

Drain a host gracefully:

```bash
tailscale serve drain svc:webapp    # stop new connections, existing ones continue
tailscale serve advertise svc:webapp  # bring back online
```

## Accessing services

From any tailnet device:

```bash
# Via MagicDNS
curl https://webapp.tailnet-name.ts.net

# Via TailVIP
curl https://100.100.100.50
```

TailVIPs are automatically routed to clients — no `--accept-routes` needed (requires client v1.94.1+).

## Limitations

- TCP only (no UDP, except via layer 3 on Linux with iptables)
- TailVIPs accept incoming connections only (no outgoing)
- No hairpinning: a host device cannot access the service it hosts
- `text:` and `file:` targets not supported in declarative config (CLI only)

## ACLs for services

Services can be tagged and referenced in grants:

```jsonc
{
  "grants": [{
    "src": ["group:eng"],
    "dst": ["tag:webapp"],
    "ip": ["*"]
  }]
}
```
