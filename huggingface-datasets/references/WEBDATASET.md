# WebDataset

TAR-archive format for large-scale multimodal streaming. Ideal when the dataset has millions of files and would be slow to list individually.

## Format

A WebDataset is a set of TAR shard files (typically ~1GB each):

```
shards/00000.tar
shards/00001.tar
shards/00002.tar
```

Inside each TAR, files with the **same prefix** are one example:

```
e39871fd.jpg
e39871fd.json
e39871fd.cls
f18b9158.jpg
f18b9158.json
f18b9158.cls
```

One column per file extension. Example: `ds[0]["jpg"]`, `ds[0]["json"]`, `ds[0]["cls"]`.

Supported media: jpeg, png, tiff, mp3, m4a, wav, flac, mp4, mov, avi, npy, npz. Labels/metadata in `.json`, `.txt`, `.cls`.

## Why WebDataset

- **Sequential I/O** — one large contiguous read per shard beats random-access across millions of tiny files
- **Cloud-friendly** — works great streaming from S3/HF/GCS
- **Ideal for DataLoader** — shard-per-worker parallelism

## Loading via `datasets` library

```python
from datasets import load_dataset

# Local
ds = load_dataset("webdataset", data_files="shards/*.tar",
                  split="train", streaming=True)

# Remote (URL list)
base = "https://huggingface.co/datasets/user/repo/resolve/main/"
urls = [base + f"shard-{i:05d}.tar" for i in range(100)]
ds = load_dataset("webdataset", data_files={"train": urls},
                  split="train", streaming=True)
```

## Loading via native `webdataset` library

Higher performance and more control. Install:

```bash
uv add webdataset
```

Stream from the Hub:

```python
import webdataset as wds
from huggingface_hub import get_token
from torch.utils.data import DataLoader

hf_token = get_token()
url = "https://huggingface.co/datasets/timm/imagenet-12k-wds/resolve/main/imagenet12k-train-{{0000..1023}}.tar"
url = f"pipe:curl -s -L {url} -H 'Authorization:Bearer {hf_token}'"

dataset = wds.WebDataset(url).decode()
loader = DataLoader(dataset, batch_size=64, num_workers=4)
```

The `{{0000..1023}}` brace expansion is a webdataset idiom — expanded client-side into 1024 shard URLs.

## Shuffle

WebDatasets are usually pre-shuffled at creation. For a second pass of randomization, combine shard shuffling with a buffer:

```python
dataset = (
    wds.WebDataset(url, shardshuffle=True)
    .shuffle(1000)       # buffer-based
    .decode()
)
```

## Multiple files of the same type per example

Dot-separated naming creates named columns:

```
e39871fd.input.jpg
e39871fd.output.jpg
e39871fd.json
```

Columns: `input.jpg`, `output.jpg`, `json`.

## Writing a WebDataset

```python
import webdataset as wds

with wds.ShardWriter("shards/shard-%05d.tar", maxcount=10_000) as sink:
    for i, (img_bytes, label) in enumerate(source):
        sink.write({
            "__key__": f"{i:09d}",
            "jpg": img_bytes,
            "cls": str(label),
        })
```

## Gotcha: no per-shard row count

WebDataset shards don't store row counts in their index. For multi-node training, either:

- Fix the row count per shard at creation and hardcode it
- Use the `wids` (indexed WebDataset) variant, which stores an index
- Accept that shards may have slightly different sizes and handle it in your sampler

## Uploading WebDatasets to the Hub

Use `HfApi.upload_folder` with the tar shards in a `data/` directory. Add to README YAML:

```yaml
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.tar"
```

See [UPLOADING.md](UPLOADING.md) and [DATA_CARD.md](DATA_CARD.md).
