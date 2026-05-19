# Connecting to Claude

## Claude.ai (remote MCP servers)

Claude.ai supports connecting to remote MCP servers via the integrations UI.

### Steps

1. Go to **Settings → Integrations** (or the integrations panel in the sidebar)
2. Click **"Add integration"**
3. Enter the server's Streamable HTTP URL (e.g., `https://mcp.example.com/mcp`)
4. If the server requires OAuth, you'll be redirected to authenticate
5. Once connected, the server's tools appear in Claude's tool picker

### Requirements for your server

- Must serve Streamable HTTP transport at a public HTTPS URL
- Must implement OAuth 2.1 if the server requires auth (Claude handles the OAuth flow)
- Must serve Protected Resource Metadata at `/.well-known/oauth-protected-resource` for OAuth servers
- Must support Dynamic Client Registration (RFC 7591) — Claude registers as a new client on connect

### No-auth servers

If your server doesn't require auth (e.g., behind Cloudflare Access that handles auth separately), Claude connects directly to the URL with no OAuth dance.

## Claude Desktop (local stdio servers)

Configure in `claude_desktop_config.json`:

**macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["/absolute/path/to/dist/index.js"],
      "env": {
        "API_KEY": "your-key-here"
      }
    }
  }
}
```

For npm-published servers:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"]
    }
  }
}
```

### Claude Desktop with remote servers

Use `mcp-remote` to bridge a remote Streamable HTTP server into Claude Desktop's stdio expectation:

```json
{
  "mcpServers": {
    "my-remote-server": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "--transport", "http-first", "https://mcp.example.com/mcp"]
    }
  }
}
```

`mcp-remote` handles OAuth, transport bridging (HTTP → stdio), and token caching.

## Other MCP clients

Any compliant client (Cursor, Windsurf, VS Code Copilot, custom agents) connects the same way:
- **stdio**: configure `command` + `args` in the client's MCP settings
- **Streamable HTTP**: provide the URL; client handles OAuth if needed
