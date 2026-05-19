# Large Files & Sync

## Large File Uploads

`upload_local_file` handles large files automatically — it switches to the multipart upload API when the file exceeds `recommendedPartSize` (typically 100MB). No code changes needed.

### Cancel unfinished large files

```python
bucket = b2_api.get_bucket_by_name("my-bucket")
for unfinished in bucket.list_unfinished_large_files():
    b2_api.cancel_large_file(unfinished.file_id, unfinished.file_name)
```

## Synchronizer

Sync is the preferred method for bulk transfers — it parallelizes scanning and data transfer for highest performance.

### Local → B2

```python
from b2sdk.v3 import (
    ScanPoliciesManager, parse_folder, Synchronizer, SyncReport
)
import sys, time

source = parse_folder("/local/path", b2_api)
destination = parse_folder("b2://my-bucket", b2_api)

policies = ScanPoliciesManager(exclude_all_symlinks=True)

synchronizer = Synchronizer(
    max_workers=10,
    policies_manager=policies,
    dry_run=False,
    allow_empty_source=True,
)

with SyncReport(sys.stdout, no_progress=False) as reporter:
    synchronizer.sync_folders(
        source_folder=source,
        dest_folder=destination,
        now_millis=int(round(time.time() * 1000)),
        reporter=reporter,
    )
```

### B2 → Local

```python
source = parse_folder("b2://my-bucket", b2_api)
destination = parse_folder("/local/download", b2_api)

with SyncReport(sys.stdout, no_progress=False) as reporter:
    synchronizer.sync_folders(
        source_folder=source,
        dest_folder=destination,
        now_millis=int(round(time.time() * 1000)),
        reporter=reporter,
    )
```

### B2 → B2

```python
source = parse_folder("b2://source-bucket", b2_api)
destination = parse_folder("b2://dest-bucket", b2_api)
# same pattern as above
```

### Scan Policies

Control which files are included/excluded:

```python
policies = ScanPoliciesManager(
    exclude_all_symlinks=True,
    exclude_file_regexes=[".*\\.tmp$", ".*\\.log$"],
    exclude_dir_regexes=["__pycache__", "\\.git"],
    include_file_regexes=None,  # None = include all (after excludes)
)
```

### Sync with Encryption

```python
from b2sdk.v3 import (
    BasicSyncEncryptionSettingsProvider,
    EncryptionSetting, EncryptionMode, EncryptionKey,
)

encryption_provider = BasicSyncEncryptionSettingsProvider({
    "my-bucket": EncryptionSetting(mode=EncryptionMode.SSE_B2),
    "secure-bucket": EncryptionSetting(
        mode=EncryptionMode.SSE_C,
        key=EncryptionKey(
            secret=b"your-32-byte-key-here-1234567890",
            id="my-key-id",
        ),
    ),
    "default-bucket": None,  # use bucket default
})

with SyncReport(sys.stdout, no_progress=False) as reporter:
    synchronizer.sync_folders(
        source_folder=source,
        dest_folder=destination,
        now_millis=int(round(time.time() * 1000)),
        reporter=reporter,
        encryption_settings_provider=encryption_provider,
    )
```

## Multithreaded Downloads

For files over 200MB, split into parts using the `Range` header. The SDK handles this for `download_file_by_id` and `download_file_by_name` when downloading to a file with `save_to()`.

## Part Size Constants

After authorization, the API info contains:

- `absoluteMinimumPartSize` — smallest allowed part (5MB)
- `recommendedPartSize` — default part size for large files (100MB)
