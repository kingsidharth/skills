# Backblaze B2 Cloud Sync

Complete guide to integrating Backblaze B2 cloud storage with RunPod workflows for artifact backup, dataset distribution, and disaster recovery.

## Overview

**Use Cases**:
- Automatic checkpoint backup during fine-tuning
- Persistent artifact storage (survives pod termination)
- Dataset distribution across multiple pods
- Cost-effective long-term storage ($6/TB/month)

**Key Assumption**: B2 bucket is already set up and credentials are available.

## Setup & Authentication

### B2 Credentials
You need three pieces of information:
1. **Application Key ID** - Like a username
2. **Application Key** - Like a password (secret)
3. **Bucket Name** - Where your files are stored

Get these from: https://secure.backblaze.com/app_keys.htm

### Configure B2 Access on Pod

#### Method 1: rclone (Recommended)
```bash
# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Configure B2 backend
rclone config create b2 b2 \
  account $B2_KEY_ID \
  key $B2_APP_KEY

# Test connection
rclone lsd b2:YOUR_BUCKET_NAME

# List files
rclone ls b2:YOUR_BUCKET_NAME/checkpoints/
```

#### Method 2: b2 CLI
```bash
# Install B2 CLI
pip install b2sdk

# Configure
b2 authorize-account $B2_KEY_ID $B2_APP_KEY

# Test
b2 ls YOUR_BUCKET_NAME

# Download file
b2 download-file YOUR_BUCKET_NAME remote_file.zip local_file.zip

# Upload file
b2 upload-file YOUR_BUCKET_NAME local_file.tar.gz remote_file.tar.gz
```

#### Method 3: s3cmd (S3-compatible API)
```bash
# Install s3cmd
pip install s3cmd

# Configure
cat > ~/.s3cfg << EOF
[default]
access_key = $B2_KEY_ID
secret_key = $B2_APP_KEY
host_base = s3.us-west-004.backblazeb2.com
host_bucket = %(bucket)s.s3.us-west-004.backblazeb2.com
use_https = True
EOF

# Test
s3cmd ls s3://YOUR_BUCKET_NAME/

# Sync directory
s3cmd sync /workspace/checkpoints/ s3://YOUR_BUCKET_NAME/checkpoints/
```

## Integration Patterns

### Pattern 1: Pre-Stage Datasets from B2
**Use Case**: Download datasets before training starts
```bash
#!/bin/bash
# /workspace/download_datasets.sh

set -e

echo "Downloading datasets from B2..."

# Configure rclone
rclone config create b2 b2 account $B2_KEY_ID key $B2_APP_KEY

# Download datasets
mkdir -p /workspace/datasets
rclone copy b2:$B2_BUCKET/datasets/training_data /workspace/datasets/ \
  --progress \
  --transfers 8 \
  --checkers 16

# Verify download
echo "Dataset size:"
du -sh /workspace/datasets/

echo "Dataset download complete!"
```

**Add to pod startup**:
```bash
runpodctl create pods \
  --name "training-pod" \
  --gpuType "NVIDIA A40" \
  --imageName "your-image" \
  --networkVolumeId "$VOLUME_ID" \
  --env "B2_KEY_ID=$B2_KEY_ID,B2_APP_KEY=$B2_APP_KEY,B2_BUCKET=$B2_BUCKET" \
  --args "bash -c '/workspace/download_datasets.sh && python /workspace/train.py'"
```

### Pattern 2: Continuous Checkpoint Backup
**Use Case**: Backup checkpoints every 30 minutes during training
```bash
#!/bin/bash
# /workspace/continuous_backup.sh

set -e

# Configure rclone
rclone config create b2 b2 account $B2_KEY_ID key $B2_APP_KEY

echo "Starting continuous B2 backup (every 30 min)..."

while true; do
  TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")
  echo "[$TIMESTAMP] Syncing to B2..."
  
  # Sync checkpoints
  rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
    --transfers 4 \
    --checkers 8 \
    --progress
  
  # Sync logs
  rclone sync /workspace/logs b2:$B2_BUCKET/logs \
    --transfers 4 \
    --checkers 8 \
    --progress
  
  # Sync samples
  rclone sync /workspace/samples b2:$B2_BUCKET/samples \
    --transfers 4 \
    --checkers 8 \
    --progress
  
  echo "[$TIMESTAMP] Sync complete!"
  
  # Wait 30 minutes
  sleep 1800
done
```

