# CLI Patterns

Comprehensive guide to runpodctl automation patterns for pod lifecycle management, file transfers, and workflow automation.

## Installation & Configuration

### Install runpodctl
```bash
# Linux
wget --quiet --show-progress \
  https://github.com/runpod/runpodctl/releases/latest/download/runpodctl-linux-amd64 \
  -O runpodctl && \
  chmod +x runpodctl && \
  sudo mv runpodctl /usr/local/bin/runpodctl

# macOS (Apple Silicon)
wget --quiet --show-progress \
  https://github.com/runpod/runpodctl/releases/latest/download/runpodctl-darwin-arm64 \
  -O runpodctl && \
  chmod +x runpodctl && \
  sudo mv runpodctl /usr/local/bin/runpodctl

# macOS (Intel)
wget --quiet --show-progress \
  https://github.com/runpod/runpodctl/releases/latest/download/runpodctl-darwin-amd64 \
  -O runpodctl && \
  chmod +x runpodctl && \
  sudo mv runpodctl /usr/local/bin/runpodctl

# Verify installation
runpodctl version
```

### Configure API Key
```bash
# Set API key (get from https://runpod.io/console/user/settings)
runpodctl config --apiKey YOUR_RUNPOD_API_KEY

# Verify configuration
runpodctl get pod  # Should list your pods
```

## Pod Lifecycle Management

### Create Pods
```bash
# Basic GPU pod
runpodctl create pods \
  --name "my-training-pod" \
  --gpuType "NVIDIA A40" \
  --imageName "runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04" \
  --containerDiskSize 50 \
  --volumeSize 100

# With network volume
runpodctl create pods \
  --name "my-pod" \
  --gpuType "NVIDIA RTX 4090" \
  --imageName "your-image" \
  --networkVolumeId "abc123xyz" \
  --containerDiskSize 30

# Spot pod with bid
runpodctl create pods \
  --name "spot-inference" \
  --gpuType "NVIDIA RTX 4090" \
  --imageName "your-image" \
  --spot \
  --bid 0.40 \
  --networkVolumeId "abc123xyz"

# With environment variables
runpodctl create pods \
  --name "my-pod" \
  --gpuType "NVIDIA A40" \
  --imageName "your-image" \
  --env "HF_TOKEN=hf_xxx,WANDB_API_KEY=xxx,CUSTOM_VAR=value" \
  --ports "8888/http,22/tcp"

# With custom startup command
runpodctl create pods \
  --name "training-pod" \
  --gpuType "NVIDIA A40" \
  --imageName "your-image" \
  --args "bash /workspace/start_training.sh"
```

### List & Query Pods
```bash
# List all pods
runpodctl get pod

# Get specific pod details
runpodctl get pod POD_ID

# Filter by name (requires jq)
runpodctl get pod | jq '.[] | select(.name | contains("training"))'

# Get pod ID by name
POD_ID=$(runpodctl get pod | jq -r '.[] | select(.name=="my-pod") | .id')
echo $POD_ID
```

### Start/Stop Pods
```bash
# Start pod
runpodctl start pod POD_ID

# Stop pod
runpodctl stop pod POD_ID

# Schedule auto-stop after 12 hours
sleep 12h && runpodctl stop pod POD_ID &

# Conditional stop (if still running)
if runpodctl get pod POD_ID | grep -q "RUNNING"; then
  runpodctl stop pod POD_ID
fi
```

### Terminate Pods
```bash
# Remove single pod
runpodctl remove pod POD_ID

# Remove by name
runpodctl remove pods "my-training-pod"

# Bulk remove (up to 40 pods with same name)
runpodctl remove pods "batch-job" --podCount 40

# Remove all stopped pods (requires jq)
runpodctl get pod | jq -r '.[] | select(.desiredStatus=="EXITED") | .id' | \
  while read pod_id; do
    echo "Removing $pod_id"
    runpodctl remove pod $pod_id
  done
```

### View Logs
```bash
# View container logs
runpodctl logs pod POD_ID

# Follow logs (real-time)
runpodctl logs pod POD_ID --follow

# Save logs to file
runpodctl logs pod POD_ID > training_logs_$(date +%Y%m%d).txt
```

## File Transfer Patterns

### Send Files to Pod
```bash
# Send single file
runpodctl send myfile.txt

# Send multiple files
runpodctl send file1.txt file2.txt file3.txt

# Send entire directory
runpodctl send my_dataset/

# Send with wildcard
runpodctl send *.safetensors

# Compressed transfer (for large datasets)
tar czf - my_dataset/ | runpodctl send -
```

