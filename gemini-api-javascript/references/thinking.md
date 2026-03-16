# Thinking

Gemini 3 and 2.5 models use internal reasoning ("thinking") before responding.

## Thinking Levels (Gemini 3)

Control reasoning depth via `thinkingLevel`:

| Level | Gemini 3 Pro | Gemini 3 Flash | Description |
|---|---|---|---|
| `minimal` | ❌ | ✅ | Near-zero thinking. May still think on complex code |
| `low` | ✅ | ✅ | Minimizes latency/cost |
| `medium` | ❌ | ✅ | Balanced |
| `high` | ✅ (default) | ✅ (default) | Maximum reasoning depth |

```ts
import { GoogleGenAI } from "@google/genai";
const ai = new GoogleGenAI({});

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "How does AI work?",
  config: {
    thinkingConfig: {
      thinkingLevel: "low",
    },
  },
});
```

Cannot disable thinking entirely on Gemini 3 Pro. `minimal` on Flash means likely no thinking but not guaranteed.

## Thinking Budgets (Gemini 2.5)

For 2.5 series models, use `thinkingBudget` (token count) instead of `thinkingLevel`:

```ts
config: {
  thinkingConfig: {
    thinkingBudget: 4096,  // 0 to disable, -1 for dynamic (default)
  },
}
```

Do not mix `thinkingLevel` and `thinkingBudget` in the same request — returns 400 error.

## Thought Summaries

Enable `includeThoughts` to see summarized reasoning:

```ts
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "What is the sum of the first 50 prime numbers?",
  config: {
    thinkingConfig: { includeThoughts: true },
  },
});

for (const part of response.candidates[0].content.parts) {
  if (!part.text) continue;
  if (part.thought) {
    console.log("Thought:", part.text);
  } else {
    console.log("Answer:", part.text);
  }
}
```

### Streaming thought summaries

```ts
const stream = await ai.models.generateContentStream({
  model: "gemini-3-flash-preview",
  contents: "Solve this logic puzzle...",
  config: { thinkingConfig: { includeThoughts: true } },
});

for await (const chunk of stream) {
  for (const part of chunk.candidates[0].content.parts) {
    if (part.thought) {
      process.stdout.write(`[THOUGHT] ${part.text}`);
    } else if (part.text) {
      process.stdout.write(part.text);
    }
  }
}
```

## Thought Signatures (Gemini 3)

Gemini 3 responses include `thoughtSignature` fields — encrypted tokens that preserve reasoning across turns.

**When they're required:**

| Context | Enforcement | What happens if missing |
|---|---|---|
| Function calling | Strict | 400 error |
| Image generation/editing | Strict | 400 error |
| Text/chat | Not enforced | Degraded reasoning quality |

**The SDK handles signatures automatically** when using `ai.chats.create()`.

For manual history management, return all `thoughtSignature` fields exactly as received:

```ts
// Model responds with a function call + signature
const modelParts = response.candidates[0].content.parts;
// modelParts[0] = { functionCall: {...}, thoughtSignature: "<Sig_A>" }

// When sending function response back, include the signature:
const history = [
  { role: "user", parts: [{ text: "Book a meeting" }] },
  { role: "model", parts: modelParts },  // Preserves thoughtSignature
  { role: "user", parts: [{ functionResponse: { name: "book", response: {...} } }] },
];
```

**Migrating from another model**: Use dummy signature `"context_engineering_is_the_way_to_go"` to bypass validation.

### Parallel function calls

Only the first `functionCall` part has a signature. Return parts in exact order received.

### Sequential (multi-step) function calls

Each step produces its own signature. All accumulated signatures must be returned in the full history.