**Launch as background process**:
```bash
ssh pod
nohup /workspace/continuous_backup.sh > /workspace/logs/b2_backup.log 2>&1 &

# Monitor backup logs
tail -f /workspace/logs/b2_backup.log
```

### Pattern 3: Final Sync Before Termination
**Use Case**: Ensure all artifacts are backed up before pod stops
```bash
#!/bin/bash
# /workspace/final_sync.sh

set -e

echo "=== Final B2 Sync Started at $(date) ==="

# Configure rclone
rclone config create b2 b2 account $B2_KEY_ID key $B2_APP_KEY

# Sync all artifacts with detailed logging
echo "Syncing checkpoints..."
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --progress \
  --log-file /workspace/logs/final_sync.log \
  --log-level INFO

echo "Syncing logs..."
rclone sync /workspace/logs b2:$B2_BUCKET/logs \
  --progress \
  --log-file /workspace/logs/final_sync.log \
  --log-level INFO

echo "Syncing samples..."
rclone sync /workspace/samples b2:$B2_BUCKET/samples \
  --progress \
  --log-file /workspace/logs/final_sync.log \
  --log-level INFO

echo "Syncing final models..."
rclone sync /workspace/output b2:$B2_BUCKET/final_models \
  --progress \
  --log-file /workspace/logs/final_sync.log \
  --log-level INFO

# Verify upload
echo "Verifying B2 contents..."
rclone size b2:$B2_BUCKET/checkpoints
rclone size b2:$B2_BUCKET/logs
rclone size b2:$B2_BUCKET/samples

echo "=== Final B2 Sync Completed at $(date) ==="
```

**Integrate with training script**:
```bash
#!/bin/bash
# /workspace/start_training.sh

set -e

# Training
python /workspace/ai-toolkit/run.py config/train_lora.yaml

# Final sync
/workspace/final_sync.sh

# Auto-terminate pod
echo "Stopping pod..."
runpodctl stop pod $RUNPOD_POD_ID
```

### Pattern 4: Selective Sync (Incremental)
**Use Case**: Only sync new/modified files to save bandwidth
```bash
#!/bin/bash
# /workspace/incremental_sync.sh

set -e

# Configure rclone
rclone config create b2 b2 account $B2_KEY_ID key $B2_APP_KEY

# Incremental sync - only new/modified files
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --update \  # Skip files that are newer on destination
  --transfers 8 \
  --checkers 16 \
  --progress \
  --stats-one-line

# Alternative: Only sync files from last hour
find /workspace/checkpoints -type f -mmin -60 -print0 | \
  while IFS= read -r -d '' file; do
    relative_path="${file#/workspace/checkpoints/}"
    rclone copy "$file" "b2:$B2_BUCKET/checkpoints/$(dirname "$relative_path")" --progress
  done
```

### Pattern 5: Download LoRAs for Inference
**Use Case**: Pull custom LoRAs from B2 before batch generation
```bash
#!/bin/bash
# /workspace/download_loras.sh

set -e

echo "Downloading LoRAs from B2..."

# Configure rclone
rclone config create b2 b2 account $B2_KEY_ID key $B2_APP_KEY

# Download all LoRAs
mkdir -p /workspace/loras
rclone copy b2:$B2_BUCKET/loras /workspace/loras \
  --include "*.safetensors" \
  --progress \
  --transfers 8

# Download specific LoRA
rclone copy b2:$B2_BUCKET/loras/character-lora-v1.safetensors /workspace/loras/ \
  --progress

# Verify
ls -lh /workspace/loras/
```

## Advanced Sync Strategies

### Bandwidth-Limited Sync
```bash
# Limit bandwidth to 50MB/s (useful for large transfers)
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --bwlimit 50M \
  --progress
```

### Parallel Transfers
```bash
# Increase parallelism for faster uploads
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --transfers 16 \  # More parallel transfers
  --checkers 32 \   # More parallel checksum verifiers
  --progress
```

### Compression During Upload
```bash
# Compress before uploading (saves bandwidth, takes more time)
tar czf - /workspace/checkpoints | \
  rclone rcat b2:$B2_BUCKET/checkpoints_$(date +%Y%m%d).tar.gz
```

