# Transport

## stdio (local servers)

Simplest transport. Server is spawned as a child process; JSON-RPC flows over stdin/stdout.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({ name: "my-server", version: "1.0.0" });
// ... register tools/resources/prompts ...

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Critical**: never write to stdout from application code — it corrupts the JSON-RPC stream. Use `console.error()` or `process.stderr.write()` for all logging.

## Streamable HTTP (remote servers)

Recommended for network-accessible servers. Supports multiple simultaneous clients.

### Stateless (simple, scales horizontally)

One transport per request, no session tracking. Best default for most servers:

```typescript
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

const app = express();
app.use(express.json());

app.post("/mcp", async (req, res) => {
  const server = new McpServer({ name: "my-server", version: "1.0.0" });
  // ... register tools ...

  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,    // no sessions
    enableJsonResponse: true,         // JSON instead of SSE streaming
  });
  res.on("close", () => transport.close());
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000, () => console.error("MCP server on http://localhost:3000/mcp"));
```

### Stateful (sessions, server-to-client notifications)

Track sessions when you need server-initiated messages or per-user state:

```typescript
import { randomUUID } from "node:crypto";
import { isInitializeRequest } from "@modelcontextprotocol/sdk/types.js";

const sessions: Record<string, StreamableHTTPServerTransport> = {};

app.post("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;

  if (sessionId && sessions[sessionId]) {
    await sessions[sessionId].handleRequest(req, res, req.body);
    return;
  }

  if (!sessionId && isInitializeRequest(req.body)) {
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
    });
    const server = new McpServer({ name: "my-server", version: "1.0.0" });
    // ... register tools ...
    await server.connect(transport);

    const newSessionId = transport.sessionId!;
    sessions[newSessionId] = transport;

    await transport.handleRequest(req, res, req.body);
    return;
  }

  res.status(400).json({ error: "Bad request" });
});

// GET for server-to-client SSE stream (notifications)
app.get("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string;
  if (!sessionId || !sessions[sessionId]) {
    res.status(404).end();
    return;
  }
  await sessions[sessionId].handleRequest(req, res);
});

// DELETE to terminate session
app.delete("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string;
  if (sessionId && sessions[sessionId]) {
    sessions[sessionId].close();
    delete sessions[sessionId];
  }
  res.status(200).end();
});
```

## Middleware packages (v2)

The v2 SDK publishes thin adapters for common frameworks:

| Package | Use case |
|---|---|
| `@modelcontextprotocol/node` | Node.js `IncomingMessage`/`ServerResponse` |
| `@modelcontextprotocol/express` | Express (adds Host header validation, app defaults) |
| `@modelcontextprotocol/hono` | Hono (Workers, Deno, Bun) |

These handle CORS, DNS rebinding protection, and Host header validation out of the box.

## Transport selection

| Criterion | stdio | Streamable HTTP |
|---|---|---|
| Deployment | Local | Remote |
| Clients | Single | Multiple |
| Auth | Implicit (process) | Bearer/OAuth |
| Notifications | N/A | SSE stream via GET |

## Dual transport

Support both in a single binary:

```typescript
const transport = process.env.TRANSPORT || "stdio";
if (transport === "http") {
  await runHTTP();
} else {
  await runStdio();
}
```

## Notifications

Notify clients when server capabilities change:

```typescript
server.notification({ method: "notifications/tools/list_changed" });
server.notification({ method: "notifications/resources/list_changed" });
```

Use sparingly — only when capabilities genuinely change at runtime.
