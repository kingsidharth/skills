# Safety Settings and Configuration

## Safety Settings

Filter content by category and threshold:

```ts
import { GoogleGenAI } from "@google/genai";

const ai = new GoogleGenAI({});

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Write a story...",
  config: {
    safetySettings: [
      { category: "HARM_CATEGORY_HARASSMENT", threshold: "BLOCK_ONLY_HIGH" },
      { category: "HARM_CATEGORY_HATE_SPEECH", threshold: "BLOCK_ONLY_HIGH" },
      { category: "HARM_CATEGORY_SEXUALLY_EXPLICIT", threshold: "BLOCK_MEDIUM_AND_ABOVE" },
      { category: "HARM_CATEGORY_DANGEROUS_CONTENT", threshold: "BLOCK_ONLY_HIGH" },
    ],
  },
});
```

### Categories

| Category | Description |
|---|---|
| `HARM_CATEGORY_HARASSMENT` | Harassment content |
| `HARM_CATEGORY_HATE_SPEECH` | Hate speech |
| `HARM_CATEGORY_SEXUALLY_EXPLICIT` | Sexual content |
| `HARM_CATEGORY_DANGEROUS_CONTENT` | Dangerous content |
| `HARM_CATEGORY_CIVIC_INTEGRITY` | Civic integrity |

### Thresholds

| Threshold | Behavior |
|---|---|
| `BLOCK_NONE` | Always show (may still be filtered for some categories) |
| `BLOCK_ONLY_HIGH` | Block only high-probability harmful content |
| `BLOCK_MEDIUM_AND_ABOVE` | Block medium and above |
| `BLOCK_LOW_AND_ABOVE` | Block low and above (most restrictive) |

### Check Safety Ratings in Response

```ts
if (response.candidates?.[0]?.finishReason === "SAFETY") {
  console.log("Response blocked by safety filters");
  console.log(response.candidates[0].safetyRatings);
}
```

## Token Counting

Count tokens before sending requests:

```ts
const result = await ai.models.countTokens({
  model: "gemini-3-flash-preview",
  contents: "How many tokens is this text?",
});
console.log(`Total tokens: ${result.totalTokens}`);
```

With images/files:

```ts
const result = await ai.models.countTokens({
  model: "gemini-3-flash-preview",
  contents: [
    { text: "Describe this image" },
    { inlineData: { mimeType: "image/jpeg", data: base64Image } },
  ],
});
```

## Context Caching

Cache large content to reduce costs on repeated use:

```ts
const cache = await ai.caches.create({
  model: "gemini-3-flash-preview",
  config: {
    contents: [{ role: "user", parts: [{ text: largeDocument }] }],
    ttl: "3600s",  // 1 hour
    displayName: "my-cache",
  },
});

// Use cached content
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Summarize the cached document",
  config: { cachedContent: cache.name },
});
```

## Vercel AI SDK Integration

Use Gemini with the Vercel AI SDK:

```ts
import { google } from "@ai-sdk/google";
import { generateText, streamText } from "ai";

const result = await generateText({
  model: google("gemini-3-flash-preview"),
  prompt: "Explain quantum computing",
});

// Streaming
const stream = streamText({
  model: google("gemini-3-flash-preview"),
  prompt: "Write a story",
});
for await (const chunk of stream.textStream) {
  process.stdout.write(chunk);
}
```

Install: `npm install @ai-sdk/google ai`

## OpenAI Compatibility

Gemini supports OpenAI-compatible endpoints:

```ts
import OpenAI from "openai";

const openai = new OpenAI({
  apiKey: process.env.GEMINI_API_KEY,
  baseURL: "https://generativelanguage.googleapis.com/v1beta/openai/",
});

const response = await openai.chat.completions.create({
  model: "gemini-3-flash-preview",
  messages: [{ role: "user", content: "Explain AI" }],
});
```

Maps: `reasoning_effort` → `thinking_level`, `medium` → `high` on Gemini 3 Flash.
