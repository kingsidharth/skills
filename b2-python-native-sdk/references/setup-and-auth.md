# Setup & Authentication

## AccountInfo

`AccountInfo` stores credentials, tokens, and upload URL cache. Choose one:

| Class | Persistence | Use case |
|---|---|---|
| `InMemoryAccountInfo` | None (RAM only) | Scripts, serverless, tests |
| `SqliteAccountInfo` | SQLite file on disk | CLI tools, long-running services |

```python
from b2sdk.v3 import InMemoryAccountInfo, B2Api, AuthInfoCache

# In-memory (most common)
info = InMemoryAccountInfo()
b2_api = B2Api(info)

# With auth cache for better performance
b2_api = B2Api(info, cache=AuthInfoCache(info))
```

## Authorization

```python
application_key_id = "your-key-id"
application_key = "your-application-key"

b2_api.authorize_account("production", application_key_id, application_key)
```

The realm is always `"production"` for real accounts.

Auth tokens expire after 24 hours. The SDK handles re-authorization automatically on most calls. If you get `expired_auth_token` or `bad_auth_token`, call `authorize_account` again.

## Credentials

Get credentials from the Backblaze web console under **Account > App Keys**.

- **Master Application Key**: full access to all buckets. Shown once at account creation.
- **Application Keys**: scoped to specific buckets/prefixes/capabilities. Created via web UI or `b2_api.create_key()`.

Store credentials in environment variables or a secrets manager — never hardcode:

```python
import os

b2_api.authorize_account(
    "production",
    os.environ["B2_APPLICATION_KEY_ID"],
    os.environ["B2_APPLICATION_KEY"],
)
```

## Getting a Bucket Reference

Most file operations happen through a `Bucket` object:

```python
# By name
bucket = b2_api.get_bucket_by_name("my-bucket")

# By ID
bucket = b2_api.get_bucket_by_id("bucket-id-here")
```
