# Configuration

## Inference parameters

Passed as second argument to `.respond()` or `.complete()`:

```ts
const prediction = model.respond(chat, {
  temperature: 0.6,
  maxTokens: 50,
  topP: 0.9,
  structured: zodSchemaOrJsonSchema,
});
```

Full field list: see `LLMPredictionConfigInput` in SDK types.

## Load parameters

Set when loading a model — ignored if model already loaded via `.model()`.

```ts
const model = await client.llm.model("qwen2.5-7b-instruct", {
  config: {
    contextLength: 8192,
    gpu: { ratio: 0.5 },
  },
});
```

Or via `.load()`:

```ts
const model = await client.llm.load("qwen2.5-7b-instruct", {
  config: {
    contextLength: 8192,
    gpu: { ratio: 0.5 },
  },
});
```

Full field list: see `LLMLoadModelConfig` in SDK types.

## Authentication

Two methods:

1. **Env var (recommended):** `export LM_API_TOKEN="your-token"` — SDK reads automatically
2. **Constructor arg:** `new LMStudioClient({ apiToken: "your-token" })`

Auth is disabled by default in LM Studio. Enable in app settings or via `lms server` for shared/production environments.
