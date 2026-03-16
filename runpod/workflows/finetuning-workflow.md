# Fine-tuning Workflow

Complete end-to-end workflow for LoRA fine-tuning using ai-toolkit templates with automatic artifact capture, B2 sync, and pod lifecycle management.

## Overview

**Typical Duration**: 10-12 hours  
**Storage Required**: 100-300GB network volume  
**Termination**: Automatic + 14h timeout failsafe + manual override  
**Artifacts**: Checkpoints, logs, validation samples, final LoRA weights

## Workflow Phases

### Phase 1: Pre-Flight Setup (One-Time)

#### 1.1 Create Network Volume
```bash
# Via RunPod Console
# 1. Navigate to Storage → New Network Volume
# 2. Select datacenter (affects GPU availability)
# 3. Size: 200GB minimum (adjust for dataset + checkpoints)
# 4. Name: "training-artifacts" or similar

# Via REST API
curl --request POST \
  --url https://rest.runpod.io/v1/networkvolumes \
  --header 'Authorization: Bearer RUNPOD_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "training-artifacts",
    "size": 200,
    "dataCenterId": "US-KS-2"
  }'

# Save the returned volume ID for pod deployment
```

#### 1.2 Prepare Datasets
**Option A: Pre-stage to Network Volume**
```bash
# Deploy temporary pod with volume attached
runpodctl create pods \
  --name "data-staging" \
  --gpuType "NVIDIA RTX 4090" \  # Cheapest for data prep
  --imageName "runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04" \
  --networkVolumeId "YOUR_VOLUME_ID" \
  --containerDiskSize 20

# SSH into pod and download datasets
ssh pod
cd /workspace/datasets
# Download your training data
wget https://your-dataset-url.com/dataset.zip
unzip dataset.zip

# Terminate staging pod
runpodctl remove pod POD_ID
```

**Option B: Download from B2 on Pod Start**
```bash
# Add to pod startup script
#!/bin/bash
# Install B2 CLI
pip install b2sdk

# Download datasets from B2
b2 download-file YOUR_BUCKET_NAME dataset.zip /workspace/datasets/dataset.zip
cd /workspace/datasets && unzip dataset.zip
```

#### 1.3 Create ai-toolkit Template (Lite Template Creation)
```dockerfile
# Dockerfile for ai-toolkit fine-tuning template
FROM runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04

# Install ai-toolkit dependencies
RUN pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
RUN pip install transformers diffusers accelerate peft bitsandbytes
RUN pip install huggingface_hub wandb omegaconf

# Clone ai-toolkit
WORKDIR /workspace
RUN git clone https://github.com/ostris/ai-toolkit.git
WORKDIR /workspace/ai-toolkit
RUN pip install -r requirements.txt

# Setup directories
RUN mkdir -p /workspace/checkpoints /workspace/logs /workspace/samples /workspace/datasets

# Copy startup script
COPY start_training.sh /workspace/start_training.sh
RUN chmod +x /workspace/start_training.sh

# Set working directory
WORKDIR /workspace/ai-toolkit

# Default command (can be overridden at pod creation)
CMD ["bash", "/workspace/start_training.sh"]
```

**Create startup script template**:
```bash
#!/bin/bash
# /workspace/start_training.sh

set -e  # Exit on error

echo "=== Fine-tuning Job Started at $(date) ===" | tee -a /workspace/logs/training.log

# Setup environment
export HF_HOME=/workspace/huggingface
export TRANSFORMERS_CACHE=/workspace/huggingface
export CHECKPOINT_DIR=/workspace/checkpoints
export LOG_DIR=/workspace/logs
export SAMPLE_DIR=/workspace/samples

# Verify dataset exists
if [ ! -d "/workspace/datasets/your-dataset" ]; then
  echo "ERROR: Dataset not found" | tee -a /workspace/logs/training.log
  exit 1
fi

# Start training with logging
cd /workspace/ai-toolkit
python run.py config/examples/train_lora_flux_24gb.yaml 2>&1 | tee -a /workspace/logs/training_$(date +%Y%m%d_%H%M%S).log

# Mark completion
echo "=== Fine-tuning Completed at $(date) ===" | tee -a /workspace/logs/training.log

# Optional: Trigger B2 sync before termination
if [ -n "$B2_BUCKET" ]; then
  echo "Syncing to B2..." | tee -a /workspace/logs/training.log
  rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints
  rclone sync /workspace/logs b2:$B2_BUCKET/logs
  rclone sync /workspace/samples b2:$B2_BUCKET/samples
fi

# Auto-terminate pod
echo "Stopping pod..." | tee -a /workspace/logs/training.log
runpodctl stop pod $RUNPOD_POD_ID
```

