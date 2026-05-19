---
name: tailscale-networking
description: Private mesh networking with Tailscale — server setup, auth keys, ACLs, Tailscale Serve/Funnel, Services (TailVIPs), firewall lockdown, cloud-init provisioning, and the tailscale Python API client. Use when connecting backend servers, workers, or services over Tailscale, setting up MagicDNS, HTTPS certs, or locking down VPS/cloud nodes.
---

# Tailscale Networking

## Quick orientation

Tailscale creates a WireGuard mesh VPN ("tailnet") where every device gets a stable `100.x.y.z` IP and a MagicDNS name (`hostname.tailnet-name.ts.net`). All traffic between devices is end-to-end encrypted. No ports need to be opened.

**Core concepts**: tailnet, MagicDNS, auth keys, tags, Tailscale Serve, Tailscale Funnel, Tailscale Services (TailVIPs), ACLs/grants, ephemeral nodes.

## When to read which reference

| Task | Reference file |
|---|---|
| Provision a server (auth keys, tags, cloud-init, Hetzner) | [server-setup.md](references/server-setup.md) |
| Expose a local service to tailnet or internet | [serve-and-funnel.md](references/serve-and-funnel.md) |
| Tailscale Services with virtual IPs and HA | [services.md](references/services.md) |
| Lock down a VPS with UFW, disable public SSH | [firewall-lockdown.md](references/firewall-lockdown.md) |
| ACL policy file, tags, grants, access control | [acls-and-policy.md](references/acls-and-policy.md) |
| Python API client (`tailscale` / `tailscale-api` on PyPI) | [python-api.md](references/python-api.md) |
| Backend app that accepts traffic on multiple domains/IPs | [multi-domain-binding.md](references/multi-domain-binding.md) |

## Common architecture pattern

```
┌─────────────────────────────────────────┐
│  Mac Mini (home)                        │
│  ├─ Frontend  → localhost:3000          │
│  ├─ Backend   → localhost:4000          │
│  └─ tailscale serve --bg 4000           │
│     → https://mac-mini.ts.net (tailnet) │
└─────────────────────────────────────────┘
        ▲ tailnet (encrypted)
        │
┌───────┴─────────────────────────────────┐
│  Hetzner Worker (remote)                │
│  ├─ Worker process                      │
│  ├─ tailscale up --auth-key=...         │
│  ├─ UFW: deny all except tailscale0     │
│  └─ Outbound internet: ✓ (B2, images)  │
└─────────────────────────────────────────┘
```

- Frontend on the Mac Mini connects to backend via `localhost:4000` (same machine, no Tailscale needed).
- Remote workers and other tailnet devices reach backend via `https://mac-mini.ts.net` (Tailscale Serve provides auto-TLS).
- Workers register with backend on startup using the Tailscale hostname/IP.
- UFW on workers blocks all inbound except `tailscale0`, but allows all outbound (internet for B2 uploads, image downloads, etc.).
