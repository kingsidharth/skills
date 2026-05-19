# Discovering Datasets

Finding datasets on the Hub — by tag, size, license, author, modality — and pulling metadata (rows, bytes, splits, features) without downloading.

## `list_datasets` — the workhorse

```python
from huggingface_hub import list_datasets

# Iterate all (paginated)
for info in list_datasets(limit=20):
    print(info.id, info.downloads, info.likes)

# Search in name
list_datasets(search="wikipedia", limit=10)

# By author / organization
list_datasets(author="HuggingFaceFW")

# Top N by a sort key, descending
list_datasets(sort="downloads", direction=-1, limit=5)
list_datasets(sort="likes", direction=-1, limit=20)
list_datasets(sort="trendingScore", direction=-1, limit=20)
list_datasets(sort="lastModified", direction=-1, limit=20)
```

Returns an iterator of `DatasetInfo` objects with: `id`, `author`, `sha`, `last_modified`, `created_at`, `private`, `gated`, `downloads`, `likes`, `tags`, `card_data`.

Pass `full=True` to fetch extended attributes (files list, etc.). Pass `cardData=True` to include YAML front-matter.

## Filter by tags

`filter` accepts a single tag string or a list of tags (AND semantics):

```python
# Single tag
list_datasets(filter="task_categories:text-classification")

# Multiple tags (AND)
list_datasets(filter=[
    "task_categories:image-classification",
    "language:en",
    "size_categories:1M<n<10M",
])
```

### Common tag namespaces

| Prefix | Examples |
|---|---|
| `task_categories:` | `text-classification`, `image-classification`, `question-answering`, `text-generation`, `automatic-speech-recognition`, `object-detection` |
| `language:` | `en`, `es`, `zh`, `multilingual` |
| `license:` | `apache-2.0`, `mit`, `cc-by-4.0`, `cc-by-nc-4.0` |
| `size_categories:` | `n<1K`, `1K<n<10K`, `10K<n<100K`, `100K<n<1M`, `1M<n<10M`, `10M<n<100M`, `100M<n<1B`, `n>1T` |
| `modality:` | `image`, `audio`, `video`, `text`, `tabular` |
| `format:` | `parquet`, `json`, `csv`, `webdataset`, `imagefolder`, `soundfolder` |

To discover exact tag values, visit `https://huggingface.co/datasets`, set filters in the UI, and read them off the URL.

## Filter by row count

Pass an integer range to `num_rows` (same syntax as the Hub UI):

```python
# Datasets with > 10B rows
list_datasets(filter="num_rows:>10B")

# Bounded
list_datasets(filter="num_rows:min:1M,max:100M")
```

Row counts are available for all datasets in supported formats; for very large datasets, the count is estimated from the first 5GB.

## Combining filters — practical recipes

```python
# Top 10 Parquet English text-classification datasets, 100K–1M rows, permissive license
list_datasets(
    filter=[
        "task_categories:text-classification",
        "language:en",
        "size_categories:100K<n<1M",
        "format:parquet",
        "license:apache-2.0",
    ],
    sort="downloads",
    direction=-1,
    limit=10,
)

# Recently updated image datasets in WebDataset format
list_datasets(
    filter=["modality:image", "format:webdataset"],
    sort="lastModified",
    direction=-1,
    limit=20,
)
```

## Per-dataset metadata

### Lightweight: `dataset_info`

```python
from huggingface_hub import dataset_info

info = dataset_info("HuggingFaceFW/fineweb")
info.id
info.card_data          # parsed YAML front-matter
info.tags
info.downloads
info.siblings           # files in the repo with sizes
```

### Rich: `load_dataset_builder` (schema + split sizes)

```python
from datasets import load_dataset_builder

b = load_dataset_builder("cornell-movie-review-data/rotten_tomatoes")
b.info.features
b.info.splits            # {split_name: SplitInfo(num_examples, num_bytes, ...)}
b.info.download_size
b.info.dataset_size
b.info.description
b.info.license
b.info.citation
```

Neither call downloads the dataset itself — only README, YAML, and index metadata.

## Listing files without downloading

```python
from huggingface_hub import HfFileSystem

fs = HfFileSystem()
fs.ls("datasets/user/repo", detail=True)           # sizes, last_commit
fs.glob("datasets/user/repo/data/*.parquet")
fs.info("datasets/user/repo/data/train.parquet")   # size, blob_id, last_commit
```

See [FILE_ACCESS.md](FILE_ACCESS.md) for more.

## Dataset Viewer API — sample rows, stats, search

The Hub exposes a Dataset Viewer REST API for any public dataset in a supported format. Useful for previews without pulling the `datasets` library in.

```python
import requests

# First 100 rows
r = requests.get("https://datasets-server.huggingface.co/rows",
                 params={"dataset": "ibm/duorc", "config": "SelfRC",
                         "split": "train", "offset": 0, "length": 100})

# Filter (Parquet-backed datasets only, first 5GB)
r = requests.get("https://datasets-server.huggingface.co/filter",
                 params={"dataset": "ibm/duorc", "config": "SelfRC",
                         "split": "train", "where": '"no_answer"=true',
                         "offset": 0, "length": 10})

# Columns statistics (distributions, cardinality)
r = requests.get("https://datasets-server.huggingface.co/statistics",
                 params={"dataset": "ibm/duorc", "config": "SelfRC", "split": "train"})

# Full-text search (Parquet-backed)
r = requests.get("https://datasets-server.huggingface.co/search",
                 params={"dataset": "ibm/duorc", "config": "SelfRC",
                         "split": "train", "query": "alien"})
```

Private/gated datasets require `Authorization: Bearer <HF_TOKEN>`.

## CLI alternatives

```bash
hf datasets ls --search "instruction"
hf datasets ls --author Qwen --sort downloads --limit 10
hf datasets info user/repo
```

See [CLI.md](CLI.md).
