# Keys & Security

## Application Keys

### Create a key

```python
key = b2_api.create_key(
    capabilities=[
        "listFiles", "readFiles", "writeFiles", "deleteFiles",
        "listBuckets", "readBuckets",
    ],
    key_name="my-app-key",
    valid_duration_in_seconds=86400,  # optional, 24h
    bucket_id="bucket-id",           # optional, restrict to one bucket
    name_prefix="uploads/",          # optional, restrict to prefix
)
print(key.id_)              # application key ID
print(key.application_key)  # the secret — only shown once
```

### List keys

```python
for key in b2_api.list_keys():
    print(key.id_, key.key_name, key.capabilities)
```

### Delete a key

```python
b2_api.delete_key(key_id)
```

### Available Capabilities

`listAllBucketNames`, `listBuckets`, `readBuckets`, `writeBuckets`, `deleteBuckets`, `listFiles`, `readFiles`, `writeFiles`, `deleteFiles`, `shareFiles`, `readBucketEncryption`, `writeBucketEncryption`, `readBucketRetentions`, `writeBucketRetentions`, `readFileRetentions`, `writeFileRetentions`, `readFileLegalHolds`, `writeFileLegalHolds`, `readBucketNotifications`, `writeBucketNotifications`, `bypassGovernance`, `listKeys`, `writeKeys`, `deleteKeys`

## Server-Side Encryption

### SSE-B2 (Backblaze-managed keys)

Backblaze manages the encryption keys. Enable per-upload or as bucket default:

```python
from b2sdk.v3 import EncryptionSetting, EncryptionMode

# Per upload
bucket.upload_local_file(
    local_file="secret.pdf",
    file_name="secret.pdf",
    encryption=EncryptionSetting(mode=EncryptionMode.SSE_B2),
)

# As bucket default
bucket.update(
    default_server_side_encryption=EncryptionSetting(mode=EncryptionMode.SSE_B2),
)
```

SSE-B2 files can be downloaded without any special parameters.

### SSE-C (Customer-managed keys)

You provide the encryption key. You must supply the same key for downloads:

```python
from b2sdk.v3 import EncryptionSetting, EncryptionMode, EncryptionKey

enc = EncryptionSetting(
    mode=EncryptionMode.SSE_C,
    key=EncryptionKey(
        secret=b"exactly-32-bytes-of-key-data!!!!",  # must be 32 bytes for AES-256
        id="my-key-id",  # your identifier, stored in file info
    ),
)

# Upload
bucket.upload_local_file(
    local_file="classified.pdf",
    file_name="classified.pdf",
    encryption=enc,
)

# Download — must provide the same key
downloaded = bucket.download_file_by_name("classified.pdf", encryption=enc)
downloaded.save_to("/local/classified.pdf")
```

### SSE-C Key Rules

- Key `secret` must be exactly 32 bytes (AES-256)
- The key is sent over HTTPS and never stored by Backblaze
- If you lose the key, the file is unrecoverable
- Key `id` is optional metadata stored in `fileInfo` as `sse_c_key_id`

## Object Lock

For immutable storage / compliance:

```python
from b2sdk.v3 import LegalHold, FileRetentionSetting, RetentionMode
import time

# Set legal hold
b2_api.update_file_legal_hold(file_id, file_name, LegalHold.ON)

# Set retention (governance mode)
retention = FileRetentionSetting(
    RetentionMode.GOVERNANCE,
    int((time.time() + 86400 * 365) * 1000),  # 1 year from now, in millis
)
b2_api.update_file_retention(file_id, file_name, retention)
```

Requires `writeFileLegalHolds` / `writeFileRetentions` capabilities and an Object Lock-enabled bucket.

## User-Agent

Set a custom User-Agent for your integration:

```python
b2_api = B2Api(info, user_agent_append="my-app/1.0.0")
```

Format: `product/version+dependencies`. Helps Backblaze identify your integration for support.
