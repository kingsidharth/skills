# Client Setup & Transports

## Installation

```bash
bun add @ai-sdk/mcp ai
# Optional: official MCP SDK transports
bun add @modelcontextprotocol/sdk
```

## Transport Options

### HTTP (Recommended for production)

Two approaches — built-in config or official SDK transport:

```ts
import { createMCPClient } from '@ai-sdk/mcp';

// Built-in HTTP config
const client = await createMCPClient({
  transport: {
    type: 'http',
    url: 'https://your-server.com/mcp',
    headers: { Authorization: 'Bearer my-api-key' },     // optional
    authProvider: myOAuthClientProvider,                   // optional: OAuth
    redirect: 'error',                                     // optional: reject redirects (SSRF protection)
  },
});

// Or via official SDK transport
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

const client = await createMCPClient({
  transport: new StreamableHTTPClientTransport(
    new URL('https://your-server.com/mcp'),
    { sessionId: 'session_123' },
  ),
});
```

### SSE (Alternative HTTP-based)

```ts
const client = await createMCPClient({
  transport: {
    type: 'sse',
    url: 'https://your-server.com/sse',
    headers: { Authorization: 'Bearer my-api-key' },
    authProvider: myOAuthClientProvider,
  },
});
```

SSE is the older remote transport. Streamable HTTP supersedes it per MCP spec 2025-03-26. Use SSE only if the server hasn't migrated.

### Stdio (Local dev only)

```ts
import { Experimental_StdioMCPTransport } from '@ai-sdk/mcp/mcp-stdio';
// Or: import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

const client = await createMCPClient({
  transport: new Experimental_StdioMCPTransport({
    command: 'node',
    args: ['src/server.js'],
  }),
});
```

Cannot be deployed to production — requires spawning a child process.

### Custom Transport

Implement the `MCPTransport` interface for non-standard requirements.

## Lifecycle Management

Always close clients to release resources.

**Non-streaming** — use `try/finally`:

```ts
import { createMCPClient, type MCPClient } from '@ai-sdk/mcp';

let client: MCPClient | undefined;
try {
  client = await createMCPClient({ transport: { type: 'http', url: '...' } });
  const tools = await client.tools();
  const { text } = await generateText({ model: '...', tools, prompt: '...' });
} finally {
  await client?.close();
}
```

**Streaming** — close in `onFinish`:

```ts
const client = await createMCPClient({ /* ... */ });
const tools = await client.tools();

const result = await streamText({
  model: 'anthropic/claude-sonnet-4.5',
  tools,
  prompt: '...',
  onFinish: async () => await client.close(),
  onError: async () => await client.close(), // optional: free resources immediately on error
});
```

**Long-running apps** (CLI tools, daemons): keep client open, close on process exit.

## Multi-Server Setup

```ts
const clientA = await createMCPClient({ transport: { type: 'http', url: 'https://server-a.com/mcp' } });
const clientB = await createMCPClient({ transport: { type: 'http', url: 'https://server-b.com/mcp' } });

const tools = {
  ...(await clientA.tools()),
  ...(await clientB.tools()),  // ⚠️ tools with same name override earlier ones
};
```

Close all clients when done:
```ts
await Promise.all([clientA.close(), clientB.close()]);
```

## Limitations of the AI SDK MCP Client

The `createMCPClient` is a lightweight wrapper for tool conversion. It does **not** currently support: session management, resumable streams, or receiving server notifications (beyond elicitation). For full MCP client features, use the official `@modelcontextprotocol/sdk` `Client` class and feed tools manually.
