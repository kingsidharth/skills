# Embeddings

LanceDB can auto-vectorize data at ingestion and query time via the embedding registry. In TypeScript, this is primarily a Python-SDK feature — the TS SDK requires you to compute embeddings externally and pass vectors directly.

## TypeScript pattern

Compute embeddings externally (OpenAI, Cohere, local model, etc.) and include the `vector` column in your data:

```typescript
import OpenAI from "openai";
const openai = new OpenAI();

async function embed(text: string): Promise<number[]> {
  const res = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: text,
  });
  return res.data[0].embedding;
}

// At ingestion
const data = await Promise.all(items.map(async (item) => ({
  ...item,
  vector: await embed(item.text),
})));
await table.add(data);

// At query time
const queryVec = await embed("search query");
const results = await table.search(queryVec).limit(10).toArray();
```

## Python embedding registry (for reference)

Python SDK has built-in auto-embedding via `get_registry()`:

```python
from lancedb.embeddings import get_registry
from lancedb.pydantic import LanceModel, Vector

model = get_registry().get("openai").create(name="text-embedding-3-small")

class Doc(LanceModel):
    text: str = model.SourceField()
    vector: Vector(model.ndims()) = model.VectorField()

table = db.create_table("docs", schema=Doc)
table.add([{"text": "hello"}])  # auto-embeds
table.search("hello").limit(5)  # auto-embeds query
```

Supported providers: OpenAI, Cohere, Sentence Transformers, Hugging Face, Gemini, Ollama, AWS Bedrock, Jina, VoyageAI, CLIP, ImageBind, and more.

## Embedding with merge_insert

During `mergeInsert`, if the input data contains the source field but not the vector field, embeddings are auto-generated. If a vector is already provided, embedding generation is skipped.
