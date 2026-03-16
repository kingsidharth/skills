# Text Generation

## Basic Generation

```ts
import { GoogleGenAI } from "@google/genai";
const ai = new GoogleGenAI({});

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Explain how AI works in a few words",
});
console.log(response.text);
```

## System Instructions

```ts
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "What should I eat today?",
  config: {
    systemInstruction: "You are a nutritionist. Recommend healthy options.",
  },
});
```

## Streaming

```ts
const stream = await ai.models.generateContentStream({
  model: "gemini-3-flash-preview",
  contents: "Write a poem about the ocean",
});

for await (const chunk of stream) {
  process.stdout.write(chunk.text ?? "");
}
```

## Multi-turn Chat

The SDK manages conversation history automatically via `ai.chats.create()`:

```ts
const chat = ai.chats.create({
  model: "gemini-3-flash-preview",
  config: {
    systemInstruction: "You are a helpful assistant.",
  },
});

const response1 = await chat.sendMessage({ message: "Hello, my name is Alex." });
console.log(response1.text);

const response2 = await chat.sendMessage({ message: "What's my name?" });
console.log(response2.text); // Will remember "Alex"
```

Chat with streaming:

```ts
const stream = await chat.sendMessageStream({ message: "Tell me a story" });
for await (const chunk of stream) {
  process.stdout.write(chunk.text ?? "");
}
```

## Generation Config Options

```ts
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Write a haiku",
  config: {
    temperature: 1.0,          // Gemini 3: keep at 1.0
    topP: 0.95,
    topK: 40,
    maxOutputTokens: 1024,
    stopSequences: ["END"],
    candidateCount: 1,
  },
});
```

## Manual History Management

If you need control over the conversation history instead of using chat:

```ts
const history = [
  { role: "user", parts: [{ text: "Hello" }] },
  { role: "model", parts: [{ text: "Hi! How can I help?" }] },
  { role: "user", parts: [{ text: "Tell me about Gemini" }] },
];

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: history,
});
```

For Gemini 3 models, preserve `thoughtSignature` on model parts when managing history manually — see [thinking.md](thinking.md).
