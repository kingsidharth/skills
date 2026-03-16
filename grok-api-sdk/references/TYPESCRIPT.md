# TypeScript / JavaScript / REST Reference

## Setup Options

### OpenAI SDK (recommended for JS/TS)

```bash
npm install openai
```

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.XAI_API_KEY,
  baseURL: "https://api.x.ai/v1",
  timeout: 360_000, // 6 min for reasoning models
});
```

### Vercel AI SDK

```bash
pnpm add ai @ai-sdk/xai
```

```typescript
import { xai } from "@ai-sdk/xai";
import { generateText } from "ai";

const result = await generateText({
  model: xai("grok-4"),
  system: "You are a helpful assistant.",
  prompt: "Hello!",
});
console.log(result.text);
```

### Direct fetch (no SDK)

```typescript
const BASE = "https://api.x.ai";
const headers = {
  Authorization: `Bearer ${process.env.XAI_API_KEY}`,
  "Content-Type": "application/json",
};
```

## Basic Text Generation

### OpenAI SDK — Responses API

```typescript
const response = await client.responses.create({
  model: "grok-4",
  input: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "What is 2+2?" },
  ],
});
console.log(response.output[0].content);
```

### OpenAI SDK — Chat Completions (legacy-compatible)

```typescript
const completion = await client.chat.completions.create({
  model: "grok-4",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "What is 2+2?" },
  ],
});
console.log(completion.choices[0].message.content);
```

### curl

```bash
curl https://api.x.ai/v1/responses \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -m 3600 \
  -d '{
    "model": "grok-4",
    "input": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is 2+2?"}
    ]
  }'
```

## Image Understanding (Vision)

### OpenAI SDK

```typescript
const response = await client.responses.create({
  model: "grok-4",
  input: [
    {
      role: "user",
      content: [
        {
          type: "image_url",
          image_url: {
            url: "https://example.com/photo.jpg",
            detail: "high",
          },
        },
        { type: "text", text: "Describe this image." },
      ],
    },
  ],
});
```

### Vercel AI SDK

```typescript
import { xai } from "@ai-sdk/xai";
import { generateText } from "ai";

const result = await generateText({
  model: xai("grok-4"),
  messages: [
    {
      role: "user",
      content: [
        { type: "image", image: "https://example.com/photo.jpg" },
        { type: "text", text: "What's in this image?" },
      ],
    },
  ],
});
```

### Base64 image from file (Node.js)

```typescript
import { readFileSync } from "fs";

function loadImageB64(path: string): string {
  const data = readFileSync(path);
  const b64 = data.toString("base64");
  const ext = path.split(".").pop()?.toLowerCase();
  const mime = ext === "png" ? "image/png" : "image/jpeg";
  return `data:${mime};base64,${b64}`;
}
```

## Structured Outputs

### OpenAI SDK

```typescript
const response = await client.chat.completions.create({
  model: "grok-4",
  messages: [
    { role: "system", content: "Analyze sentiment." },
    { role: "user", content: "This product is great!" },
  ],
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "SentimentAnalysis",
      schema: {
        type: "object",
        properties: {
          label: {
            type: "string",
            enum: ["positive", "negative", "neutral"],
          },
          confidence: { type: "number" },
          reasoning: { type: "string" },
        },
        required: ["label", "confidence", "reasoning"],
        additionalProperties: false,
      },
    },
  },
});

const result = JSON.parse(response.choices[0].message.content);
```

### OpenAI SDK — beta.parse (Zod)

```typescript
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const Sentiment = z.object({
  label: z.enum(["positive", "negative", "neutral"]),
  confidence: z.number(),
  reasoning: z.string(),
});

const response = await client.beta.chat.completions.parse({
  model: "grok-4",
  messages: [{ role: "user", content: "This is great!" }],
  response_format: zodResponseFormat(Sentiment, "SentimentAnalysis"),
});

const result = response.choices[0].message.parsed;
// result is typed as { label: string, confidence: number, reasoning: string }
```

## Image Generation

### curl

```bash
curl https://api.x.ai/v1/images/generations \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "grok-imagine-image",
    "prompt": "A sunset over mountains",
    "n": 1
  }'
```

## Conversation Chaining (Stateful)

### curl

```bash
# First request
RESPONSE=$(curl -s https://api.x.ai/v1/responses \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "grok-4", "input": [{"role": "user", "content": "What is 2+2?"}]}')

RESPONSE_ID=$(echo $RESPONSE | jq -r '.id')

# Continue conversation
curl https://api.x.ai/v1/responses \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\": \"grok-4\", \"previous_response_id\": \"$RESPONSE_ID\", \"input\": [{\"role\": \"user\", \"content\": \"Multiply that by 10\"}]}"
```

## Encrypted Reasoning (REST)

```json
{
  "model": "grok-4-1-fast-reasoning",
  "include": ["reasoning.encrypted_content"],
  "input": [
    {"role": "user", "content": "Solve step by step: 17 * 23"}
  ]
}
```

## Batch API REST Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/batches` | Create batch |
| `GET` | `/v1/batches` | List batches |
| `GET` | `/v1/batches/{id}` | Get batch status |
| `POST` | `/v1/batches/{id}/requests` | Add requests |
| `GET` | `/v1/batches/{id}/requests` | List requests metadata |
| `GET` | `/v1/batches/{id}/results` | Get results |
| `POST` | `/v1/batches/{id}:cancel` | Cancel batch |

See [BATCH.md](patterns/BATCH.md) for complete workflow examples.