**Workflow**:
1. Local machine runs: `runpodctl send dataset.zip`
2. Output shows: `Code is: 8338-galileo-collect-fidel`
3. On pod: `runpodctl receive 8338-galileo-collect-fidel`

### Receive Files from Pod
```bash
# On pod: Start send
runpodctl send /workspace/checkpoints/*.safetensors

# On local: Receive with code
runpodctl receive 8338-galileo-collect-fidel

# Receive to specific directory
cd ~/Downloads && runpodctl receive 8338-galileo-collect-fidel
```

### Direct SSH File Transfer (Alternative)
```bash
# Get pod connection details
POD_IP=$(runpodctl get pod POD_ID | jq -r '.machine.podHostId')
SSH_PORT=$(runpodctl get pod POD_ID | jq -r '.machine.ports["22/tcp"][0].publicPort')

# Upload with scp
scp -P $SSH_PORT local_file.txt root@$POD_IP:/workspace/

# Download with scp
scp -P $SSH_PORT root@$POD_IP:/workspace/results.tar.gz ./

# Sync with rsync
rsync -avzP -e "ssh -p $SSH_PORT" \
  ./local_dir/ root@$POD_IP:/workspace/remote_dir/
```

## Automation Scripts

### Automated Training Pipeline
```bash
#!/bin/bash
# automated_training.sh

set -e  # Exit on error

# Configuration
NETWORK_VOLUME_ID="your-volume-id"
GPU_TYPE="NVIDIA A40"
IMAGE_NAME="your-training-image"
TRAINING_DURATION_HOURS=12
TIMEOUT_BUFFER_HOURS=2

# Create pod
echo "Creating training pod..."
POD_OUTPUT=$(runpodctl create pods \
  --name "auto-training-$(date +%Y%m%d-%H%M)" \
  --gpuType "$GPU_TYPE" \
  --imageName "$IMAGE_NAME" \
  --networkVolumeId "$NETWORK_VOLUME_ID" \
  --env "HF_TOKEN=$HF_TOKEN,WANDB_KEY=$WANDB_KEY" \
  --args "bash /workspace/start_training.sh")

POD_ID=$(echo "$POD_OUTPUT" | jq -r '.id')
echo "Pod created: $POD_ID"

# Set timeout
TIMEOUT_SECONDS=$(( ($TRAINING_DURATION_HOURS + $TIMEOUT_BUFFER_HOURS) * 3600 ))
(
  sleep $TIMEOUT_SECONDS
  echo "Timeout reached, stopping pod $POD_ID"
  runpodctl stop pod $POD_ID
) &
TIMEOUT_PID=$!

# Monitor pod status
echo "Monitoring pod..."
while true; do
  STATUS=$(runpodctl get pod $POD_ID | jq -r '.desiredStatus')
  
  if [ "$STATUS" == "EXITED" ]; then
    echo "Pod stopped successfully"
    kill $TIMEOUT_PID 2>/dev/null || true
    break
  fi
  
  sleep 300  # Check every 5 minutes
done

# Download artifacts
echo "Downloading training artifacts..."
ssh -p $(runpodctl get pod $POD_ID | jq -r '.machine.ports["22/tcp"][0].publicPort') \
  root@$(runpodctl get pod $POD_ID | jq -r '.machine.podHostId') \
  "cd /workspace && tar czf artifacts.tar.gz checkpoints/ logs/ samples/"

runpodctl send artifacts.tar.gz
# Receive code will be displayed

echo "Training complete. Pod ID: $POD_ID"
```

### Bulk Pod Management
```bash
#!/bin/bash
# bulk_pod_ops.sh

# Start multiple pods for parallel jobs
for i in {1..5}; do
  echo "Starting pod $i..."
  runpodctl create pods \
    --name "parallel-job-$i" \
    --gpuType "NVIDIA RTX 4090" \
    --imageName "your-image" \
    --networkVolumeId "$VOLUME_ID" \
    --env "JOB_ID=$i" \
    --args "python /workspace/process_batch.py --batch-id $i"
  
  sleep 2  # Avoid rate limiting
done

# Monitor all pods
while true; do
  RUNNING=$(runpodctl get pod | jq '[.[] | select(.name | contains("parallel-job")) | select(.desiredStatus=="RUNNING")] | length')
  echo "Running pods: $RUNNING"
  
  if [ $RUNNING -eq 0 ]; then
    echo "All pods completed"
    break
  fi
  
  sleep 60
done
```

