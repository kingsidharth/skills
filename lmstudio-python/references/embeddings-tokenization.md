# Embeddings & Tokenization

## Embeddings

```python
import lmstudio as lms

model = lms.embedding_model("nomic-embed-text-v1.5")
vector = model.embed("Hello, world!")
```

Get a model: `lms get nomic-ai/nomic-embed-text-v1.5`

## Tokenization

Works on both LLM and embedding model handles.

```python
model = lms.llm()
tokens = model.tokenize("Hello, world!")   # list of token IDs
count = len(tokens)
```

### Check if chat fits in context

```python
def fits_in_context(model: lms.LLM, chat: lms.Chat) -> bool:
    formatted = model.apply_prompt_template(chat)
    token_count = len(model.tokenize(formatted))
    return token_count < model.get_context_length()
```
