# Bucket Operations

## List Buckets

```python
buckets = b2_api.list_buckets()
for b in buckets:
    print(b.id_, b.type_, b.name)
```

## Create Bucket

```python
bucket = b2_api.create_bucket("my-new-bucket", "allPrivate")

# With lifecycle rules
bucket = b2_api.create_bucket(
    "my-bucket",
    "allPrivate",
    lifecycle_rules=[{
        "fileNamePrefix": "logs/",
        "daysFromUploadingToHiding": 30,
        "daysFromHidingToDeleting": 1,
    }],
)
```

Bucket names must be globally unique across all B2 accounts. Names can contain letters, digits, and `-`. Cannot start with `b2-`.

Bucket types: `"allPrivate"` (auth required for downloads) or `"allPublic"` (anyone can download). Always default to `allPrivate`.

## Update Bucket

```python
bucket = b2_api.get_bucket_by_name("my-bucket")

# Change type
updated = bucket.update(bucket_type="allPrivate")

# Set default SSE-B2 encryption
from b2sdk.v3 import EncryptionSetting, EncryptionMode
updated = bucket.update(
    default_server_side_encryption=EncryptionSetting(mode=EncryptionMode.SSE_B2),
)

# Set CORS rules
updated = bucket.update(
    cors_rules=[{
        "corsRuleName": "allowAll",
        "allowedOrigins": ["*"],
        "allowedHeaders": ["*"],
        "allowedOperations": [
            "b2_download_file_by_id",
            "b2_download_file_by_name",
        ],
        "maxAgeSeconds": 3600,
    }],
)
```

## Delete Bucket

```python
bucket = b2_api.get_bucket_by_name("bucket-to-delete")
b2_api.delete_bucket(bucket)
```

The bucket must be empty before deletion.

## Bucket Object Properties

```python
bucket = b2_api.get_bucket_by_name("my-bucket")
bucket.id_          # bucket ID
bucket.name         # bucket name
bucket.type_        # "allPrivate" or "allPublic"
bucket.as_dict()    # full dict with corsRules, lifecycleRules, etc.
```

## Download URL

For public buckets, construct the friendly download URL:

```python
url = bucket.get_download_url("path/to/file.txt")
# https://f001.backblazeb2.com/file/my-bucket/path/to/file.txt
```

## Download Authorization

Generate temporary download auth for private buckets:

```python
token = bucket.get_download_authorization(
    file_name_prefix="protected/",
    valid_duration_in_seconds=3600,
)
# Use token as ?Authorization=<token> query param on download URLs
```