### Versioned Backups
```bash
#!/bin/bash
# /workspace/versioned_backup.sh

TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Create timestamped backup
rclone sync /workspace/checkpoints \
  b2:$B2_BUCKET/backups/$TIMESTAMP/checkpoints \
  --progress

# Keep latest symlink
rclone copy /workspace/checkpoints \
  b2:$B2_BUCKET/latest/checkpoints \
  --progress
```

### Differential Backup
```bash
#!/bin/bash
# Only backup files modified in last 24 hours

find /workspace/checkpoints -type f -mtime -1 -print0 | \
  rsync -av --files-from=- --from0 / \
  /tmp/recent_checkpoints/

rclone sync /tmp/recent_checkpoints/workspace/checkpoints \
  b2:$B2_BUCKET/incremental_backups/$(date +%Y%m%d) \
  --progress
```

## Monitoring & Verification

### Check Sync Status
```bash
#!/bin/bash
# /workspace/check_sync_status.sh

echo "=== B2 Sync Status ==="

# Configure rclone
rclone config create b2 b2 account $B2_KEY_ID key $B2_APP_KEY

# Local sizes
echo "Local sizes:"
du -sh /workspace/checkpoints
du -sh /workspace/logs
du -sh /workspace/samples

# Remote sizes
echo -e "\nB2 bucket sizes:"
rclone size b2:$B2_BUCKET/checkpoints
rclone size b2:$B2_BUCKET/logs
rclone size b2:$B2_BUCKET/samples

# File counts
echo -e "\nFile counts:"
echo "Local checkpoints: $(find /workspace/checkpoints -type f | wc -l)"
echo "B2 checkpoints: $(rclone ls b2:$B2_BUCKET/checkpoints | wc -l)"
```

### Verify Checksums
```bash
# Compare checksums between local and B2
rclone check /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --one-way \
  --progress

# Output shows missing or different files
```

### Sync Alerts (via webhook)
```bash
#!/bin/bash
# /workspace/monitored_sync.sh

WEBHOOK_URL="https://your-webhook.com/notify"

# Perform sync
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --progress \
  --stats-one-line > /tmp/sync_stats.txt 2>&1

# Send notification
SYNC_STATUS=$?
STATS=$(cat /tmp/sync_stats.txt)

curl -X POST $WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d "{
    \"status\": $SYNC_STATUS,
    \"stats\": \"$STATS\",
    \"timestamp\": \"$(date)\",
    \"pod_id\": \"$RUNPOD_POD_ID\"
  }"
```

## Python Integration

### Basic Sync Script
```python
#!/usr/bin/env python3
# /workspace/b2_sync.py

import subprocess
import os
from pathlib import Path

class B2Syncer:
    def __init__(self):
        self.bucket = os.getenv('B2_BUCKET')
        self.key_id = os.getenv('B2_KEY_ID')
        self.app_key = os.getenv('B2_APP_KEY')
        
        # Configure rclone
        self.setup_rclone()
    
    def setup_rclone(self):
        """Configure rclone for B2 access"""
        cmd = [
            'rclone', 'config', 'create', 'b2', 'b2',
            'account', self.key_id,
            'key', self.app_key
        ]
        subprocess.run(cmd, check=True)
    
    def sync_directory(self, local_dir, remote_dir, verbose=True):
        """Sync local directory to B2"""
        cmd = [
            'rclone', 'sync',
            str(local_dir),
            f'b2:{self.bucket}/{remote_dir}',
            '--progress' if verbose else '--quiet',
            '--transfers', '8',
            '--checkers', '16'
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        return result.returncode == 0
    
    def download_directory(self, remote_dir, local_dir, verbose=True):
        """Download directory from B2"""
        cmd = [
            'rclone', 'copy',
            f'b2:{self.bucket}/{remote_dir}',
            str(local_dir),
            '--progress' if verbose else '--quiet',
            '--transfers', '8'
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        return result.returncode == 0
    
    def get_size(self, remote_dir):
        """Get size of remote directory"""
        cmd = ['rclone', 'size', f'b2:{self.bucket}/{remote_dir}', '--json']
        result = subprocess.run(cmd, capture_output=True, text=True)
        
        if result.returncode == 0:
            import json
            data = json.loads(result.stdout)
            return data['bytes']
        return 0

# Usage
syncer = B2Syncer()

# Sync checkpoints
print("Syncing checkpoints...")
success = syncer.sync_directory('/workspace/checkpoints', 'checkpoints')
print(f"Checkpoint sync: {'Success' if success else 'Failed'}")

# Sync logs
print("Syncing logs...")
success = syncer.sync_directory('/workspace/logs', 'logs')
print(f"Log sync: {'Success' if success else 'Failed'}")
```

