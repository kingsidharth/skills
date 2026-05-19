# Debugging

## MCP Inspector

Browser-based interactive debugger for MCP servers. The "Postman for MCP."

### Launch

```bash
# Against a local stdio server
npx @modelcontextprotocol/inspector node dist/index.js

# Against a built npm package
npx @modelcontextprotocol/inspector npx my-mcp-server

# With environment variables
API_KEY=xxx npx @modelcontextprotocol/inspector node dist/index.js

# Against a remote Streamable HTTP server
# Open http://localhost:6274, select "Streamable HTTP", enter URL
npx @modelcontextprotocol/inspector
```

Opens at `http://localhost:6274`. Proxy runs on port `6277`.

### Security

The Inspector generates a random session token on startup printed to the console. The browser URL includes the token automatically. For non-default network configs:

```bash
# Custom auth token
MCP_PROXY_AUTH_TOKEN=$(openssl rand -hex 32) npx @modelcontextprotocol/inspector node dist/index.js

# Bind to all interfaces (trusted networks only)
HOST=0.0.0.0 npx @modelcontextprotocol/inspector node dist/index.js
```

### Features

| Tab | Purpose |
|---|---|
| **Tools** | List tools, fill input forms, execute, inspect JSON response |
| **Resources** | Browse resources, read contents, test subscriptions |
| **Prompts** | Preview prompt templates, supply arguments, see generated messages |
| **History** | Full JSON-RPC request/response log |

### Docker

```bash
docker run --rm \
  -p 127.0.0.1:6274:6274 \
  -p 127.0.0.1:6277:6277 \
  -e HOST=0.0.0.0 \
  -e MCP_AUTO_OPEN_ENABLED=false \
  ghcr.io/modelcontextprotocol/inspector
```

## Claude Desktop developer mode

Enable in Settings → Developer → "Enable developer mode". Logs go to:

- **macOS**: `~/Library/Logs/Claude/mcp*.log`
- **Windows**: `%APPDATA%\Claude\logs\mcp*.log`

Tail logs while testing:

```bash
tail -f ~/Library/Logs/Claude/mcp*.log
```

## Server-side logging

### stdio servers

All logging to `stderr` — stdout is the JSON-RPC channel:

```typescript
console.error("INFO: Processing request...");
console.error(`ERROR: Failed to fetch: ${error.message}`);
```

### HTTP servers

Use structured logging (pino, winston, etc.) with request IDs for correlation. Don't leak internal errors to clients — return actionable error messages in tool responses instead.

## MCPJam Inspector (alternative)

Open-source alternative with additional features: OAuth flow debugger, multi-server chat playground, CI/SDK integration. Available as hosted web app, desktop app, or CLI:

```bash
npx mcpjam@latest
```

Useful when debugging OAuth flows specifically — it visualizes each step of the authorization handshake.
