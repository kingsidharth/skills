---
name: lmstudio-python
description: LM Studio Python SDK (lmstudio package). Chat completions, streaming, structured output, agentic tool use (.act()), embeddings, tokenization, model management, VLM image input. Use when writing Python code that talks to LM Studio, local LLMs via lmstudio SDK, or building agents with local models. Also covers lms CLI, llmster headless daemon, LM Link remote access, and server configuration.
---

# LM Studio Python SDK

`pip install lmstudio` — native SDK for LM Studio's local inference server.

Three API styles exist for every operation; pick one per project:
- **Convenience API** — `lms.llm()`, `lms.embedding_model()` (sync, global client)
- **Scoped resource API** — context managers for deterministic cleanup
- **Async API** — structured concurrency (SDK ≥ 1.5.0)

All examples below use the convenience API. See references for scoped/async variants.

## Quick start

```python
import lmstudio as lms

model = lms.llm("qwen2.5-7b-instruct")
print(model.respond("What is the meaning of life?"))
```

Streaming:
```python
for fragment in model.respond_stream("What is the meaning of life?"):
    print(fragment.content, end="", flush=True)
```

## Choosing a reference

| Need | File |
|---|---|
| Chat, streaming, multi-turn, progress callbacks | [chat-and-streaming.md](references/chat-and-streaming.md) |
| Structured output (Pydantic / JSON schema) | [structured-output.md](references/structured-output.md) |
| Agentic tool use (`.act()`, tool definitions, error handling) | [agentic-tools.md](references/agentic-tools.md) |
| Image input (VLMs) | [image-input.md](references/image-input.md) |
| Embeddings and tokenization | [embeddings-tokenization.md](references/embeddings-tokenization.md) |
| Model management (load, unload, list, TTL, config) | [model-management.md](references/model-management.md) |
| Inference & load parameters | [parameters.md](references/parameters.md) |
| Server, headless daemon, LM Link, CLI | [server-and-cli.md](references/server-and-cli.md) |
