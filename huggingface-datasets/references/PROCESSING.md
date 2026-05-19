# Processing & Transforms

Reshape, filter, and cast datasets. All methods return a new `Dataset` unless noted.

## Column operations

```python
ds = ds.rename_column("old", "new")
ds = ds.remove_columns(["col1", "col2"])
ds = ds.select_columns(["text", "label"])
ds = ds.add_column("const", [0] * len(ds))
```

## Row operations

```python
ds = ds.sort("label")
ds = ds.shuffle(seed=42)
ds = ds.select([0, 10, 20])                              # by index
ds = ds.filter(lambda ex: ex["score"] > 0.9)              # by predicate
ds = ds.filter(lambda ex, i: i % 2 == 0, with_indices=True)
ds = ds.train_test_split(test_size=0.1, seed=42)           # -> DatasetDict
ds = ds.shard(num_shards=4, index=0)                       # deterministic
```

After `.shuffle()`, indexed lookups are 10× slower (indices mapping). For training, either `.flatten_indices()` to rewrite on disk, or convert to `IterableDataset` and use its fast buffered shuffle:

```python
ids = ds.to_iterable_dataset(num_shards=128).shuffle(seed=42, buffer_size=1000)
```

## `.map` — the core transform

```python
# Per-example
ds = ds.map(lambda ex: {"upper": ex["text"].upper()})

# Batched (much faster for vectorizable ops)
ds = ds.map(tokenize_fn, batched=True, batch_size=1000)

# Parallel processes
ds = ds.map(my_fn, num_proc=8)

# Remove columns after transform
ds = ds.map(encode, remove_columns=ds.column_names)

# With index / rank (multi-GPU)
import torch
ds = ds.map(gpu_fn, batched=True, with_rank=True,
            num_proc=torch.cuda.device_count())

# Change feature types
ds = ds.map(fn, features=new_features)
```

Rules for batched map:

- Input is a dict of lists, all same length
- Output dict's lists must all have the same length (but can differ from input)
- Can grow or shrink the dataset (drop rows by returning fewer elements; augment by returning more)

## `.filter` performance

`.filter` can be slow on large datasets. For Parquet-backed sources, prefer `filters=[("col", ">=", v)]` in `load_dataset` (pushdown — skips unread row groups). Use `.filter` only for logic that can't be expressed as pushdown predicates.

## Type casting

```python
from datasets import Audio, Image, ClassLabel, Value, Features

ds = ds.cast_column("audio", Audio(sampling_rate=16000))
ds = ds.cast_column("image", Image())
ds = ds.cast_column("label", ClassLabel(names=["neg", "pos"]))

# Cast multiple at once
ds = ds.cast(Features({
    "text": Value("string"),
    "label": ClassLabel(names=["neg", "pos"]),
}))
```

## Features — the schema

```python
from datasets import Features, Value, ClassLabel, Sequence, Image, Audio

Features({
    "id": Value("int64"),
    "text": Value("string"),
    "label": ClassLabel(names=["neg", "pos"]),
    "tokens": Sequence(Value("string")),
    "image": Image(),
    "audio": Audio(sampling_rate=16000),
    "boxes": Sequence(Sequence(Value("float32"), length=4)),
    "meta": {"source": Value("string"), "year": Value("int32")},
})
```

Inspect:

```python
ds.features
ds.column_names
ds.num_rows
ds.info
```

## Output formats — zero-copy tensor access

```python
# Persistent
ds = ds.with_format("torch")        # returns new Dataset object
ds.set_format("torch")              # mutates in place

# Per-column
ds.set_format("torch", columns=["input_ids", "attention_mask"])

# Available: "torch", "numpy", "pandas", "polars", "pyarrow", "tensorflow", "jax", None
```

### Dataframe formats

When formatted as `pandas` or `polars`, rows come back as DataFrames, enabling fast vectorized ops in `map`:

```python
ds = Dataset.from_dict({"text": ["foo", "bar"], "label": [0, 1]})
ds = ds.with_format("pandas")
ds = ds.map(lambda df: df.assign(upper=df.text.str.upper()), batched=True)
```

## On-the-fly transforms

`with_transform` applies a function only when examples are accessed (no caching, no materialization):

```python
def collate(batch):
    return tokenizer(batch["text"], padding=True, return_tensors="pt")

ds = ds.with_transform(collate)
ds[0]      # transform runs now
```

## Save and export

```python
# Arrow on disk (fastest to reload, larger, less metadata than Parquet)
ds.save_to_disk("path/dir")
from datasets import load_from_disk
ds = load_from_disk("path/dir")

# Parquet (compressed, portable, preferred for sharing/long-term)
ds.to_parquet("out.parquet")

# Other formats
ds.to_csv("out.csv")
ds.to_json("out.jsonl")

# Parallel upload
ds.push_to_hub("user/repo", num_proc=8)
```

See [UPLOADING.md](UPLOADING.md).

## Combining datasets

```python
from datasets import concatenate_datasets, interleave_datasets

combined = concatenate_datasets([a, b])                      # vertical
wide = concatenate_datasets([a, b], axis=1)                  # horizontal, same nrows
mixed = interleave_datasets([en, fr], probabilities=[0.8, 0.2], seed=42)
```

## Working with `DatasetDict`

All the above methods also work on `DatasetDict` (applied to each split):

```python
ds = load_dataset("glue", "mrpc")        # DatasetDict with train/validation/test
ds = ds.map(tokenize, batched=True)      # applied to all splits
ds["train"]                              # Dataset
```

## Common gotcha

```python
# WRONG
torch.cuda.get_device_properties(0).total_mem

# RIGHT
torch.cuda.get_device_properties(0).total_memory
```
