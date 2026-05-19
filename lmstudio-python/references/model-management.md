# Model Management

## Namespaces

- `lms.llm()` / `client.llm` — LLM models
- `lms.embedding_model()` / `client.embedding` — embedding models

## Get a model handle

```python
model = lms.llm()                          # any loaded model
model = lms.llm("qwen2.5-7b-instruct")    # specific; JIT loads if not loaded
```

## Load a new instance

Forces a fresh load even if already loaded:
```python
client = lms.get_default_client()
model = client.llm.load_new_instance("qwen2.5-7b-instruct")
model2 = client.llm.load_new_instance("qwen2.5-7b-instruct", "my-id")
```

Providing a duplicate identifier raises an error. Omit to auto-generate.

## Unload

```python
model.unload()
```

## List models

```python
downloaded = lms.list_downloaded_models()          # all
llms_only = lms.list_downloaded_models("llm")
emb_only = lms.list_downloaded_models("embedding")
```

Results have `.model()` and `.load_new_instance()` for converting to handles.

## TTL (auto-unload)

```python
model = lms.llm("qwen2.5-7b-instruct", ttl=3600)  # seconds idle before unload
```

TTL only applies if `.llm()` JIT-loads; ignored for already-loaded models.

## Load config

See [parameters.md](parameters.md) for `contextLength`, `gpu.ratio`, etc.