### Training Callback Integration
```python
#!/usr/bin/env python3
# Integration with training loop

from b2_sync import B2Syncer

class B2CheckpointCallback:
    def __init__(self, sync_every_n_steps=500):
        self.syncer = B2Syncer()
        self.sync_every_n_steps = sync_every_n_steps
        self.step_count = 0
    
    def on_step_end(self, step, checkpoint_path):
        """Called after each training step"""
        self.step_count += 1
        
        if self.step_count % self.sync_every_n_steps == 0:
            print(f"[Step {step}] Syncing checkpoint to B2...")
            success = self.syncer.sync_directory(
                checkpoint_path,
                f'checkpoints/step_{step}'
            )
            print(f"B2 sync: {'✓' if success else '✗'}")

# Usage in training script
callback = B2CheckpointCallback(sync_every_n_steps=500)

for step in range(total_steps):
    # Training logic
    train_step()
    
    # Save checkpoint
    save_checkpoint(f'/workspace/checkpoints/step_{step}.safetensors')
    
    # Sync to B2
    callback.on_step_end(step, '/workspace/checkpoints')
```

## Cost Optimization

### B2 Pricing (as of 2024)
- **Storage**: $6/TB/month
- **Downloads**: $0.01/GB (free first 1GB/day per bucket)
- **Uploads**: Free
- **API calls**: Free (first 2,500/day per bucket)

### Cost-Saving Tips
```bash
# 1. Compress before uploading
tar czf checkpoints.tar.gz /workspace/checkpoints
rclone copy checkpoints.tar.gz b2:$B2_BUCKET/compressed/

# 2. Use lifecycle rules to auto-delete old backups
# (Configure in B2 console)

# 3. Only sync final checkpoints, not all intermediate
# Instead of syncing every checkpoint:
rclone copy /workspace/checkpoints/final_model.safetensors \
  b2:$B2_BUCKET/final_models/

# 4. Use B2's free tier effectively
# Free: 10GB storage, 1GB daily download, 2,500 API calls
```

## Troubleshooting

### Common Issues

**Problem: Sync hangs or times out**
```bash
# Solution: Reduce parallelism
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --transfers 4 \  # Instead of 8+
  --timeout 60m \  # Add timeout
  --retries 3
```

**Problem: Authentication fails**
```bash
# Verify credentials
echo $B2_KEY_ID
echo $B2_APP_KEY

# Re-configure rclone
rclone config delete b2
rclone config create b2 b2 account $B2_KEY_ID key $B2_APP_KEY

# Test connection
rclone lsd b2:
```

**Problem: Slow uploads**
```bash
# Check network speed
curl -o /dev/null http://speedtest.tele2.net/100MB.zip

# Optimize rclone settings
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints \
  --transfers 16 \  # More parallel
  --buffer-size 64M \  # Larger buffer
  --checkers 32 \  # More checksum workers
  --progress
```

**Problem: Out of disk space during sync**
```bash
# Stream directly to B2 without local storage
tar czf - /workspace/large_dataset | \
  rclone rcat b2:$B2_BUCKET/datasets/large_dataset.tar.gz
```

## Best Practices

1. **Always sync before pod termination** - Spot pods can be interrupted
2. **Use incremental syncs** - Faster and uses less bandwidth
3. **Verify uploads** - Use `rclone check` after critical syncs
4. **Monitor sync logs** - Catch failures early
5. **Test download speed** - Ensure datasets can be retrieved quickly
6. **Use environment variables** - Never hardcode credentials
7. **Compress large files** - Saves bandwidth and storage costs
8. **Keep backups versioned** - Use timestamps or version numbers

## Next Steps

- **Network volumes**: See [network-volumes.md](network-volumes.md)
- **Fine-tuning workflow**: See [../workflows/finetuning-workflow.md](../workflows/finetuning-workflow.md)
- **Artifact management**: See [artifact-management.md](artifact-management.md)
