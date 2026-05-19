---
name: huggingface-datasets
description: When using the `datasets` library, `huggingface_hub`, or working with datasets on the HuggingFace Hub. Covers loading, streaming, resumable download, partial access, discovery, creation, upload, folder builders, WebDataset, data cards, caching, FAISS, and integrations.
---

# HuggingFace Datasets

Assume Python 3.10+ with `uv`, `torch` available. Parquet is the default serialization format.

```bash
uv add datasets huggingface_hub
```

## Route by intent

| Intent | File |
|---|---|
| Load from Hub, local files (csv/json/parquet/sql), or inspect without downloading | [LOADING.md](references/LOADING.md) |
| Stream without downloading; column/filter pushdown; slice; shard; interleave; DataLoader | [STREAMING.md](references/STREAMING.md) |
| **Resume a stream from a cursor after crash/restart**; checkpoint position; resumable downloads | [RESUMABLE.md](references/RESUMABLE.md) |
| Download specific files (`hf_hub_download`, `snapshot_download`, `HfFileSystem`, dry-run) | [FILE_ACCESS.md](references/FILE_ACCESS.md) |
| Discover datasets — `list_datasets` with filters (tags, rows, size, license, author); `dataset_info` | [DISCOVERY.md](references/DISCOVERY.md) |
| WebDataset format — TAR shards, streaming at scale, `webdataset` lib + native builder | [WEBDATASET.md](references/WEBDATASET.md) |
| ImageFolder / AudioFolder / VideoFolder — dir layout, metadata, captions, bbox | [FOLDER_BUILDERS.md](references/FOLDER_BUILDERS.md) |
| `.map` / `.filter` / `.cast` / features / formats (torch, numpy, pandas, polars, pyarrow) | [PROCESSING.md](references/PROCESSING.md) |
| `push_to_hub`, `upload_folder`, `Dataset.from_*`, creating a dataset repo | [UPLOADING.md](references/UPLOADING.md) |
| README YAML — configs, splits, tags, auto-detection rules | [DATA_CARD.md](references/DATA_CARD.md) |
| FAISS / Elasticsearch search index on a dataset | [SEARCH_INDEX.md](references/SEARCH_INDEX.md) |
| Cache dirs, `HF_HOME`, `HF_HUB_CACHE`, `HF_DATASETS_CACHE`, offline mode, `download_mode` | [CACHE.md](references/CACHE.md) |
| Polars / DuckDB / Pandas / Daft / PyArrow reading from `hf://` | [INTEGRATIONS.md](references/INTEGRATIONS.md) |
| CLI commands (`hf`, `datasets-cli`) | [CLI.md](references/CLI.md) |
| Pickling errors, auth, 429s, upload retry | [TROUBLESHOOTING.md](references/TROUBLESHOOTING.md) |

## Core mental model

- **`Dataset`** — Arrow-backed, random access, cached locally. Random access, multi-epoch, `len()`.
- **`IterableDataset`** — streamed, no cache, forward-only iteration. Needed when dataset > disk or for single-pass exploration.
- **Parquet** — default on Hub. Enables `columns=[...]` and `filters=[...]` pushdown (do not download what you don't use).
- **Shards** — a dataset is usually many Parquet/TAR files. Many features (parallelism, resharding, DataLoader workers) depend on shard count.

## Minimal patterns

```python
from datasets import load_dataset, Dataset

# Hub (cached)
ds = load_dataset("user/repo", split="train")

# Hub, stream (no download, no cache)
ds = load_dataset("user/repo", split="train", streaming=True)

# Hub, partial — columns + filters pushdown + slice
ds = load_dataset("user/repo", split="train[:1%]",
                  columns=["text", "label"],
                  filters=[("score", ">=", 0.9)])

# Local files
ds = load_dataset("parquet", data_files="data/*.parquet", split="train")

# In-memory
ds = Dataset.from_dict({"text": [...], "label": [...]})

# Push
ds.push_to_hub("user/my-dataset")
```

For anything beyond these four patterns, route to a reference file above.

## Progressive loading ladder

When a dataset is large, start at the top and only go lower if you actually need more:

1. `load_dataset_builder(repo).info` — metadata only, zero download. Rows, size, features, splits. See [DISCOVERY.md](references/DISCOVERY.md).
2. `split="train[:100]"` — tiny sample for schema inspection. See [LOADING.md](references/LOADING.md).
3. `streaming=True` + `columns=[...]` + `filters=[...]` — pushdown everything possible. See [STREAMING.md](references/STREAMING.md).
4. `streaming=True` with checkpointing — long runs; resume after crash. See [RESUMABLE.md](references/RESUMABLE.md).
5. Full `load_dataset()` — only when you need random access or multi-epoch training on data small enough to cache.
