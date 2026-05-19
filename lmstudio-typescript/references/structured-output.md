# Structured Output

Enforce model output to conform to a schema. Two approaches: `zod` (recommended, typed) or raw JSON Schema.

## With zod

```ts
import { z } from "zod";

const bookSchema = z.object({
  title: z.string(),
  author: z.string(),
  year: z.number().int(),
});

const result = await model.respond("Tell me about The Hobbit.", {
  structured: bookSchema,
  maxTokens: 100,
});

const book = result.parsed;
// typed as { title: string; author: string; year: number }
```

## With JSON Schema

```ts
const schema = {
  type: "object",
  properties: {
    title: { type: "string" },
    author: { type: "string" },
    year: { type: "integer" },
  },
  required: ["title", "author", "year"],
};

const result = await model.respond("Tell me about The Hobbit.", {
  structured: { type: "json", jsonSchema: schema },
  maxTokens: 100,
});

const book = JSON.parse(result.content);
```

## Caveats

- Always set `maxTokens` — smaller models can get stuck in unclosed structures
- Schema compliance only guaranteed for complete generations; interrupted output will likely violate the schema
- With zod input, incomplete generation raises an error; with JSON Schema, you get an invalid string
