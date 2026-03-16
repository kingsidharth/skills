# Stream Resumption

Resume interrupted streams after page reload or network failure.

> **Note:** Stream resumption requires Redis + a persistence layer. Not compatible with `stop()`/abort.

## Frontend

```tsx
const { messages, sendMessage } = useChat({
  id: chatId,
  messages: initialMessages,
  resume: true, // enable auto-resume on mount
  transport: new DefaultChatTransport({ api: `/api/chat/${chatId}` }),
});
```

When `resume: true`, on mount the hook makes a GET to `/api/chat/${chatId}/stream` to reconnect to any active stream.

## Backend setup (requires Redis + resumable-stream package)

```bash
pnpm add resumable-stream ioredis
```

```ts
// POST /api/chat/:id — create resumable stream
import { createResumableStreamContext } from 'resumable-stream';
import { streamText, createUIMessageStream, createUIMessageStreamResponse } from 'ai';

const streamContext = createResumableStreamContext({ waitUntil: after });

app.post('/api/chat/:id', async (req, res) => {
  const { message } = req.body;
  const { id } = req.params;

  const previousMessages = await loadChat(id);
  const messages = [...previousMessages, message];

  const stream = createUIMessageStream({
    execute: async ({ writer }) => {
      const result = streamText({
        model: openai('gpt-4o'),
        messages: await convertToModelMessages(messages),
      });
      writer.merge(result.toUIMessageStream());
    },
    onFinish: ({ messages }) => saveChat({ chatId: id, messages }),
  });

  // Create resumable stream and save streamId to DB
  const { streamId, response } = await streamContext.resumableStream(stream);
  await saveActiveStreamId(id, streamId); // save to your DB

  return response;
});

// GET /api/chat/:id/stream — resume active stream
app.get('/api/chat/:id/stream', async (req, res) => {
  const { id } = req.params;
  const streamId = await getActiveStreamId(id); // load from DB

  if (!streamId) {
    return res.status(404).json({ error: 'No active stream' });
  }

  const stream = await streamContext.getResumableStream(streamId);
  if (!stream) {
    return res.status(404).json({ error: 'Stream expired' });
  }

  return createUIMessageStreamResponse({ stream });
});
```

## Key requirements

- Redis to store stream data
- DB to store `activeStreamId` per chat
- `after()` equivalent to continue work after response sent (in Next.js: `after` from next/server; in Express: fire-and-forget)
