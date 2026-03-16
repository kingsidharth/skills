---
name: huggingface-datasets
description: "Work with HuggingFace datasets — loading, creating, uploading, streaming, partial downloading, and configuring data cards. Use this skill whenever the user mentions HuggingFace datasets, the `datasets` library, `load_dataset`, dataset repos on the Hub, data cards/README YAML configs, streaming large datasets, or partial/selective downloading of dataset files. Also trigger when the user wants to push data to the Hub, create ImageFolder/AudioFolder/VideoFolder datasets, use WebDataset format, or work with `huggingface_hub` for file-level dataset access. Trigger even if the user just says 'dataset' in the context of ML training pipelines — HuggingFace datasets is the de-facto standard."
---

# HuggingFace Datasets Skill

## Quick Decision Tree

| User wants to... | Pattern |
|---|---|
| Load from Hub | `load_dataset("repo/name")` |
| Load local files | `load_dataset("csv"/"json"/"parquet", data_files=...)` |
| Stream without downloading | `load_dataset(..., streaming=True)` |
| Load only some columns | `load_dataset(..., columns=["col1", "col2"])` |
| Load a slice of a split | `load_dataset(..., split="train[:10%]")` |
| Filter at storage level (Parquet) | `load_dataset(..., filters=[("col", ">=", val)])` |
| Download a single file from a repo | `hf_hub_download(repo_id, filename, repo_type="dataset")` |
| Browse repo without downloading | `HfFileSystem().ls("datasets/repo")` |
| Create from local images/audio/video | Use folder-based builders: `imagefolder`, `audiofolder`, `videofolder` |
| Create from Python data | `Dataset.from_dict()`, `from_list()`, `from_generator()`, `from_pandas()` |
| Push to Hub | `ds.push_to_hub("user/repo")` |
| Configure splits/subsets in YAML | Edit `README.md` front-matter `configs` block |

## Core API

```python
from datasets import load_dataset, Dataset, IterableDataset, load_from_disk

# Hub
ds = load_dataset("user/repo", split="train")
ds = load_dataset("user/repo", "config_name", split="train")

# Local files (csv, json, jsonl, parquet, text, arrow, hdf5)
ds = load_dataset("parquet", data_files={"train": "data/train-*.parquet"})

# Streaming (returns IterableDataset — no download, no cache)
ds = load_dataset("big/repo", split="train", streaming=True)

# In-memory
ds = Dataset.from_dict({"text": [...], "label": [...]})
ds = Dataset.from_generator(my_generator_fn)
ds = Dataset.from_pandas(df)
```

## Critical Patterns

### Partial Loading (avoid downloading everything)

This is the single most important pattern for large-scale work. Six complementary strategies exist — combine them freely:

1. **`streaming=True`** — data fetched on-the-fly during iteration, nothing cached
2. **`columns=["col1"]`** — Parquet columnar pushdown, unselected columns never leave the server
3. **`filters=[("col", ">=", val)]`** — Parquet predicate pushdown, skips entire row groups
4. **`split="train[:10%]"`** or `split="train[100:200]"` — load a slice
5. **`data_files="shard-0000*.parquet"`** — glob specific shards only
6. **`.take(N)` / `.skip(N)`** on IterableDataset — grab first N or skip past N

For detailed examples and advanced Parquet tuning, see [PARTIAL_LOADING.md](references/PARTIAL_LOADING.md).

### File-Level Access (download individual files, not whole datasets)

```python
from huggingface_hub import hf_hub_download, snapshot_download, HfFileSystem

# Single file
path = hf_hub_download(repo_id="user/repo", filename="data/train.parquet", repo_type="dataset")

# Selective download with glob filters
snapshot_download(repo_id="user/repo", repo_type="dataset",
                  allow_patterns=["*.parquet"], ignore_patterns=["test*"])

# Browse/read without downloading (fsspec interface)
fs = HfFileSystem()
fs.ls("datasets/user/repo", detail=False)
with fs.open("datasets/user/repo/config.json") as f:
    content = f.read()
```

For full reference including CLI, dry-run, and `local_dir`, see [FILE_ACCESS.md](references/FILE_ACCESS.md).

### Folder-Based Builders (images, audio, video)

Directory structure → automatic splits + labels:

```
data/train/cats/img001.png
data/train/dogs/img002.png
data/test/cats/img003.png
```

```python
ds = load_dataset("imagefolder", data_dir="data")  # auto: split=train/test, label=cats/dogs
```

