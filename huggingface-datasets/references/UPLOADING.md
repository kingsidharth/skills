# Uploading & Creating Datasets

How to publish to the Hub. Auth via `hf auth login` or `HF_TOKEN` env var — token must have **write** scope.

## `push_to_hub` — from a `Dataset` object

```python
from datasets import Dataset, DatasetDict

# Simple
ds.push_to_hub("user/my-dataset")

# Named config (subset), specific split
ds.push_to_hub("user/my-dataset", config_name="english", split="train")

# Private
ds.push_to_hub("user/my-dataset", private=True)

# Parallel shard upload
ds.push_to_hub("user/my-dataset", num_proc=8)

# DatasetDict — pushes all splits at once
dd = DatasetDict({"train": train_ds, "test": test_ds})
dd.push_to_hub("user/my-dataset")
```

`push_to_hub` serializes to Parquet, shards, and uploads. A `README.md` with a reasonable `configs` block is generated automatically if missing.

## `upload_folder` — raw file upload

Use when you already have files on disk in a layout you control (e.g., WebDataset shards, ImageFolder tree, pre-made Parquet):

```python
from huggingface_hub import HfApi
api = HfApi()

api.upload_folder(
    folder_path="path/to/local/dataset",
    repo_id="user/my-dataset",
    repo_type="dataset",
    commit_message="initial upload",
)

# Filter what gets uploaded
api.upload_folder(
    folder_path="./data",
    repo_id="user/my-dataset",
    repo_type="dataset",
    allow_patterns=["*.parquet", "README.md"],
    ignore_patterns=["*.tmp", ".DS_Store"],
)
```

## Creating a repo explicitly

```python
from huggingface_hub import create_repo

create_repo("user/my-dataset", repo_type="dataset", private=False)
```

## Creating datasets from code

```python
from datasets import Dataset

# From dict / list / pandas
Dataset.from_dict({"text": [...], "label": [...]})
Dataset.from_list([{"text": "a", "label": 0}, ...])
Dataset.from_pandas(df)

# From a generator function (must be picklable — not a generator object)
def gen():
    for line in open("huge.txt"):
        yield {"text": line}
ds = Dataset.from_generator(gen)

# With a schema
from datasets import Features, Value, ClassLabel
features = Features({"text": Value("string"), "label": ClassLabel(names=["a", "b"])})
ds = Dataset.from_dict(data, features=features)
```

## Folder-based source → Hub

Build locally from a folder, push to the Hub:

```python
from datasets import load_dataset

ds = load_dataset("imagefolder", data_dir="path/to/images")
ds.push_to_hub("user/my-image-dataset")
```

Or upload the folder as-is (preserves the ImageFolder convention):

```python
HfApi().upload_folder(folder_path="path/to/images", repo_id="user/ds", repo_type="dataset")
```

## Dataset card (README)

Write a `README.md` with YAML front-matter alongside your data. At minimum:

```yaml
---
license: apache-2.0
task_categories:
  - image-classification
language:
  - en
pretty_name: My Image Dataset
size_categories:
  - 100K<n<1M
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
  - split: test
    path: "data/test-*.parquet"
---

# My Image Dataset
...
```

Full tag reference: [DATA_CARD.md](DATA_CARD.md).

## Gated datasets

Require users to accept terms before download:

1. Push the dataset
2. Hub → dataset Settings → enable "Access requests"
3. Optionally add a gate form (terms of use)

After that, `load_dataset` requires an accepted-access token.

## Multi-commit safety

`push_to_hub` splits large uploads into multiple commits. If interrupted:

- Just re-run — already-uploaded shards are skipped via SHA check
- `HfHubHTTPError: 429 Too Many Requests` → upgrade `datasets ≥ 2.15.0` and retry
- Transient 500s from S3 backend → safe to retry

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## Deleting

```python
from huggingface_hub import HfApi, delete_repo

# Delete a single file
HfApi().delete_file(path_in_repo="data/old.parquet",
                    repo_id="user/my-dataset", repo_type="dataset")

# Delete the whole repo
delete_repo("user/my-dataset", repo_type="dataset")
```

Delete a config (and only its files) via CLI:

```bash
datasets-cli delete_from_hub USERNAME/DATASET_NAME CONFIG_NAME
```

See [CLI.md](CLI.md).
