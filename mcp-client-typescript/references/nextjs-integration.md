# Next.js Integration

## Route Handler with MCP Tools

Create MCP clients inside the route handler. Close them when the response finishes.

```ts
// app/api/chat/route.ts
import { createMCPClient } from '@ai-sdk/mcp';
import { streamText } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const client = await createMCPClient({
    transport: { type: 'http', url: process.env.MCP_SERVER_URL! },
  });

  const tools = await client.tools();

  const result = await streamText({
    model: 'anthropic/claude-sonnet-4.5',
    tools,
    messages,
    onFinish: async () => await client.close(),
    onError: async () => await client.close(),
  });

  return result.toDataStreamResponse();
}
```

## Client Component with useChat

```tsx
'use client';
import { useChat } from '@ai-sdk/react';

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    api: '/api/chat',
  });

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>{m.role}: {m.content}</div>
      ))}
      <input value={input} onChange={handleInputChange} />
      <button onClick={handleSubmit}>Send</button>
    </div>
  );
}
```

## Client Component with useCompletion

For simpler single-prompt flows:

```tsx
'use client';
import { useCompletion } from '@ai-sdk/react';

export default function Page() {
  const { completion, complete } = useCompletion({ api: '/api/completion' });

  return (
    <div>
      <button onClick={() => complete('Schedule a meeting for tomorrow at 10am')}>
        Run
      </button>
      <p>{completion}</p>
    </div>
  );
}
```

## Performance Considerations

**Client creation cost**: Each `createMCPClient` call establishes a connection. For HTTP/SSE, this involves network round-trips. Don't create clients per-render — do it in route handlers or server actions.

**Stdio in serverless**: Stdio spawns child processes. Incompatible with edge runtime and most serverless environments. Use HTTP/SSE for deployed Next.js apps.

**Tool caching**: If your MCP server's tool list is stable, consider caching the tool definitions to avoid re-fetching on every request. The SDK doesn't do this automatically.

## Server Actions Alternative

```ts
'use server';
import { createMCPClient } from '@ai-sdk/mcp';
import { generateText } from 'ai';

export async function runAgent(prompt: string) {
  const client = await createMCPClient({
    transport: { type: 'http', url: process.env.MCP_SERVER_URL! },
  });

  try {
    const tools = await client.tools();
    const { text } = await generateText({
      model: 'anthropic/claude-sonnet-4.5',
      tools,
      prompt,
    });
    return text;
  } finally {
    await client.close();
  }
}
```
