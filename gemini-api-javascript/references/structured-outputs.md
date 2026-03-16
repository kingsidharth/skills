# Structured Outputs

Force Gemini to return responses matching a JSON Schema.

## With Zod (Recommended for TypeScript)

```ts
import { GoogleGenAI } from "@google/genai";
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";

const ai = new GoogleGenAI({});

const recipeSchema = z.object({
  recipe_name: z.string().describe("The name of the recipe."),
  prep_time_minutes: z.number().optional(),
  ingredients: z.array(z.object({
    name: z.string(),
    quantity: z.string(),
  })),
  instructions: z.array(z.string()),
});

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Extract the recipe from: To make cookies, you need 2 cups flour...",
  config: {
    responseMimeType: "application/json",
    responseJsonSchema: zodToJsonSchema(recipeSchema),
  },
});

const recipe = recipeSchema.parse(JSON.parse(response.text));
```

## With Raw JSON Schema

```ts
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Classify this review: 'The UI is great!'",
  config: {
    responseMimeType: "application/json",
    responseJsonSchema: {
      type: "object",
      properties: {
        sentiment: { type: "string", enum: ["positive", "neutral", "negative"] },
        summary: { type: "string" },
      },
      required: ["sentiment", "summary"],
    },
  },
});
```

## Enum Constraints

Use `enum` for classification tasks:

```ts
const schema = z.object({
  category: z.enum(["bug", "feature", "question", "docs"]),
  priority: z.enum(["low", "medium", "high", "critical"]),
  summary: z.string(),
});
```

## Streaming Structured Outputs

Chunks are valid partial JSON; concatenate for the full object:

```ts
const stream = await ai.models.generateContentStream({
  model: "gemini-3-flash-preview",
  contents: "Analyze this feedback...",
  config: {
    responseMimeType: "application/json",
    responseJsonSchema: zodToJsonSchema(feedbackSchema),
  },
});

let fullJson = "";
for await (const chunk of stream) {
  fullJson += chunk.candidates[0].content.parts[0].text;
}
const result = JSON.parse(fullJson);
```

## Structured Outputs + Tools (Gemini 3 only)

Combine JSON output with built-in tools like Google Search:

```ts
const matchSchema = z.object({
  winner: z.string(),
  final_match_score: z.string(),
  scorers: z.array(z.string()),
});

const response = await ai.models.generateContent({
  model: "gemini-3-pro-preview",
  contents: "Search for all details for the latest Euro.",
  config: {
    tools: [{ googleSearch: {} }, { urlContext: {} }],
    responseMimeType: "application/json",
    responseJsonSchema: zodToJsonSchema(matchSchema),
  },
});
```

## Key Notes

- `responseMimeType: "application/json"` is required
- `responseJsonSchema` accepts a JSON Schema object (use `zodToJsonSchema` for Zod)
- Supports recursive structures (for tree-like data)
- Supported schema types: `string`, `number`, `integer`, `boolean`, `array`, `object`, `enum`
- `description` fields on properties improve extraction accuracy
