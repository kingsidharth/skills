---
name: mcp-server-typescript
description: Build MCP servers in TypeScript using the official SDK. Exposing internal functions as tools, prompts, resources. Covers transport (stdio, Streamable HTTP), auth (API keys, OAuth 2.1), debugging with MCP Inspector, and remote exposure via Tailscale/Cloudflare Tunnel. Use when building, scaffolding, or deploying MCP servers in TypeScript/Node.
---

# MCP Server — TypeScript

Build MCP servers using the official `@modelcontextprotocol/sdk` (v1) or `@modelcontextprotocol/server` (v2).

## SDK versions

Two generations coexist. Choose based on stability needs:

| | v1 (stable) | v2 (current) |
|---|---|---|
| **Package** | `@modelcontextprotocol/sdk` | `@modelcontextprotocol/server` |
| **Import** | `@modelcontextprotocol/sdk/server/mcp.js` | `@modelcontextprotocol/server` |
| **Zod** | `zod` (v3.25+) | `zod/v4` (or any Standard Schema) |
| **Schema** | Zod only | Standard Schema (Zod v4, Valibot, ArkType) |
| **Status** | Bug fixes, security patches | Active development |

v1 imports use deep paths (`/sdk/server/mcp.js`). v2 uses top-level package names. All examples in this skill use **v1** unless noted — it's what most production servers run today. Migration is straightforward: change imports, switch to `zod/v4`.

## Primitives

- **Tools** — LLM-invoked actions with validated input/output schemas. See [TOOLS.md](references/TOOLS.md).
- **Resources** — URI-addressed read-only data (files, DB records, API responses). See [RESOURCES.md](references/RESOURCES.md).
- **Prompts** — Reusable message templates with typed arguments. See [PROMPTS.md](references/PROMPTS.md).

## Transport

- **stdio** — local servers spawned as child processes (Claude Desktop, CLI tools)
- **Streamable HTTP** — remote servers over network (recommended for multi-client)
- **SSE** — legacy, avoid for new projects

Details and session management: [TRANSPORT.md](references/TRANSPORT.md).

## Auth

Progresses from simple to production-grade:

1. **Environment variable** — API key in env, validated at startup
2. **Bearer token** — static token checked per-request
3. **OAuth 2.1 + PKCE** — full spec compliance for public remote servers

Details: [AUTH.md](references/AUTH.md).

## Debugging

- **MCP Inspector** — browser UI for testing tools, resources, prompts
- **Claude Desktop** — `developer_mode` log tailing
- **Server-side logging** — stderr for stdio, structured logs for HTTP

Details: [DEBUGGING.md](references/DEBUGGING.md).

## Remote exposure

Make a local MCP server reachable from remote clients (Claude.ai, ChatGPT, other agents):

- **Tailscale Serve/Funnel** — private mesh or public HTTPS, zero config
- **Cloudflare Tunnel** — public endpoint with Zero Trust access policies
- **Direct deploy** — Cloudflare Workers, Fly.io, Railway

Details: [REMOTE.md](references/REMOTE.md).

## Connecting to Claude

Claude.ai supports remote MCP servers via the integrations UI. Also covers Claude Desktop `claude_desktop_config.json` for local stdio servers. See [CONNECTING.md](references/CONNECTING.md).

## Project structure & quality checklist

Scaffolding, `package.json`, `tsconfig.json`, build pipeline, and pre-ship checklist: [PROJECT.md](references/PROJECT.md).
