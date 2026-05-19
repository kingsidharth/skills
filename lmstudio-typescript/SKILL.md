---
name: lmstudio-typescript
description: LM Studio TypeScript SDK (@lmstudio/sdk), lmstudio-js, local LLM inference, chat completions, structured output, tool use, agentic .act(), plugins, embeddings, model loading, LM Link, lms CLI, llmster headless daemon
---

# LM Studio TypeScript SDK

Package: `@lmstudio/sdk` (npm). Connects to a running LM Studio instance (desktop app or `llmster` daemon) over local WebSocket.

## Install

```
npm install @lmstudio/sdk
```

## Quickstart

```ts
import { LMStudioClient } from "@lmstudio/sdk";
const client = new LMStudioClient();
const model = await client.llm.model("qwen/qwen3-4b-2507");
const result = await model.respond("Hello");
console.info(result.content);
```

## Reference routing

- **Chat completions, streaming, multi-turn** → [references/chat-completions.md](references/chat-completions.md)
- **Structured output (zod / JSON schema)** → [references/structured-output.md](references/structured-output.md)
- **Image/vision input** → [references/image-input.md](references/image-input.md)
- **Tool definition & agentic `.act()`** → [references/agentic-tools.md](references/agentic-tools.md)
- **Plugins (tools provider, preprocessor, generator)** → [references/plugins.md](references/plugins.md)
- **Embeddings** → [references/embeddings.md](references/embeddings.md)
- **Model management (load/unload/list/info)** → [references/model-management.md](references/model-management.md)
- **Configuration (inference & load params)** → [references/configuration.md](references/configuration.md)
- **Infrastructure (headless, LM Link, CLI)** → [references/infrastructure.md](references/infrastructure.md)

## Key conventions

- `client.llm.model()` — get any loaded model; `client.llm.model("key")` — JIT-load specific model
- `client.llm.load("key")` — force-load a new instance
- `.respond()` returns an async iterable (streaming) or awaitable (non-streaming via `.result()`)
- Auth: set `LM_API_TOKEN` env var or pass `apiToken` to constructor
- Project scaffold: `lms create node-typescript`
