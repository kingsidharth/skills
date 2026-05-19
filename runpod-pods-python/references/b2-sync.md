# Backblaze B2 Sync from RunPod Pods

## Console Cloud Sync (simplest)

Pod page > Cloud Sync > Backblaze B2. Enter:
- B2 Account ID (from B2 > App Keys)
- Application Key
- Bucket path (e.g. `my-bucket/training-runs/`)

Copies selected pod directories to/from B2.

## rclone (programmatic, recommended)

Install and configure inside pod:

```bash
# Install
curl https://rclone.org/install.sh | bash

# Configure (non-interactive)
mkdir -p ~/.config/rclone
cat > ~/.config/rclone/rclone.conf << EOF
[b2]
type = b2
account = ${B2_ACCOUNT_ID}
key = ${B2_APPLICATION_KEY}
EOF
```

### Common Operations

```bash
# Upload directory
rclone copy /workspace/checkpoints b2:my-bucket/checkpoints/ --progress

# Download to pod
rclone copy b2:my-bucket/datasets/ /workspace/datasets/ --progress

# Sync (mirror source to dest, deletes extras in dest)
rclone sync /workspace/outputs b2:my-bucket/outputs/ --progress

# List bucket contents
rclone ls b2:my-bucket/

# Single file
rclone copyto /workspace/model.safetensors b2:my-bucket/models/model.safetensors
```

### Useful Flags

| Flag | Purpose |
|---|---|
| `--progress` | Show transfer progress |
| `--transfers 16` | Parallel transfers (default 4) |
| `--fast-list` | Reduce API calls for large directories |
| `--include "*.safetensors"` | Filter by pattern |
| `--exclude "*.tmp"` | Exclude pattern |
| `--bwlimit 100M` | Bandwidth limit |
| `--dry-run` | Preview without transferring |

### Automated Periodic Sync

Useful for long training runs — sync checkpoints every N minutes:

```bash
# In background during training
while true; do
    rclone copy /workspace/checkpoints b2:my-bucket/checkpoints/ --quiet
    sleep 600  # every 10 min
done &
```

## b2 CLI (alternative)

```bash
pip install b2
b2 authorize-account $B2_ACCOUNT_ID $B2_APPLICATION_KEY

# Upload
b2 sync /workspace/outputs/ b2://my-bucket/outputs/

# Download
b2 sync b2://my-bucket/datasets/ /workspace/datasets/
```

## Credentials via RunPod Secrets

Store B2 credentials as RunPod secrets (console > Secrets):
- `b2_account_id`
- `b2_app_key`

Reference in pod/template env:

```python
"env": {
    "B2_ACCOUNT_ID": "{{ RUNPOD_SECRET_b2_account_id }}",
    "B2_APPLICATION_KEY": "{{ RUNPOD_SECRET_b2_app_key }}",
}
```

## S3-Compatible Access

B2 buckets are also accessible via S3-compatible API. Endpoint: `s3.us-west-004.backblazeb2.com` (varies by region).

rclone config for S3 mode:

```ini
[b2s3]
type = s3
provider = B2
access_key_id = YOUR_B2_KEY_ID
secret_access_key = YOUR_B2_APP_KEY
endpoint = s3.us-west-004.backblazeb2.com
```

This mode works with any S3 tool (boto3, aws cli, etc.).
