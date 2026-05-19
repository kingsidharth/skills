# Folder-Based Builders

ImageFolder, AudioFolder, VideoFolder — auto-create a dataset from a directory tree. For TAR-based multimodal at scale, use [WEBDATASET.md](WEBDATASET.md) instead.

## Auto-labels from directory names

```
data/train/cats/img001.png
data/train/dogs/img002.png
data/test/cats/img003.png
data/test/dogs/img004.png
```

```python
from datasets import load_dataset
ds = load_dataset("imagefolder", data_dir="data")
# Columns: image (PIL Image), label (ClassLabel: cats=0, dogs=1)
# Splits: train, test (from directory names)
```

Flat directories:

- `drop_labels=False` (default when subdirs present): labels inferred from subdirs
- `drop_labels=True`: no label column

Identical pattern for `audiofolder` (wav/mp3/flac/ogg/mp4 and anything ffmpeg decodes) and `videofolder` (mp4/avi/mov).

## Metadata file (for captions, bboxes, extra cols)

Place `metadata.csv`, `metadata.jsonl`, or `metadata.parquet` next to the files:

```
data/train/metadata.csv
data/train/0001.png
data/train/0002.png
```

The `file_name` column is mandatory.

### Captions

```csv
file_name,text
0001.png,A golden retriever playing with a ball
0002.png,A german shepherd sitting in grass
```

### Object detection

```jsonl
{"file_name": "0001.png", "objects": {"bbox": [[302.0, 109.0, 73.0, 52.0]], "categories": [0]}}
{"file_name": "0002.png", "objects": {"bbox": [[810.0, 100.0, 57.0, 28.0]], "categories": [1]}}
```

### Multiple files per row (input/output pairs)

```jsonl
{"input_file_name": "0001.png", "output_file_name": "0001_out.png"}
```

Use `*_file_name` for single refs, `*_file_names` (plural) for lists:

```jsonl
{"frames_file_names": ["0001_t0.png", "0001_t1.png"], "label": "moving_up"}
```

### Audio transcription

```csv
file_name,transcription
clip001.wav,Hello world
clip002.wav,How are you
```

## Zipped directories per split

Each zip can contain both media and `metadata.csv`:

```
data/train.zip
data/test.zip
data/validation.zip
```

Loader unpacks on-the-fly; no manual extraction.

## From a list of paths (no folder convention)

```python
from datasets import Dataset, Image, Audio

ds = Dataset.from_dict({
    "image": ["path/to/img1.png", "path/to/img2.png"],
    "caption": ["a dog", "a cat"],
})
ds = ds.cast_column("image", Image())

ds = Dataset.from_dict({"audio": ["clip1.wav", "clip2.wav"]})
ds = ds.cast_column("audio", Audio(sampling_rate=16000))    # on-the-fly resample
```

## Filtering with Parquet metadata

If metadata is stored as `metadata.parquet`, filter pushdown works:

```python
ds = load_dataset("user/imagenet-subset", streaming=True,
                  filters=[("label", "=", 0)])
```

## Pushing to the Hub

Two options:

```python
# A) Via Dataset object
ds.push_to_hub("user/my-dataset")

# B) Upload folder as-is, preserve structure
from huggingface_hub import HfApi
HfApi().upload_folder(
    folder_path="path/to/local/dataset",
    repo_id="user/my-dataset",
    repo_type="dataset",
)
```

See [UPLOADING.md](UPLOADING.md).

## Gotchas

- Metadata file must be named exactly `metadata.csv`/`.jsonl`/`.parquet` — other names won't be picked up
- `file_name` is case-sensitive and must use forward slashes relative to the metadata file
- If both subdirectories and a metadata file exist, metadata wins for columns it defines; subdir-based labels appear only if `drop_labels=False` is explicit
