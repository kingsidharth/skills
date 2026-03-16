# Tool Calling + Generative UI

## Define tools (backend)

```ts
import { tool, streamText } from 'ai';
import { z } from 'zod';

const tools = {
  getWeather: tool({
    description: 'Get current weather for a location',
    inputSchema: z.object({
      location: z.string().describe('City name'),
      unit: z.enum(['celsius', 'fahrenheit']).optional(),
    }),
    execute: async ({ location, unit = 'celsius' }) => {
      // Call weather API
      return { temp: 22, condition: 'sunny', location };
    },
  }),

  searchWeb: tool({
    description: 'Search the web',
    inputSchema: z.object({ query: z.string() }),
    execute: async ({ query }) => {
      // Call search API
      return { results: ['...'] };
    },
  }),
};

// In route handler:
const result = streamText({
  model: openai('gpt-4o'),
  messages: await convertToModelMessages(messages),
  tools,
  maxSteps: 5, // allow multi-step tool loops
});
```

## Render tool results in UI (Generative UI)

Tool invocations appear as `tool-{toolName}` parts in messages:

```tsx
{m.parts.map((part, i) => {
  switch (part.type) {
    case 'text':
      return <Markdown key={i}>{part.text}</Markdown>;

    case 'tool-getWeather':
      return (
        <div key={i} className="tool-card">
          {part.state === 'input-available' && (
            <p>🌍 Getting weather for {part.input.location}...</p>
          )}
          {part.state === 'output-available' && (
            <WeatherCard
              location={part.output.location}
              temp={part.output.temp}
              condition={part.output.condition}
            />
          )}
        </div>
      );

    case 'tool-searchWeb':
      return part.state === 'output-available'
        ? <SearchResults results={part.output.results} key={i} />
        : <p key={i}>Searching...</p>;

    default:
      return null;
  }
})}
```

## Tool approval (human-in-the-loop)

```ts
// Backend: require approval before execution
const tools = {
  deleteRecord: tool({
    description: 'Delete a database record',
    inputSchema: z.object({ id: z.string() }),
    needsApproval: true, // user must approve
    execute: async ({ id }) => { /* ... */ },
  }),
};
```

```tsx
// Frontend: handle approval state
{part.type === 'tool-deleteRecord' && part.state === 'input-available' && (
  <div>
    <p>Delete record {part.input.id}?</p>
    <button onClick={() => addToolApprovalResponse({ toolCallId: part.toolCallId, approved: true })}>
      Approve
    </button>
    <button onClick={() => addToolApprovalResponse({ toolCallId: part.toolCallId, approved: false })}>
      Deny
    </button>
  </div>
)}
```

## Streaming structured objects (useObject)

For non-chat structured generation:

```tsx
import { useObject } from '@ai-sdk/react';
import { z } from 'zod';

const schema = z.object({
  title: z.string(),
  tags: z.array(z.string()),
  summary: z.string(),
});

const { object, submit, isLoading } = useObject({
  api: '/api/generate',
  schema,
});

// object is partially filled as it streams in
<p>{object?.title ?? 'Generating...'}</p>
```

## Type-safe tool parts

```ts
// Infer part types from tool definitions
import { InferUITools } from 'ai';
type MyTools = InferUITools<typeof tools>;
// Use for typed access to part.input / part.output
```
