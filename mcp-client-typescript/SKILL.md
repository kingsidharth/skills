---
name: mcp-client-typescript
description: "Connect to MCP servers from TypeScript using Vercel AI SDK's @ai-sdk/mcp client. Use when building MCP clients, consuming MCP tools/resources/prompts, connecting to remote or local MCP servers, or integrating MCP with generateText/streamText. Covers HTTP, SSE, and stdio transports, schema discovery vs definition, typed outputs, resources, prompts, elicitation, and lifecycle management."
---

# MCP Client — TypeScript (Vercel AI SDK)

Connect to MCP servers and use their tools, resources, and prompts from TypeScript applications via `@ai-sdk/mcp`.

Packages: `@ai-sdk/mcp` (client), `ai` (core SDK), optionally `@modelcontextprotocol/sdk` (official transports).

Docs: https://ai-sdk.dev/docs/ai-sdk-core/mcp-tools

## When to Read Reference Files

| Scenario | Reference File |
|---|---|
| Client setup, transports (HTTP/SSE/stdio), lifecycle, closing | `references/client-setup.md` |
| Using tools: discovery, typed schemas, outputSchema, multi-server merging | `references/tools.md` |
| Resources, prompts, elicitation, advanced features | `references/advanced-features.md` |
| Next.js integration (route handlers, useChat, useCompletion) | `references/nextjs-integration.md` |

## Quick Start

```ts
import { createMCPClient } from '@ai-sdk/mcp';
import { generateText } from 'ai';

const client = await createMCPClient({
  transport: { type: 'http', url: 'https://your-server.com/mcp' },
});

const tools = await client.tools();

const { text } = await generateText({
  model: 'anthropic/claude-sonnet-4.5',
  tools,
  prompt: 'What is the weather in NYC?',
});

await client.close();
```

## Key Decisions

**Transport choice**: HTTP (`type: 'http'`) for production. SSE (`type: 'sse'`) as alternative. Stdio for local-only dev servers.

**Tool schema approach**: Schema discovery (`client.tools()`) for rapid prototyping — auto-syncs with server but no TS types. Schema definition (`client.tools({ schemas: {...} })`) for type safety — explicit Zod schemas, selective tool loading.

**Lifecycle**: Always close clients. For streaming, close in `onFinish`. For non-streaming, use `try/finally`.

**Multi-server**: Spread multiple tool sets into one object. Beware name collisions — later tools override earlier ones.
