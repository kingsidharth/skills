# Remote Exposure

Make a locally-running MCP server reachable by remote clients.

## Decision matrix

| Method | Audience | Auth | Complexity | Public internet? |
|---|---|---|---|---|
| Tailscale Serve | Private (your devices/team) | Tailnet membership | Low | No |
| Tailscale Funnel | Anyone with URL | HTTPS + optional app auth | Low | Yes |
| Cloudflare Tunnel | Anyone with URL | Zero Trust policies | Medium | Yes |
| Cloudflare Workers | Anyone with URL | OAuth 2.1 | Medium | Yes |
| Direct deploy (Fly/Railway) | Anyone with URL | OAuth 2.1 | Medium | Yes |

## Tailscale Serve (private access)

Exposes a local port to your tailnet with automatic HTTPS and MagicDNS:

```bash
# Expose local MCP server to your tailnet
tailscale serve --https=443 http://localhost:3000

# Your server is now at https://<hostname>.<tailnet>.ts.net/mcp
```

No port forwarding, no certificates to manage. All devices on your tailnet can reach it. Good for: dev machines, Electron apps exposing tools to your own agents, home lab setups.

## Tailscale Funnel (public access)

Same as Serve but proxied through Tailscale's edge to the public internet:

```bash
tailscale funnel --https=443 http://localhost:3000
# Public URL: https://<hostname>.<tailnet>.ts.net
```

Good for: quick public sharing, demos, webhook callbacks. Free tier supports 3 funnels.

## Cloudflare Tunnel (production public access)

Creates an encrypted outbound-only tunnel to Cloudflare's edge. No inbound ports needed.

### Setup

```bash
# Install cloudflared
brew install cloudflared   # or apt, or Docker

# Login and create tunnel
cloudflared tunnel login
cloudflared tunnel create my-mcp-tunnel

# Route traffic
cloudflared tunnel route dns my-mcp-tunnel mcp.example.com
```

### Config file (`~/.cloudflared/config.yml`)

```yaml
tunnel: <tunnel-id>
credentials-file: ~/.cloudflared/<tunnel-id>.json
ingress:
  - hostname: mcp.example.com
    service: http://localhost:3000
  - service: http_status:404
```

### Run

```bash
cloudflared tunnel run my-mcp-tunnel
```

### Docker Compose

```yaml
services:
  mcp-server:
    build: .
    ports: ["3000:3000"]
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      TUNNEL_TOKEN: ${TUNNEL_TOKEN}
```

### Add Zero Trust access policies

In Cloudflare dashboard → Zero Trust → Access → Applications:
- Restrict by email, IP, identity provider
- Require device posture checks
- MFA enforcement

## Electron app as MCP server

An Electron app can embed an MCP server to expose its functionality to external agents.

### Architecture

```
Electron App
├── Main Process
│   ├── App logic
│   └── MCP Server (Streamable HTTP on localhost:3000)
└── Renderer Process
    └── UI
```

### Implementation

In Electron's main process:

```typescript
// main.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const mcpApp = express();
mcpApp.use(express.json());

const server = new McpServer({ name: "my-electron-app", version: "1.0.0" });

// Register tools that bridge to your app's internal APIs
server.registerTool("app_get_data", { /* ... */ }, async (params) => {
  const data = await myAppModule.getData(params);
  return { content: [{ type: "text", text: JSON.stringify(data) }] };
});

mcpApp.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
    enableJsonResponse: true,
  });
  res.on("close", () => transport.close());
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

mcpApp.listen(3847, "127.0.0.1"); // bind to localhost only
```

### Exposing to other machines

**Tailscale** (recommended for private access):
```bash
# On the machine running the Electron app
tailscale serve --https=443 http://localhost:3847
```

**Cloudflare Tunnel** (public access):
```bash
cloudflared tunnel --url http://localhost:3847
```

### Security considerations

- Bind to `127.0.0.1`, not `0.0.0.0` — don't expose to LAN by default
- Add bearer token auth middleware if exposing beyond localhost
- Validate `Origin` header for DNS rebinding protection
- Consider adding an in-app toggle for users to enable/disable the MCP server
- For Tailscale: access is implicitly restricted to tailnet members
- For Cloudflare: use Access policies to restrict who can connect

## Cloudflare Workers (serverless deploy)

For fully managed deployment without running infrastructure:

```bash
npm create cloudflare@latest -- my-mcp-server --template=cloudflare/ai/demos/remote-mcp-authless
```

Add OAuth via Cloudflare Access or bring your own provider. See Cloudflare Agents docs for `createMcpHandler()` and `McpAgent` (Durable Object per session).
