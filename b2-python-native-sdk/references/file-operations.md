# File Operations

## Upload

### Upload local file

```python
bucket = b2_api.get_bucket_by_name("my-bucket")

file_version = bucket.upload_local_file(
    local_file="/path/to/file.pdf",
    file_name="remote/path/file.pdf",
    file_infos={"author": "jane", "project": "demo"},  # optional custom metadata
)

print(file_version.id_)          # file version ID
print(file_version.file_name)    # remote file name
```

`upload_local_file` automatically uses the large file API for files exceeding the recommended part size (~100MB). No special handling needed.

### Upload bytes

```python
data = b"Hello, B2!"
file_version = bucket.upload_bytes(
    data_bytes=data,
    file_name="hello.txt",
    file_infos={"description": "greeting"},
)
```

### Upload with content type

```python
bucket.upload_local_file(
    local_file="image.png",
    file_name="images/photo.png",
    content_type="image/png",  # default: auto-detected from extension
)
```

## Download

### Download by name

```python
bucket = b2_api.get_bucket_by_name("my-bucket")
downloaded_file = bucket.download_file_by_name("remote/path/file.pdf")

# Inspect before saving
dv = downloaded_file.download_version
print(dv.file_name, dv.size, dv.content_type)

# Save to disk
downloaded_file.save_to("/local/path/file.pdf")
```

### Download by ID

```python
downloaded_file = b2_api.download_file_by_id(file_id)
downloaded_file.save_to("/local/path/file.pdf")
```

### Download to memory

```python
from io import BytesIO

downloaded_file = bucket.download_file_by_name("data.json")
buffer = BytesIO()
downloaded_file.save(buffer)
content = buffer.getvalue()
```

### Progress listeners

```python
from b2sdk.v3 import DoNothingProgressListener

downloaded_file = b2_api.download_file_by_id(
    file_id,
    progress_listener=DoNothingProgressListener(),
)
```

Available listeners: `DoNothingProgressListener`, `TqdmProgressListener`, `SimpleProgressListener`. You can subclass `AbstractProgressListener` for custom progress reporting.

## List Files

### List latest versions

```python
bucket = b2_api.get_bucket_by_name("my-bucket")

for file_version, folder_name in bucket.ls(latest_only=True):
    print(file_version.file_name, file_version.upload_timestamp)
```

### Recursive listing

```python
for file_version, folder_name in bucket.ls(latest_only=True, recursive=True):
    print(file_version.file_name)
```

### List within a folder (prefix)

```python
for file_version, folder_name in bucket.ls(
    folder_to_list="images/",
    latest_only=True,
):
    print(file_version.file_name)
```

### List all versions

```python
for file_version, folder_name in bucket.ls(latest_only=False):
    print(file_version.file_name, file_version.id_)
```

## Get File Metadata

```python
file_version = b2_api.get_file_info(file_id)
print(file_version.file_name)
print(file_version.content_type)
print(file_version.size)              # contentLength
print(file_version.content_sha1)
print(file_version.file_info)         # custom metadata dict
print(file_version.upload_timestamp)  # millis since epoch
```

## Copy File

```python
# Copy within same bucket
new_version = bucket.copy(file_id, "new_name.txt")

# Copy with byte range (offset + length)
new_version = bucket.copy(file_id, "partial.txt", offset=1024, length=2048)

# Copy to different bucket (same account)
dest_bucket = b2_api.get_bucket_by_name("other-bucket")
new_version = dest_bucket.copy(file_id, "copied.txt")
```

Files over 5GB require `length` to be specified so the SDK can use large file copy.

## Hide File

Hiding a file makes it invisible to `ls` / `list_file_names` but doesn't delete the data:

```python
bucket.hide_file("path/to/file.txt")
```

## Delete File Version

```python
# Delete a specific version
b2_api.delete_file_version(file_id, "file_name.txt")
```

To fully delete a file with versioning, delete all versions:

```python
for file_version, _ in bucket.ls(latest_only=False, prefix="target.txt"):
    b2_api.delete_file_version(file_version.id_, file_version.file_name)
```

## FileVersion and DownloadVersion Objects

API methods return rich objects with high-level methods:

```python
# FileVersion — returned by upload, get_file_info, ls
file_version = b2_api.get_file_info(file_id)
file_version.id_
file_version.file_name
file_version.size
file_version.content_type
file_version.upload_timestamp
file_version.as_dict()  # full dict representation

# DownloadVersion — available on downloaded files
downloaded = bucket.download_file_by_name("file.txt")
dv = downloaded.download_version
dv.file_name
dv.size
dv.content_type
dv.content_sha1
```
