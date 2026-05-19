# Inference & Load Parameters

## Inference parameters

Passed via `config` on `.respond()` / `.respond_stream()` / `.act()`:

```python
result = model.respond(chat, config={
    "temperature": 0.6,
    "maxTokens": 50,
    "topP": 0.9,
})
```

Key fields: `temperature`, `maxTokens`, `topP`, `topK`, `repeatPenalty`, `seed`, `stop`.

Full reference: `LLMPredictionConfigInput` in the TypeScript SDK docs (shared schema).

For structured output, prefer `response_format=` over `config["structured"]`.

## Load parameters

Set when loading a model via `.llm()` or `.load_new_instance()`:

```python
model = lms.llm("qwen2.5-7b-instruct", config={
    "contextLength": 8192,
    "gpu": {"ratio": 0.5},
})
```

If the model is already loaded, config is ignored on `.llm()`. Use `.load_new_instance()` to guarantee config applies.

Key fields: `contextLength`, `gpu.ratio` (0.0–1.0), `ropeFrequencyBase`, `ropeFrequencyScale`.

Full reference: `LLMLoadModelConfig` in the TypeScript SDK docs.