### Phase 2: Job Launch

#### 2.1 Deploy Fine-tuning Pod
```bash
# Set environment variables
export NETWORK_VOLUME_ID="your-volume-id"
export TEMPLATE_ID="your-ai-toolkit-template-id"  # From RunPod console

# Deploy pod via CLI
runpodctl create pods \
  --name "flux-lora-$(date +%Y%m%d-%H%M)" \
  --gpuType "NVIDIA A40" \
  --gpuCount 1 \
  --imageName "your-registry/ai-toolkit:latest" \
  --networkVolumeId "$NETWORK_VOLUME_ID" \
  --containerDiskSize 50 \
  --env "HF_TOKEN=$HF_TOKEN,B2_KEY_ID=$B2_KEY_ID,B2_APP_KEY=$B2_APP_KEY,B2_BUCKET=$B2_BUCKET" \
  --ports "22/tcp,8888/http" \
  --args "bash /workspace/start_training.sh"

# Save pod ID from output
export POD_ID="returned-pod-id"
```

**Via Python SDK**:
```python
import runpod
import os
from datetime import datetime

runpod.api_key = os.getenv("RUNPOD_API_KEY")

# Create pod
pod = runpod.create_pod(
    name=f"flux-lora-{datetime.now().strftime('%Y%m%d-%H%M')}",
    image_name="your-registry/ai-toolkit:latest",
    gpu_type_id="NVIDIA A40",
    cloud_type="SECURE",  # or "COMMUNITY"
    network_volume_id=os.getenv("NETWORK_VOLUME_ID"),
    container_disk_in_gb=50,
    volume_in_gb=0,  # Using network volume instead
    ports="22/tcp,8888/http",
    env={
        "HF_TOKEN": os.getenv("HF_TOKEN"),
        "B2_KEY_ID": os.getenv("B2_KEY_ID"),
        "B2_APP_KEY": os.getenv("B2_APP_KEY"),
        "B2_BUCKET": os.getenv("B2_BUCKET"),
    },
    docker_args="bash /workspace/start_training.sh"
)

pod_id = pod['id']
print(f"Pod created: {pod_id}")
```

#### 2.2 Set Timeout Failsafe
```bash
# Schedule auto-termination after 14 hours (buffer for 12h training)
# Run this on your local machine after pod starts
sleep 14h && runpodctl stop pod $POD_ID &

# OR schedule from within the pod (more reliable)
ssh into pod
nohup bash -c "sleep 14h; runpodctl stop pod $RUNPOD_POD_ID" > /workspace/logs/timeout.log 2>&1 &
```

**Python timeout monitoring**:
```python
import time
import runpod
import os

def monitor_with_timeout(pod_id, max_hours=14):
    """Monitor pod and terminate after max_hours"""
    start_time = time.time()
    timeout_seconds = max_hours * 3600
    
    while True:
        elapsed = time.time() - start_time
        
        # Check if timeout reached
        if elapsed >= timeout_seconds:
            print(f"Timeout reached ({max_hours}h), terminating pod")
            runpod.stop_pod(pod_id)
            break
        
        # Check pod status
        pod = runpod.get_pod(pod_id)
        if pod['desiredStatus'] == 'EXITED':
            print("Pod completed successfully")
            break
        
        # Wait 5 minutes before next check
        time.sleep(300)

# Usage
monitor_with_timeout(pod_id, max_hours=14)
```

### Phase 3: Monitoring & Artifact Capture