### Pod Health Check
```bash
#!/bin/bash
# check_pod_health.sh

POD_ID=$1

if [ -z "$POD_ID" ]; then
  echo "Usage: $0 POD_ID"
  exit 1
fi

# Get pod status
STATUS=$(runpodctl get pod $POD_ID | jq -r '.desiredStatus')
echo "Status: $STATUS"

# Check disk usage (requires SSH access)
SSH_PORT=$(runpodctl get pod $POD_ID | jq -r '.machine.ports["22/tcp"][0].publicPort')
POD_IP=$(runpodctl get pod $POD_ID | jq -r '.machine.podHostId')

if [ "$STATUS" == "RUNNING" ]; then
  echo "Checking disk usage..."
  ssh -p $SSH_PORT root@$POD_IP "df -h | grep workspace"
  
  echo "Checking GPU utilization..."
  ssh -p $SSH_PORT root@$POD_IP "nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv"
fi
```

### Automatic Checkpoint Backup
```bash
#!/bin/bash
# backup_checkpoints.sh

POD_ID=$1
BACKUP_DIR="./backups/$(date +%Y%m%d)"

mkdir -p $BACKUP_DIR

# Get SSH details
SSH_PORT=$(runpodctl get pod $POD_ID | jq -r '.machine.ports["22/tcp"][0].publicPort')
POD_IP=$(runpodctl get pod $POD_ID | jq -r '.machine.podHostId')

# Sync checkpoints every hour
while true; do
  echo "[$(date)] Backing up checkpoints..."
  
  rsync -avzP -e "ssh -p $SSH_PORT" \
    root@$POD_IP:/workspace/checkpoints/ \
    $BACKUP_DIR/checkpoints/
  
  rsync -avzP -e "ssh -p $SSH_PORT" \
    root@$POD_IP:/workspace/logs/ \
    $BACKUP_DIR/logs/
  
  # Check if pod is still running
  STATUS=$(runpodctl get pod $POD_ID | jq -r '.desiredStatus')
  if [ "$STATUS" != "RUNNING" ]; then
    echo "Pod no longer running, final backup complete"
    break
  fi
  
  sleep 3600  # 1 hour
done
```

## Advanced Patterns

### Pod Creation with Retry Logic
```bash
#!/bin/bash
# create_pod_with_retry.sh

MAX_RETRIES=5
RETRY_DELAY=60

for i in $(seq 1 $MAX_RETRIES); do
  echo "Attempt $i/$MAX_RETRIES..."
  
  POD_OUTPUT=$(runpodctl create pods \
    --name "my-pod" \
    --gpuType "NVIDIA A40" \
    --imageName "your-image" \
    --networkVolumeId "$VOLUME_ID" 2>&1)
  
  if echo "$POD_OUTPUT" | grep -q "id"; then
    POD_ID=$(echo "$POD_OUTPUT" | jq -r '.id')
    echo "Success! Pod ID: $POD_ID"
    exit 0
  fi
  
  echo "Failed. Retrying in $RETRY_DELAY seconds..."
  sleep $RETRY_DELAY
done

echo "Failed to create pod after $MAX_RETRIES attempts"
exit 1
```

### Cost Tracking
```bash
#!/bin/bash
# track_pod_costs.sh

POD_ID=$1
GPU_COST_PER_HOUR=0.79  # Adjust for your GPU type

START_TIME=$(date +%s)

while true; do
  STATUS=$(runpodctl get pod $POD_ID | jq -r '.desiredStatus')
  
  if [ "$STATUS" != "RUNNING" ]; then
    break
  fi
  
  ELAPSED_HOURS=$(( ($(date +%s) - START_TIME) / 3600 ))
  COST=$(echo "$ELAPSED_HOURS * $GPU_COST_PER_HOUR" | bc -l)
  
  printf "\rRuntime: %d hours | Cost: \$%.2f" $ELAPSED_HOURS $COST
  
  sleep 60
done

echo -e "\nPod stopped. Final cost estimate: \$$(echo "$ELAPSED_HOURS * $GPU_COST_PER_HOUR" | bc -l)"
```

