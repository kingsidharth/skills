# Search Index (FAISS & Elasticsearch)

Attach a search index to a `Dataset` to retrieve nearest examples by similarity or exact match.

- **FAISS** — vector similarity (dense embeddings). Good for semantic search, retrieval.
- **Elasticsearch** — inverted text index (BM25). Good for keyword/lexical search.

## FAISS workflow

Compute embeddings → add index → query.

```python
from datasets import load_dataset
from transformers import DPRContextEncoder, DPRContextEncoderTokenizer
import torch
torch.set_grad_enabled(False)

ctx_enc = DPRContextEncoder.from_pretrained("facebook/dpr-ctx_encoder-single-nq-base")
ctx_tok = DPRContextEncoderTokenizer.from_pretrained("facebook/dpr-ctx_encoder-single-nq-base")

ds = load_dataset("community-datasets/crime_and_punish", split="train[:100]")

ds = ds.map(lambda ex: {
    "embeddings": ctx_enc(**ctx_tok(ex["line"], return_tensors="pt"))[0][0].numpy()
})

ds.add_faiss_index(column="embeddings")
```

## Query

```python
from transformers import DPRQuestionEncoder, DPRQuestionEncoderTokenizer

q_enc = DPRQuestionEncoder.from_pretrained("facebook/dpr-question_encoder-single-nq-base")
q_tok = DPRQuestionEncoderTokenizer.from_pretrained("facebook/dpr-question_encoder-single-nq-base")

question = "Is it serious?"
q_emb = q_enc(**q_tok(question, return_tensors="pt"))[0][0].numpy()

scores, examples = ds.get_nearest_examples("embeddings", q_emb, k=10)
examples["line"][0]
```

Batch queries:

```python
scores, examples = ds.get_nearest_examples_batch("embeddings", q_embs, k=10)
```

## Range search / custom FAISS ops

```python
faiss_idx = ds.get_index("embeddings").faiss_index
limits, distances, indices = faiss_idx.range_search(x=q_emb.reshape(1, -1), thresh=0.95)
```

## Persist & reload

```python
ds.save_faiss_index("embeddings", "my_index.faiss")

# Later
ds = load_dataset("community-datasets/crime_and_punish", split="train[:100]")
ds.load_faiss_index("embeddings", "my_index.faiss")
```

Saving/loading the FAISS index is independent of the dataset itself — keep them paired.

## Elasticsearch workflow

Requires a running ES instance.

```python
from datasets import load_dataset

squad = load_dataset("rajpurkar/squad", split="validation")
squad.add_elasticsearch_index("context", host="localhost", port="9200")

scores, examples = squad.get_nearest_examples("context", "machine", k=10)
examples["title"][0]
```

## Persistent ES index (reuse across runs)

```python
squad.add_elasticsearch_index("context", host="localhost", port="9200",
                               es_index_name="hf_squad_val_context")

# Later
squad.load_elasticsearch_index("context", host="localhost", port="9200",
                                es_index_name="hf_squad_val_context")
```

## Custom ES client / config (BM25, analyzers)

```python
from elasticsearch import Elasticsearch

es_client = Elasticsearch([{"host": "localhost", "port": "9200"}])
es_config = {
    "settings": {
        "number_of_shards": 1,
        "analysis": {"analyzer": {"stop_standard": {"type": "standard",
                                                     "stopwords": "_english_"}}},
    },
    "mappings": {"properties": {"text": {"type": "text",
                                          "analyzer": "standard",
                                          "similarity": "BM25"}}},
}
squad.add_elasticsearch_index("context", es_client=es_client,
                               es_config=es_config,
                               es_index_name="hf_squad_context")
```

## When to use what

| Need | Use |
|---|---|
| Semantic similarity ("related in meaning") | FAISS with a sentence encoder |
| Keyword / exact-phrase match | Elasticsearch |
| Both | Add both indexes; fuse scores at query time |
| Very large corpora (>10M docs) with GPU | FAISS with IVF/HNSW configurations — use the raw `faiss_index` and build with FAISS directly, then wrap |