#### 3.1 Monitor Training Progress
```bash
# View container logs (stdout from training script)
runpodctl logs pod $POD_ID

# SSH into pod and tail training logs
ssh pod
tail -f /workspace/logs/training_*.log

# Check checkpoint directory
ls -lh /workspace/checkpoints/
```

#### 3.2 Checkpoint Strategy (ai-toolkit Config)
**Example ai-toolkit config (`train_lora_flux_24gb.yaml`)**:
```yaml
job: extension
config:
  name: flux_lora_training
  process:
    - type: sd_trainer
      training_folder: /workspace/ai-toolkit
      
      # Device and performance
      device: cuda:0
      network:
        type: lora
        linear: 16
        linear_alpha: 16
      
      # Save strategy
      save:
        dtype: float16
        save_every: 500  # Save every 500 steps
        max_step_saves_to_keep: 5
        
      # Checkpoints saved to network volume
      save_path: /workspace/checkpoints/flux_lora
      
      # Sample generation for monitoring
      sample:
        sampler: flowmatch
        sample_every: 500  # Generate samples every 500 steps
        width: 1024
        height: 1024
        prompts:
          - "your validation prompt 1"
          - "your validation prompt 2"
        save_path: /workspace/samples/
      
      # Dataset config
      datasets:
        - folder_path: /workspace/datasets/your-dataset
          caption_ext: txt
          cache_latents_to_disk: true
          cache_latents: true
```

#### 3.3 Background B2 Sync (Optional - during training)
```bash
# Run periodic sync every 30 minutes (from within pod)
ssh pod

# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Configure B2 backend
rclone config create b2 b2 \
  account $B2_KEY_ID \
  key $B2_APP_KEY

# Background sync script
cat > /workspace/sync_to_b2.sh << 'EOF'
#!/bin/bash
while true; do
  echo "Syncing to B2 at $(date)"
  rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints --progress
  rclone sync /workspace/logs b2:$B2_BUCKET/logs --progress
  rclone sync /workspace/samples b2:$B2_BUCKET/samples --progress
  sleep 1800  # 30 minutes
done
EOF

chmod +x /workspace/sync_to_b2.sh
nohup /workspace/sync_to_b2.sh > /workspace/logs/b2_sync.log 2>&1 &
```

### Phase 4: Post-Training Cleanup

#### 4.1 Final B2 Sync
```bash
# After training completes, do final sync
ssh pod
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints --progress
rclone sync /workspace/logs b2:$B2_BUCKET/logs --progress  
rclone sync /workspace/samples b2:$B2_BUCKET/samples --progress
rclone sync /workspace/ai-toolkit/output b2:$B2_BUCKET/final_models --progress
```

#### 4.2 Verify Artifacts
```bash
# Check what's on network volume
ssh pod
du -sh /workspace/*
ls -lh /workspace/checkpoints/
ls -lh /workspace/logs/
ls -lh /workspace/samples/

# Check B2 bucket
b2 ls YOUR_BUCKET_NAME
```

#### 4.3 Terminate Pod
```bash
# Stop pod (keeps network volume)
runpodctl stop pod $POD_ID

# OR terminate completely (network volume persists independently)
runpodctl remove pod $POD_ID
```

## AI-Toolkit Configuration Examples

### Flux LoRA (24GB VRAM)
```yaml
# /workspace/ai-toolkit/config/examples/train_lora_flux_24gb.yaml
job: extension
config:
  name: flux_dev_lora
  process:
    - type: sd_trainer
      training_folder: /workspace/ai-toolkit
      
      device: cuda:0
      
      # Model config
      model:
        name_or_path: black-forest-labs/FLUX.1-dev
        is_flux: true
        quantize: true  # Use quantization for 24GB
      
      # Network config (LoRA)
      network:
        type: lora
        linear: 16
        linear_alpha: 16
      
      # Save config
      save:
        dtype: float16
        save_every: 250
        max_step_saves_to_keep: 3
        push_to_hub: false
      
      save_path: /workspace/checkpoints/flux_lora
      
      # Training config
      train:
        batch_size: 1
        steps: 3000
        gradient_accumulation_steps: 1
        train_unet: true
        train_text_encoder: false
        learning_rate: 0.0001
        lr_scheduler: constant
        optimizer: adamw8bit
      
      # Sample generation
      sample:
        sampler: flowmatch
        sample_every: 250
        width: 1024
        height: 1024
        prompts:
          - "a photo of a TOK person"
          - "TOK person in professional attire"
        save_path: /workspace/samples/
      
      # Dataset
      datasets:
        - folder_path: /workspace/datasets/training_images
          caption_ext: txt
          cache_latents_to_disk: true
          cache_latents: true
```

