# Model Management

## Get a model handle

```ts
// Any currently loaded model
const model = await client.llm.model();

// Specific model (JIT-loads if not loaded)
const model = await client.llm.model("qwen2.5-7b-instruct");
```

## Force-load a new instance

```ts
const model = await client.llm.load("qwen/qwen3-4b-2507");

// With custom identifier
const model = await client.llm.load("qwen/qwen3-4b-2507", {
  identifier: "my-instance",
});
```

Duplicate identifiers throw an error. Omit to auto-generate.

## Unload

```ts
await model.unload();
```

## Auto-unload (TTL)

```ts
const model = await client.llm.load("qwen/qwen3-4b-2507", {
  ttl: 300, // seconds of idle time before auto-unload
});
```

Also works via `.model()`:

```ts
const model = await client.llm.model("qwen/qwen3-4b-2507", { ttl: 300 });
```

## List downloaded models

```ts
const models = await client.system.listDownloadedModels();
```

Returns array with `type` (`"llm"` | `"embedding"`), `modelKey`, `displayName`, `path`, `sizeBytes`, `architecture`, `maxContextLength`, `vision`, `trainedForToolUse`.

## Model info

```ts
const info = await model.getInfo();
info.modelKey;
info.trainedForToolUse;
model.contextLength; // direct property
```

## Context length

```ts
const ctxLen = await model.getContextLength();
```

### Check if chat fits in context

```ts
const formatted = await model.applyPromptTemplate(chat);
const tokenCount = await model.countTokens(formatted);
const contextLength = await model.getContextLength();
const fits = tokenCount < contextLength;
```
