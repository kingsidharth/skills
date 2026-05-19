# Embeddings

Generate vector representations of text for RAG and similarity tasks.

## Usage

```ts
const model = await client.embedding.model("nomic-embed-text-v1.5");
const { embedding } = await model.embed("Hello, world!");
```

Download an embedding model:

```bash
lms get nomic-ai/nomic-embed-text-v1.5
```

The `embedding` namespace mirrors `llm` for model management — `.model()`, `.load()`, etc. all work identically.