### Qwen2-VL LoRA
```yaml
# /workspace/ai-toolkit/config/examples/train_lora_qwen2_vl.yaml
job: extension
config:
  name: qwen2_vl_lora
  process:
    - type: sd_trainer
      training_folder: /workspace/ai-toolkit
      
      # Model config
      model:
        name_or_path: Qwen/Qwen2-VL-7B-Instruct
        is_v_pred: true
      
      # LoRA config
      network:
        type: lora
        linear: 16
        linear_alpha: 16
      
      # Save strategy
      save:
        dtype: float16
        save_every: 100
        max_step_saves_to_keep: 5
      
      save_path: /workspace/checkpoints/qwen2_vl_lora
      
      # Training config
      train:
        batch_size: 2
        steps: 2000
        gradient_accumulation_steps: 4
        learning_rate: 0.0001
        optimizer: adamw8bit
      
      # Validation samples
      sample:
        sample_every: 100
        prompts:
          - "Describe this image in detail"
        save_path: /workspace/samples/
      
      # Dataset config
      datasets:
        - folder_path: /workspace/datasets/image_caption_pairs
          caption_ext: txt
```

## Troubleshooting

### Common Issues

**Problem: Pod runs out of VRAM**
```bash
# Solution 1: Enable quantization in config
model:
  quantize: true

# Solution 2: Reduce batch size
train:
  batch_size: 1
  gradient_accumulation_steps: 4  # Effective batch size = 4

# Solution 3: Use larger GPU
--gpuType "NVIDIA A100"
```

**Problem: Training crashes mid-job**
```bash
# Check logs
cat /workspace/logs/training_*.log | grep -i error

# Resume from last checkpoint
python run.py config/your_config.yaml \
  --resume_from /workspace/checkpoints/flux_lora/latest.safetensors
```

**Problem: Checkpoints not saving**
```bash
# Verify network volume is mounted
df -h | grep workspace

# Check permissions
ls -ld /workspace/checkpoints
chmod 755 /workspace/checkpoints

# Check disk space
du -sh /workspace/*
```

**Problem: B2 sync fails**
```bash
# Test connection
rclone lsd b2:YOUR_BUCKET_NAME

# Check credentials
echo $B2_KEY_ID
echo $B2_APP_KEY

# Manual sync with verbose output
rclone sync /workspace/checkpoints b2:$B2_BUCKET/checkpoints -vv
```

## Budget & Time Estimates

| GPU Type | Hourly Cost | 12h Training Cost | Recommended For |
|----------|-------------|-------------------|-----------------|
| RTX 4090 | $0.69/hr | $8.28 | Quick experiments |
| A40 | $0.79/hr | $9.48 | Standard LoRA training |
| A100 40GB | $1.89/hr | $22.68 | Large models, fast iteration |
| A100 80GB | $2.89/hr | $34.68 | Very large models |

**Additional costs**:
- Network volume: $0.07/GB/month (200GB = $14/month)
- Container disk: $0.10/GB/month (50GB = $5/month)

## Next Steps

- **Inference workflow**: See [inference-workflow.md](inference-workflow.md)
- **B2 integration details**: See [../storage/b2-cloud-sync.md](../storage/b2-cloud-sync.md)
- **Template customization**: See [../templates/finetuning-templates.md](../templates/finetuning-templates.md)
- **Cost optimization**: See [../reference/cost-optimization.md](../reference/cost-optimization.md)
