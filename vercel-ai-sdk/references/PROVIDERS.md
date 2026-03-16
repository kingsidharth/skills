# Providers

## OpenAI

```ts
import { openai } from '@ai-sdk/openai';
// Env: OPENAI_API_KEY

// Chat models
openai('gpt-4o')
openai('gpt-4o-mini')
openai('gpt-4.1')

// Embeddings
openai.textEmbeddingModel('text-embedding-3-small')

// Via OpenAI API — also accepts any model string via Responses API
```

## Anthropic

```ts
import { anthropic } from '@ai-sdk/anthropic';
// Env: ANTHROPIC_API_KEY

anthropic('claude-opus-4-5')
anthropic('claude-sonnet-4-5')
anthropic('claude-haiku-4-5-20251001')
```

## OpenRouter (300+ models, one API key)

```ts
import { createOpenRouter } from '@openrouter/ai-sdk-provider';
// pnpm add @openrouter/ai-sdk-provider
// Env: OPENROUTER_API_KEY

const openrouter = createOpenRouter({ apiKey: process.env.OPENROUTER_API_KEY });

// Drop-in replacement for any provider
openrouter.chat('anthropic/claude-sonnet-4')
openrouter.chat('openai/gpt-4o')
openrouter.chat('meta-llama/llama-3.1-70b-instruct')
openrouter.chat('google/gemini-2.0-flash')

// Embeddings
openrouter.textEmbeddingModel('openai/text-embedding-3-small')
```

## Querying via OpenAI-compatible API

Any OpenAI-compatible endpoint (OpenRouter, local models, etc.) works with `createOpenAI`:

```ts
import { createOpenAI } from '@ai-sdk/openai';

// OpenRouter via OpenAI compat (alternative to @openrouter/ai-sdk-provider)
const openrouter = createOpenAI({
  baseURL: 'https://openrouter.ai/api/v1',
  apiKey: process.env.OPENROUTER_API_KEY,
});
openrouter('anthropic/claude-sonnet-4')

// Local Ollama
const ollama = createOpenAI({
  baseURL: 'http://localhost:11434/v1',
  apiKey: 'ollama',
});
ollama('llama3.2')

// LM Studio
const lmstudio = createOpenAI({
  baseURL: 'http://localhost:1234/v1',
  apiKey: 'lmstudio',
});
```

## Vercel AI Gateway (all providers, one key)

```ts
import { generateText } from 'ai';
// Env: AI_GATEWAY_API_KEY

// Use model strings directly — no provider import needed
const result = await generateText({
  model: 'openai/gpt-4o',
  prompt: 'Hello',
});

// Also works with anthropic/, google/, meta-llama/, etc.
```

## Provider in streamText

```ts
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

const result = streamText({
  model: openai('gpt-4o-mini'),
  // model: openrouter.chat('anthropic/claude-sonnet-4'),
  // model: 'anthropic/claude-sonnet-4', // via AI Gateway
  messages: await convertToModelMessages(messages),
  system: 'You are a helpful assistant.',
  temperature: 0.7,
  maxTokens: 1000,
});
```

## Switching providers at runtime

```ts
function getModel(provider: string) {
  switch (provider) {
    case 'openai': return openai('gpt-4o');
    case 'anthropic': return anthropic('claude-sonnet-4-5');
    case 'openrouter': return openrouter.chat('meta-llama/llama-3.1-70b-instruct');
    default: return openai('gpt-4o-mini');
  }
}
```
