# Error Handling

## HTTP Status Codes

| Status | Meaning | Action |
|---|---|---|
| 200 | Success | — |
| 400 | Bad request | Fix request parameters |
| 401 | Unauthorized | Re-authorize (`bad_auth_token`, `expired_auth_token`) or fix key permissions (`unauthorized`) |
| 403 | Forbidden | Cap exceeded or account issue — check B2 console |
| 408 | Request timeout | Retry with new upload URL |
| 429 | Too many requests | Retry after `Retry-After` header value, or exponential backoff |
| 500 | Internal error | Retry with exponential backoff |
| 503 | Service unavailable | Retry with exponential backoff, respect `Retry-After` |

## SDK Exception Handling

The SDK raises typed exceptions. Common ones:

```python
from b2sdk.v3 import (
    B2Error,
    BucketNotAllowed,
    FileNotPresent,
    UnauthorizedError,
)

try:
    bucket.download_file_by_name("missing.txt")
except FileNotPresent:
    print("File does not exist")
except UnauthorizedError:
    print("Key lacks readFiles capability")
except B2Error as e:
    print(f"B2 error: {e}")
```

## Upload Retry Logic

The SDK handles upload retries internally, but when building your own retry logic around uploads:

1. If upload fails with connection error, timeout, 408, or 5xx → call `b2_get_upload_url` again (the SDK does this automatically)
2. The same upload URL/token pair can be reused for multiple sequential uploads until one fails
3. For parallel uploads, each thread needs its own upload URL

### Test Mode Headers

Use test headers to verify your retry logic (raw API only, not typically used with b2sdk):

- `X-Bz-Test-Mode: fail_some_uploads` — causes intermittent upload failures
- `X-Bz-Test-Mode: expire_some_account_authorization_tokens` — causes token expiration
- `X-Bz-Test-Mode: force_cap_exceeded` — simulates cap exceeded

## Rate Limiting

B2 may throttle on a per-account basis. When receiving 429:

1. Read `Retry-After` response header (seconds to wait)
2. If absent, use exponential backoff starting at 1 second
3. Cap backoff at ~60 seconds

## Integration Best Practices

- **User-Agent**: always set to identify your integration (`my-app/1.0.0+python/3.12`)
- **Metadata**: set `X-Bz-Info-src_last_modified_millis` (the SDK does this automatically for `upload_local_file`)
- **Bucket privacy**: always create buckets as `allPrivate` unless you specifically need public access
- **Background sync**: randomize start times to spread load (don't all sync at minute 0)
- **File deletion**: B2 uses versioning — hiding a file doesn't delete data. Delete all versions to fully remove
- **Bucket names**: globally unique across all B2 accounts; letters, digits, and `-` only; no `b2-` prefix