Add metadata via `metadata.csv` or `metadata.jsonl` with a `file_name` column. For multi-file references per row, use `input_file_name`/`output_file_name` or `*_file_names` for lists.

Identical pattern for `audiofolder` and `videofolder`.

For WebDataset (TAR-based, for scale), captioning metadata, and object detection patterns, see [MODALITIES.md](references/MODALITIES.md).

### Data Card YAML Configuration

Every dataset repo has a `README.md`. The YAML front-matter controls Hub behavior:

```yaml
---
license: apache-2.0
task_categories:
  - image-classification
language:
  - en
pretty_name: My Dataset
size_categories:
  - 100K<n<1M
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
  - split: test
    path: "data/test-*.parquet"
  default: true
---
```

Multiple configs = multiple subsets loadable via `load_dataset("repo", "config_name")`. Builder params like `sep` for CSV can be set per-config.

Auto-detection works without YAML: files named `train.csv`/`test.csv`, or placed in `train/`/`test/` directories, are inferred automatically.

For the full tag reference and advanced config patterns, see [DATA_CARD.md](references/DATA_CARD.md).

## Processing & Transforms

```python
# Column ops
ds = ds.rename_column("old", "new")
ds = ds.remove_columns(["col1", "col2"])
ds = ds.select_columns(["text", "label"])

# Row ops
ds = ds.sort("label")
ds = ds.shuffle(seed=42)
ds = ds.select([0, 10, 20])                              # by index
ds = ds.filter(lambda ex: ex["score"] > 0.9)              # by predicate
ds = ds.train_test_split(test_size=0.1)                    # create splits
ds = ds.shard(num_shards=4, index=0)                       # deterministic sharding

# map() — the core transform
ds = ds.map(lambda ex: {"upper": ex["text"].upper()})      # per-example
ds = ds.map(tokenize_fn, batched=True, batch_size=1000)    # batched (faster)
ds = ds.map(my_fn, num_proc=8)                             # parallel
ds = ds.map(gpu_fn, batched=True, with_rank=True,          # multi-GPU
            num_proc=torch.cuda.device_count())

# Type casting
from datasets import Audio, Image, ClassLabel, Value
ds = ds.cast_column("audio", Audio(sampling_rate=16000))
ds = ds.cast_column("image", Image())
```

## Saving & Sharing

```python
# Local
ds.save_to_disk("path/to/dir")
ds = load_from_disk("path/to/dir")

# Export
ds.to_parquet("out.parquet")
ds.to_csv("out.csv")
ds.to_json("out.jsonl")

# Hub
ds.push_to_hub("user/my-dataset")
ds.push_to_hub("user/my-dataset", config_name="v2", split="train")
```

## Performance Tips

- **Parquet is the recommended format** — columnar, compressed, predicate pushdown
- Use `num_proc=N` on `load_dataset()` and `.map()` for parallelism
- After `.shuffle()`, call `.flatten_indices()` to restore contiguous reads (10x speedup)
- For DataLoader: convert to `IterableDataset` with sharding for multi-worker support:
  ```python
  ids = ds.to_iterable_dataset(num_shards=128)
  ids = ids.shuffle(seed=42, buffer_size=10_000)
  loader = DataLoader(ids.with_format("torch"), num_workers=4)
  ```
- Inspect without downloading: `load_dataset_builder("repo").info`
- Use `.with_format("torch")` / `.set_format("torch")` for zero-copy tensor access

## Common Gotcha: `total_mem` vs `total_memory`

```python
# ❌ Wrong (AttributeError)
torch.cuda.get_device_properties(0).total_mem

# ✅ Correct
torch.cuda.get_device_properties(0).total_memory
```

## Reference Files

For deep-dives, read the relevant reference file:

| Reference | When to read |
|---|---|
| [PARTIAL_LOADING.md](references/PARTIAL_LOADING.md) | Streaming, column/filter pushdown, split slicing, advanced Parquet tuning |
| [FILE_ACCESS.md](references/FILE_ACCESS.md) | `hf_hub_download`, `snapshot_download`, `HfFileSystem`, CLI, dry-run |
| [MODALITIES.md](references/MODALITIES.md) | ImageFolder, AudioFolder, VideoFolder, WebDataset, metadata patterns |
| [DATA_CARD.md](references/DATA_CARD.md) | YAML config reference, split detection rules, tag options, builder params |
