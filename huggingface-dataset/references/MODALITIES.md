# Modality-Specific Patterns Reference

## ImageFolder

### Basic Structure (Auto-Labels from Directories)

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

If all images are in a flat directory (no label subdirectories), set `drop_labels=False` explicitly to still get labels, or `drop_labels=True` to suppress the label column entirely.

### With Metadata

Place `metadata.csv`, `metadata.jsonl`, or `metadata.parquet` alongside images:

```
data/train/metadata.csv
data/train/0001.png
data/train/0002.png
```

**Image captioning:**

```csv
file_name,text
0001.png,A golden retriever playing with a ball
0002.png,A german shepherd sitting in grass
```

```python
ds = load_dataset("imagefolder", data_dir="data", split="train")
ds[0]["text"]  # "A golden retriever playing with a ball"
```

**Object detection:**

```jsonl
{"file_name": "0001.png", "objects": {"bbox": [[302.0, 109.0, 73.0, 52.0]], "categories": [0]}}
{"file_name": "0002.png", "objects": {"bbox": [[810.0, 100.0, 57.0, 28.0]], "categories": [1]}}
```

**Multiple images per row** (e.g., input/output pairs):

```jsonl
{"input_file_name": "0001.png", "output_file_name": "0001_out.png"}
{"input_file_name": "0002.png", "output_file_name": "0002_out.png"}
```

**Lists of images** (use `*_file_names` — plural):

```jsonl
{"frames_file_names": ["0001_t0.png", "0001_t1.png"], "label": "moving_up"}
```

### Zipped Images

Each zip can contain both images and metadata:

```
data/train.zip
data/test.zip
data/validation.zip
```

### From Paths with Cast

```python
from datasets import Dataset, Image

ds = Dataset.from_dict({
    "image": ["path/to/img1.png", "path/to/img2.png"],
    "caption": ["a dog", "a cat"]
})
ds = ds.cast_column("image", Image())
```

### Filtering (Especially with Parquet Metadata)

```python
filters = [("label", "=", 0)]
ds = load_dataset("user/dataset", streaming=True, filters=filters)
```

## AudioFolder

Identical pattern to ImageFolder. Supports `.wav`, `.mp3`, `.flac`, `.ogg`, `.mp4`, and anything ffmpeg can decode.

```
data/train/speech/clip001.wav
data/train/music/clip002.mp3
```

```python
ds = load_dataset("audiofolder", data_dir="data")
# Columns: audio (Audio), label (ClassLabel)
```

### From Paths with Cast

```python
from datasets import Dataset, Audio

ds = Dataset.from_dict({"audio": ["clip1.wav", "clip2.wav"]})
ds = ds.cast_column("audio", Audio(sampling_rate=16000))
```

### Resampling on the Fly

```python
ds = ds.cast_column("audio", Audio(sampling_rate=16000))
# All audio decoded at 16kHz regardless of original sample rate
```

### Metadata for Transcription

```csv
file_name,transcription
clip001.wav,Hello world
clip002.wav,How are you
```

## VideoFolder

Same pattern. Supports `.mp4`, `.avi`, `.mov`, etc.

```
data/train/action/clip001.mp4
data/train/comedy/clip002.mp4
```

```python
ds = load_dataset("videofolder", data_dir="data")
```

### Captioning with Metadata

```csv
file_name,text
clip001.mp4,A person running in the park
clip002.mp4,Two people laughing together
```

## WebDataset (TAR-Based, for Scale)

Best for datasets with millions of files. Group files into TAR archives (e.g., ~1GB each) where each example shares a filename prefix:

```
shards/00000.tar
shards/00001.tar
shards/00002.tar
```

Inside each TAR:

```
e39871fd.jpg
e39871fd.json
f18b9158.jpg
f18b9158.json
```

One column is created per file extension: `ds[0]["jpg"]`, `ds[0]["json"]`.

### Loading

```python
# Local
ds = load_dataset("webdataset", data_files="shards/*.tar", split="train", streaming=True)

# Remote
base_url = "https://huggingface.co/datasets/user/repo/resolve/main/"
urls = [base_url + f"shard-{i:05d}.tar" for i in range(100)]
ds = load_dataset("webdataset", data_files={"train": urls}, split="train", streaming=True)
```

### Multiple Files per Example

Use a dot-separated naming convention for multiple files of the same type:

```
e39871fd.input.jpg
e39871fd.output.jpg
e39871fd.json
```

This creates columns: `input.jpg`, `output.jpg`, `json`.

## Text

Line-by-line text files:

```python
ds = load_dataset("text", data_files={"train": ["file1.txt", "file2.txt"]})
# Column: text (one row per line)
```

## Tabular (CSV / JSON / Parquet)

Parquet is the recommended format for ML — columnar, compressed, and supports predicate pushdown.

```python
# CSV
ds = load_dataset("csv", data_files="data.csv")
ds = load_dataset("csv", data_files="data.csv", sep="\t")  # TSV

# JSON (one object per line = JSONL)
ds = load_dataset("json", data_files="data.jsonl")

# Nested JSON with a specific field
ds = load_dataset("json", data_files="data.json", field="data")

# Parquet
ds = load_dataset("parquet", data_files="data.parquet")
```

### Specifying Features

Override auto-inferred types:

```python
from datasets import Features, Value, ClassLabel

features = Features({
    "text": Value("string"),
    "label": ClassLabel(names=["negative", "positive"])
})
ds = load_dataset("csv", data_files="data.csv", features=features)
```

## SQL Databases

```python
from datasets import Dataset

# Entire table
ds = Dataset.from_sql("my_table", con="sqlite:///mydb.db")

# Query
ds = Dataset.from_sql("SELECT text FROM table WHERE length(text) > 100 LIMIT 10",
                       con="sqlite:///mydb.db")
```

## Pushing to Hub

```python
# From Dataset object
ds.push_to_hub("user/my-dataset")
ds.push_to_hub("user/my-dataset", config_name="v2", split="train")

# From local folder (preserves structure)
from huggingface_hub import HfApi
api = HfApi()
api.upload_folder(
    folder_path="path/to/local/dataset",
    repo_id="user/my-dataset",
    repo_type="dataset",
)
```
