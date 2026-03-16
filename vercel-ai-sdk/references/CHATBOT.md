# Chatbot UI

## Minimal working chatbot

```tsx
// src/Chat.tsx
'use client';
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';
import { useState } from 'react';

export default function Chat() {
  const [input, setInput] = useState('');
  const { messages, sendMessage, status } = useChat({
    transport: new DefaultChatTransport({ api: '/api/chat' }),
  });

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          <b>{m.role === 'user' ? 'You' : 'AI'}:</b>
          {m.parts.map(part =>
            part.type === 'text' ? <p key={part.type}>{part.text}</p> : null
          )}
        </div>
      ))}

      <form onSubmit={e => { e.preventDefault(); sendMessage({ text: input }); setInput(''); }}>
        <input value={input} onChange={e => setInput(e.target.value)} />
        <button disabled={status !== 'ready'}>Send</button>
      </form>
    </div>
  );
}
```

## Rendering message parts

Messages use a `parts` array — always iterate parts, not `content`:

```tsx
{m.parts.map((part, i) => {
  switch (part.type) {
    case 'text':
      return <p key={i}>{part.text}</p>;
    case 'tool-weatherTool': // tool-{toolName}
      return (
        <div key={i}>
          {part.state === 'output-available'
            ? <WeatherCard data={part.output} />
            : <p>Getting weather...</p>}
        </div>
      );
    default:
      return null;
  }
})}
```

## Status states

```tsx
const { status } = useChat();
// 'ready'     — idle, ready for input
// 'submitted' — request sent, awaiting first token
// 'streaming' — actively receiving chunks
// 'error'     — request failed

<button disabled={status !== 'ready'}>
  {status === 'streaming' ? 'Streaming...' : 'Send'}
</button>
```

## Stop + Regenerate

```tsx
const { stop, regenerate } = useChat({ ... });

<button onClick={stop}>Stop</button>
<button onClick={() => regenerate()}>Retry</button>
// regenerate(options) accepts { messageId } to redo a specific message
```

## Backend (Express/Hono)

```ts
import { streamText, convertToModelMessages, UIMessage } from 'ai';
import { openai } from '@ai-sdk/openai';

app.post('/api/chat', async (req, res) => {
  const { messages }: { messages: UIMessage[] } = req.body;

  const result = streamText({
    model: openai('gpt-4o-mini'),
    messages: await convertToModelMessages(messages),
    maxSteps: 5, // for multi-turn tool loops
  });

  // Must return the stream response
  const response = result.toUIMessageStreamResponse();
  // For Express: pipe response headers + body
  res.set(Object.fromEntries(response.headers.entries()));
  response.body?.pipeTo(
    new WritableStream({ write: chunk => res.write(chunk), close: () => res.end() })
  );
});
```