### Multi-Pod Orchestration
```python
#!/usr/bin/env python3
# orchestrate_pods.py

import subprocess
import json
import time
from typing import List, Dict

class PodOrchestrator:
    def __init__(self):
        self.pods: List[Dict] = []
    
    def create_pod(self, name: str, config: Dict) -> str:
        """Create a pod with given config"""
        cmd = [
            "runpodctl", "create", "pods",
            "--name", name,
            "--gpuType", config['gpu_type'],
            "--imageName", config['image'],
            "--networkVolumeId", config['volume_id'],
        ]
        
        if 'env' in config:
            env_str = ','.join([f"{k}={v}" for k, v in config['env'].items()])
            cmd.extend(["--env", env_str])
        
        if config.get('spot'):
            cmd.extend(["--spot", "--bid", str(config['bid'])])
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        pod_data = json.loads(result.stdout)
        pod_id = pod_data['id']
        
        self.pods.append({
            'id': pod_id,
            'name': name,
            'status': 'RUNNING'
        })
        
        return pod_id
    
    def wait_for_completion(self, timeout_hours: int = 24):
        """Wait for all pods to complete"""
        start_time = time.time()
        timeout_seconds = timeout_hours * 3600
        
        while True:
            if time.time() - start_time > timeout_seconds:
                print("Timeout reached, stopping all pods")
                self.stop_all_pods()
                break
            
            running_pods = []
            for pod in self.pods:
                status = self.get_pod_status(pod['id'])
                pod['status'] = status
                
                if status == 'RUNNING':
                    running_pods.append(pod['name'])
            
            if not running_pods:
                print("All pods completed")
                break
            
            print(f"Running: {', '.join(running_pods)}")
            time.sleep(60)
    
    def get_pod_status(self, pod_id: str) -> str:
        """Get current status of a pod"""
        cmd = ["runpodctl", "get", "pod", pod_id]
        result = subprocess.run(cmd, capture_output=True, text=True)
        pod_data = json.loads(result.stdout)
        return pod_data['desiredStatus']
    
    def stop_all_pods(self):
        """Stop all managed pods"""
        for pod in self.pods:
            if pod['status'] == 'RUNNING':
                subprocess.run(["runpodctl", "stop", "pod", pod['id']])

# Usage
orchestrator = PodOrchestrator()

# Launch multiple training jobs
for i in range(5):
    config = {
        'gpu_type': 'NVIDIA RTX 4090',
        'image': 'your-image',
        'volume_id': 'vol-id',
        'env': {'JOB_ID': str(i)},
        'spot': True,
        'bid': 0.40
    }
    
    pod_id = orchestrator.create_pod(f'job-{i}', config)
    print(f"Created pod {i}: {pod_id}")

# Wait for all to complete
orchestrator.wait_for_completion(timeout_hours=12)
```

## Troubleshooting

### Common Issues

**Problem: `runpodctl: command not found`**
```bash
# Verify installation
which runpodctl

# Re-install to PATH
sudo mv runpodctl /usr/local/bin/
```

**Problem: API key errors**
```bash
# Check config
cat ~/.runpod/config.toml

# Reconfigure
runpodctl config --apiKey NEW_API_KEY
```

**Problem: Pod creation fails with "No GPUs available"**
```bash
# Try different GPU type
--gpuType "NVIDIA RTX 4090"  # Instead of A40

# Try different datacenter (via web console)
# Or use spot pods
--spot --bid 0.50
```

**Problem: File transfer hangs**
```bash
# Use compression for large files
tar czf - large_dataset/ | runpodctl send -

# Or use direct SSH/rsync instead
rsync -avzP -e "ssh -p $SSH_PORT" file root@$POD_IP:/workspace/
```

## Performance Tips

### Optimize Transfers
```bash
# Compress before sending
tar czf dataset.tar.gz dataset/
runpodctl send dataset.tar.gz

# On pod: Extract
tar xzf dataset.tar.gz
```

### Parallel Operations
```bash
# Launch pods in parallel
for i in {1..5}; do
  (runpodctl create pods --name "job-$i" ...) &
done
wait  # Wait for all to complete
```

### Resource Monitoring
```bash
# Monitor all pods
watch -n 30 'runpodctl get pod | jq ".[] | {name, status: .desiredStatus, gpu: .machine.gpuTypeId}"'
```

## Next Steps

- **Pod lifecycle**: See [pod-lifecycle.md](pod-lifecycle.md)
- **Python SDK**: See [python-sdk.md](python-sdk.md)
- **File workflows**: See [../workflows/finetuning-workflow.md](../workflows/finetuning-workflow.md)
