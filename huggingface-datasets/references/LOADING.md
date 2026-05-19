# Loading Datasets

Covers every way to get a `Dataset` or `IterableDataset` object. For streaming specifics, see `STREAMING.md`. For individual file download, see `FILE_ACCESS.md`.

## From the Hub

```python
from datasets import load_dataset

# Default config, all splits (returns DatasetDict)
ds = load_dataset("user/repo")

# Specific split
ds = load_dataset("user/repo", split="train")

# Named config (subset) — second positional arg
ds = load_dataset("user/repo", "english", split="train")

# Specific revision (branch, tag, commit, or PR)
ds = load_dataset("user/repo", revision="v1.0")
ds = load_dataset("user/repo", revision="refs/pr/3")

# Gated / private — pass token or set HF_TOKEN env var
ds = load_dataset("user/private-repo", token="hf_...")
```

## Split slicing (load only part of a split)

```python
ds = load_dataset("repo", split="train[:100]")       # first 100
ds = load_dataset("repo", split="train[10:20]")      # range
ds = load_dataset("repo", split="train[:10%]")       # first 10%
ds = load_dataset("repo", split="train[-80%:]")      # last 80%
ds = load_dataset("repo", split="train[:10%]+train[-80%:]")   # combine
ds = load_dataset("repo", split="train+test")        # concat splits

# Cross-validation folds
val_splits = [f"train[{k}%:{k+10}%]" for k in range(0, 100, 10)]
train_splits = [f"train[:{k}%]+train[{k+10}%:]" for k in range(0, 100, 10)]
val = load_dataset("repo", split=val_splits)
train = load_dataset("repo", split=train_splits)
```

Programmatic equivalent:

```python
from datasets import ReadInstruction

ri = ReadInstruction("train", from_=10, to=20, unit="abs")
ri = ReadInstruction("train", to=10, unit="%")
ri = ReadInstruction("train", from_=50, to=52, unit="%", rounding="pct1_dropremainder")
ds = load_dataset("repo", split=ri)
```

## Local files

Builder name as first arg, paths via `data_files`:

```python
# CSV / TSV
ds = load_dataset("csv", data_files="data.csv")
ds = load_dataset("csv", data_files="data.tsv", sep="\t")

# JSON / JSONL
ds = load_dataset("json", data_files="data.jsonl")
ds = load_dataset("json", data_files="data.json", field="data")  # nested field

# Parquet — recommended
ds = load_dataset("parquet", data_files="data/*.parquet")

# Text — one row per line
ds = load_dataset("text", data_files=["a.txt", "b.txt"])

# Arrow, HDF5 also supported
```

### Mapping files to splits

```python
data_files = {
    "train": "data/train-*.parquet",
    "validation": "data/val-*.parquet",
    "test": "data/test-*.parquet",
}
ds = load_dataset("parquet", data_files=data_files)
```

### Glob patterns

```python
ds = load_dataset("parquet", data_files="data/train-*.parquet")

# Specific shard range
ds = load_dataset("json", data_files="shards/c4-train.0000*-of-01024.json.gz")
```

### `data_dir` for folder-based builders

```python
ds = load_dataset("imagefolder", data_dir="path/to/images")
ds = load_dataset("audiofolder", data_dir="path/to/audio")
```

See [FOLDER_BUILDERS.md](FOLDER_BUILDERS.md).

## From SQL

```python
from datasets import Dataset

ds = Dataset.from_sql("my_table", con="sqlite:///mydb.db")
ds = Dataset.from_sql(
    "SELECT text FROM table WHERE length(text) > 100 LIMIT 10",
    con="sqlite:///mydb.db",
)
```

## From Python objects

```python
from datasets import Dataset

ds = Dataset.from_dict({"text": ["a", "b"], "label": [0, 1]})
ds = Dataset.from_list([{"text": "a", "label": 0}, {"text": "b", "label": 1}])
ds = Dataset.from_pandas(df)
ds = Dataset.from_generator(my_gen_fn)         # generator function, not object
```

Note: `from_generator` requires a **picklable** generator function (not a generator object). See [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## Inspect without downloading

```python
from datasets import load_dataset_builder

builder = load_dataset_builder("cornell-movie-review-data/rotten_tomatoes")
builder.info.description
builder.info.features
builder.info.splits          # split name -> num_examples, num_bytes
builder.info.download_size
builder.info.dataset_size
```

Use this *first* for any unfamiliar dataset to decide strategy. See [DISCOVERY.md](DISCOVERY.md) for hub-level discovery.

## Auto-detection rules (no YAML in repo)

When a Hub repo has no `configs` block in `README.md`, splits are inferred:

1. Shard filenames: `data/train-00000-of-00003.parquet`
2. Filename keywords (delimited by non-word chars): `train.csv`, `my_test.json`, `validation1.parquet`
3. Directory names: `train/`, `test/`, `validation/`
4. Fallback: single `train` split

Keyword equivalents:

| Canonical | Also matches |
|---|---|
| `train` | `training` |
| `validation` | `valid`, `val`, `dev` |
| `test` | `testing`, `eval`, `evaluation` |

To override, write an explicit `configs` block. See [DATA_CARD.md](DATA_CARD.md).

## Multiprocessing download

```python
ds = load_dataset("timm/imagenet-1k-wds", num_proc=8)
```

One process per shard, bounded by shard count.

## Features override

```python
from datasets import Features, Value, ClassLabel

features = Features({
    "text": Value("string"),
    "label": ClassLabel(names=["negative", "positive"]),
})
ds = load_dataset("csv", data_files="data.csv", features=features)
```
