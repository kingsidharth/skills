# Message Persistence

## Pattern: send only last message (efficient)

**Frontend** — send only the new message, backend loads history:

```tsx
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

const { messages, sendMessage } = useChat({
  id: chatId,
  messages: initialMessages, // loaded from DB at page load
  transport: new DefaultChatTransport({
    api: '/api/chat',
    prepareSendMessagesRequest({ messages, id }) {
      // Only send the last message
      return { body: { message: messages[messages.length - 1], id } };
    },
  }),
});
```

**Backend** — load history, append new message, save on finish:

```ts
import {
  convertToModelMessages, streamText, UIMessage,
  validateUIMessages, createIdGenerator,
} from 'ai';

app.post('/api/chat', async (req, res) => {
  const { message, id } = req.body;

  // Load previous messages from your DB/store
  const previousMessages: UIMessage[] = await loadChat(id);
  const messages = [...previousMessages, message];

  // Validate if messages contain tools or custom metadata
  const validatedMessages = await validateUIMessages({ messages, tools });

  const result = streamText({
    model: openai('gpt-4o-mini'),
    messages: await convertToModelMessages(validatedMessages),
    tools,
  });

  const response = result.toUIMessageStreamResponse({
    originalMessages: messages,
    // Server-side IDs for persistence consistency
    generateMessageId: createIdGenerator({ prefix: 'msg', size: 16 }),
    onFinish: ({ messages }) => {
      saveChat({ chatId: id, messages }); // save complete message array
    },
  });

  // pipe to res...
});
```

## Chat store interface (file-based example, replace with DB)

```ts
import { UIMessage, generateId } from 'ai';

// Create new chat
export async function createChat(): Promise<string> {
  const id = generateId();
  await writeFile(chatFile(id), '[]');
  return id;
}

// Load messages
export async function loadChat(id: string): Promise<UIMessage[]> {
  return JSON.parse(await readFile(chatFile(id), 'utf8'));
}

// Save messages
export async function saveChat({ chatId, messages }: {
  chatId: string;
  messages: UIMessage[];
}) {
  await writeFile(chatFile(chatId), JSON.stringify(messages, null, 2));
}
```

## Page setup (load chat on route)

```tsx
// In your route component
export async function loader({ params }) {
  const messages = await loadChat(params.chatId);
  return { chatId: params.chatId, messages };
}

// Chat component receives initialMessages
function ChatPage({ chatId, messages: initialMessages }) {
  const { messages, sendMessage } = useChat({
    id: chatId,
    messages: initialMessages,
    transport: ...
  });
  // ...
}
```

## Handle client disconnects

To prevent broken state when users close the tab mid-stream:

```ts
const result = streamText({ ... });

// consumeStream removes backpressure — stream continues on backend even if client disconnects
result.consumeStream();

return result.toUIMessageStreamResponse({
  onFinish: ({ messages }) => saveChat({ chatId, messages }),
});
```

## Key concepts

- Store in **UIMessage[]** format (has `id`, `role`, `parts`, `createdAt`)
- `UIMessage` ≠ `ModelMessage` — always `convertToModelMessages()` before `streamText`
- Use `validateUIMessages({ messages, tools })` when loading from DB (guards against schema drift)
- Use server-side `generateMessageId` so IDs are consistent across sessions
