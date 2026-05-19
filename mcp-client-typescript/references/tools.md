# Using MCP Tools

The `client.tools()` method adapts MCP tools into AI SDK tools, compatible with `generateText`, `streamText`, and `useChat`.

## Schema Discovery (Dynamic)

Auto-discovers all tools from the server. No TS type safety, but always in sync with server:

```ts
const tools = await client.tools();
// tools is Record<string, Tool> — all server tools available
```

## Schema Definition (Typed)

Define explicit Zod schemas for type safety and selective tool loading:

```ts
import { z } from 'zod';

const tools = await client.tools({
  schemas: {
    'get-data': {
      inputSchema: z.object({
        query: z.string().describe('The data query'),
        format: z.enum(['json', 'text']).optional(),
      }),
    },
    'tool-with-no-args': {
      inputSchema: z.object({}),  // empty object for zero-input tools
    },
  },
});
```

Only the tools named in `schemas` are loaded. Full IDE autocompletion and compile-time checking.

## Typed Tool Outputs

When the MCP server returns `structuredContent`, define `outputSchema` for typed results:

```ts
const tools = await client.tools({
  schemas: {
    'get-weather': {
      inputSchema: z.object({ location: z.string() }),
      outputSchema: z.object({
        temperature: z.number(),
        conditions: z.string(),
        humidity: z.number(),
      }),
    },
  },
});

// Direct execution with typed result
const result = await tools['get-weather'].execute(
  { location: 'New York' },
  { messages: [], toolCallId: 'weather-1' },
);
console.log(result.temperature); // typed as number
```

Fallback chain: `structuredContent` → JSON-parsed text content → error.

Without `outputSchema`, the tool returns the raw `CallToolResult` object (`{ content, isError? }`).

## Using with generateText / streamText

```ts
import { generateText, stepCountIs } from 'ai';

const { text } = await generateText({
  model: 'anthropic/claude-sonnet-4.5',
  tools,
  prompt: 'Find products under $100',
  stopWhen: stepCountIs(5),  // limit agent loops
});
```

```ts
import { streamText } from 'ai';

const result = await streamText({
  model: 'anthropic/claude-sonnet-4.5',
  tools,
  prompt: 'Summarize the latest issues',
  onFinish: async () => await client.close(),
});
```

## Merging Multiple Tool Sets

```ts
const tools = {
  ...(await clientA.tools()),
  ...(await clientB.tools()),
};
```

Name collisions: later entries override earlier ones silently. Namespace tools if needed or use schema definition to pick specific tools from each server.

## Dynamic Tools in useChat (AI SDK 5+)

MCP tools discovered at runtime are "dynamic tools" — their schemas aren't known at compile time. In `useChat`, render them via the `dynamic-tool` message part type. See the AI SDK dynamic tools docs for UI rendering patterns.
