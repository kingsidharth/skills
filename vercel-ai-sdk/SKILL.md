---
name: vercel-ai-sdk
description: Build AI chatbots with streaming, message persistence, tool calling, generative UI, and text-to-speech using Vercel AI SDK (v6) in Vite/React. Use when working with useChat, streamText, AI providers (OpenAI, OpenRouter, fal), or TTS (ElevenLabs, fal).
---

# Vercel AI SDK (v6)

Current stable: **AI SDK 6**. Packages: `ai` (core) + `@ai-sdk/react` (hooks).

## Quick install

```bash
pnpm add ai @ai-sdk/react zod
# Add providers you need:
pnpm add @ai-sdk/openai @ai-sdk/anthropic @openrouter/ai-sdk-provider @ai-sdk/fal @ai-sdk/elevenlabs
```

## Architecture (Vite + React)

AI SDK works in **Vite projects** — but the server-side functions (`streamText`, `generateSpeech`) must run on a backend. Use an Express/Hono server, Vite's proxy to a separate API server, or a serverless function host.

```
Frontend (Vite/React)          Backend (Node/Express/Hono)
┌─────────────────────┐        ┌──────────────────────────┐
│  useChat()          │──POST──▶  streamText()             │
│  @ai-sdk/react      │◀─SSE───│  toUIMessageStreamResponse│
└─────────────────────┘        └──────────────────────────┘
```

## Core hooks (frontend)

```tsx
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

const { messages, sendMessage, status, stop, regenerate } = useChat({
  transport: new DefaultChatTransport({ api: '/api/chat' }),
  id: chatId,               // for persistence
  messages: initialMessages, // preloaded from DB
  resume: false,             // set true to resume interrupted streams
});

// status: 'ready' | 'submitted' | 'streaming' | 'error'
// sendMessage({ text: input })
// stop()         — abort stream
// regenerate()   — redo last assistant message
```

## Core server (backend)

```ts
import { streamText, convertToModelMessages, UIMessage } from 'ai';
import { openai } from '@ai-sdk/openai';

app.post('/api/chat', async (req, res) => {
  const { messages }: { messages: UIMessage[] } = req.body;
  
  const result = streamText({
    model: openai('gpt-4o-mini'),
    messages: await convertToModelMessages(messages),
    system: 'You are a helpful assistant.',
  });

  return result.toUIMessageStreamResponse();
});
```

## Routing

| Feature | Reference |
|---|---|
| Chatbot UI + streaming | [CHATBOT.md](references/CHATBOT.md) |
| Message persistence + history | [PERSISTENCE.md](references/PERSISTENCE.md) |
| Tool calling + Generative UI | [TOOLS.md](references/TOOLS.md) |
| Stream resumption | [RESUME.md](references/RESUME.md) |
| Providers (OpenAI, OpenRouter, Anthropic) | [PROVIDERS.md](references/PROVIDERS.md) |
| Text-to-Speech (fal, ElevenLabs, OpenAI) | [TTS.md](references/TTS.md) |

## Critical rules

- Always use `convertToModelMessages()` before passing to `streamText` — `UIMessage[]` ≠ `ModelMessage[]`
- Store messages in `UIMessage[]` format (not model format)
- Use `validateUIMessages()` when loading from DB if messages contain tools/metadata
- Server-side `generateMessageId` needed for persistence consistency
- `@ai-sdk/react` imports hooks; `ai` imports core functions
