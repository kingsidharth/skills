---
name: b2-native-python
description: Backblaze B2 cloud storage via b2sdk Python SDK. Auth, upload, download, buckets, keys, large files, sync.
---

# Backblaze B2 Native Python SDK (`b2sdk`)

Official Python SDK for Backblaze B2 Cloud Storage using the B2 Native API.

## Quick start

```python
from b2sdk.v3 import InMemoryAccountInfo, B2Api

info = InMemoryAccountInfo()
b2_api = B2Api(info)
b2_api.authorize_account("production", APPLICATION_KEY_ID, APPLICATION_KEY)

bucket = b2_api.get_bucket_by_name("my-bucket")
bucket.upload_local_file(
    local_file="/path/to/file.pdf",
    file_name="remote/file.pdf",
)
```

## Install

```bash
pip install b2sdk
# or
uv add b2sdk
```

Always import from `b2sdk.v3` (latest stable interface).

## References

- [Setup & auth](references/setup-and-auth.md) — AccountInfo types, authorization, credentials
- [File operations](references/file-operations.md) — Upload, download, list, copy, delete, metadata
- [Bucket operations](references/bucket-operations.md) — Create, list, update, delete buckets
- [Large files & sync](references/large-files-and-sync.md) — Multipart uploads, Synchronizer, parallel transfers
- [Keys & security](references/keys-and-security.md) — Application keys, SSE-B2, SSE-C encryption
- [Error handling](references/error-handling.md) — Status codes, retry logic, integration best practices
