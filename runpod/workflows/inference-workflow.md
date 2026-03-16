# Inference Workflow

Optimized bulk image generation workflow using spot pods with custom LoRAs, YAML-based generation recipes, and robust volume storage.

## Overview

**Typical Duration**: Variable (1-8 hours depending on batch size)  
**Pod Type**: Spot (cheaper, can be interrupted)  
**Storage**: Network volume CRITICAL (spot pods can terminate anytime)  
**Config Pattern**: Multiple YAML files per experiment  
**LoRA Sources**: B2 or network volume

## Workflow Phases

### Phase 1: Setup & Preparation

#### 1.1 Prepare Network Volume with LoRAs
```bash
# Option A: Deploy staging pod to upload LoRAs
runpodctl create pods \
  --name "lora-staging" \
  --gpuType "NVIDIA RTX 4090" \
  --imageName "runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04" \
  --networkVolumeId "YOUR_VOLUME_ID" \
  --containerDiskSize 20

# SSH and organize LoRAs
ssh pod
mkdir -p /workspace/loras /workspace/configs /workspace/outputs
cd /workspace/loras

# Upload LoRAs from B2
b2 download-file YOUR_BUCKET_NAME your-lora-v1.safetensors lora-v1.safetensors
b2 download-file YOUR_BUCKET_NAME your-lora-v2.safetensors lora-v2.safetensors

# OR use runpodctl send/receive from local machine
# Local: runpodctl send *.safetensors
# Pod: runpodctl receive CODE

# Verify
ls -lh /workspace/loras/
```

#### 1.2 Create Generation Config Files
**Generation config structure** (multiple YAML files per experiment):
```yaml
# /workspace/configs/experiment_001_character_portraits.yaml
generation_config:
  name: character_portraits_v1
  output_dir: /workspace/outputs/experiment_001
  
  # Model configuration
  model:
    base: stabilityai/stable-diffusion-xl-base-1.0
    loras:
      - path: /workspace/loras/character-lora-v1.safetensors
        weight: 0.8
      - path: /workspace/loras/style-lora-v2.safetensors
        weight: 0.6
  
  # Generation parameters
  generation:
    width: 1024
    height: 1024
    steps: 30
    cfg_scale: 7.5
    sampler: euler_a
    seed_start: 1000  # For reproducibility
    
  # Batch configuration
  batch:
    prompts_file: /workspace/configs/prompts_experiment_001.txt
    images_per_prompt: 4
    batch_size: 2  # Process 2 prompts at once
```

**Prompts file**:
```txt
# /workspace/configs/prompts_experiment_001.txt
a portrait of TOK character in medieval armor, dramatic lighting
TOK character as a cyberpunk hacker, neon city background
close-up of TOK character, professional headshot, studio lighting
TOK character in formal wear at a gala event
```

**Example config for multiple experiments**:
```yaml
# /workspace/configs/experiment_002_style_tests.yaml
generation_config:
  name: style_comparison_v1
  output_dir: /workspace/outputs/experiment_002
  
  model:
    base: stabilityai/stable-diffusion-xl-base-1.0
    loras:
      - path: /workspace/loras/anime-style-lora.safetensors
        weight: 1.0
  
  generation:
    width: 1024
    height: 1024
    steps: 25
    cfg_scale: 8.0
    sampler: dpm++_2m
  
  batch:
    prompts_file: /workspace/configs/prompts_style_tests.txt
    images_per_prompt: 8  # More variations
    batch_size: 1
```

#### 1.3 Create Inference Template (Lite)
```dockerfile
# Dockerfile for ComfyUI + Custom Scripts
FROM runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04

# Install ComfyUI
WORKDIR /workspace
RUN git clone https://github.com/comfyanonymous/ComfyUI.git
WORKDIR /workspace/ComfyUI
RUN pip install -r requirements.txt

# Install additional dependencies
RUN pip install pyyaml omegaconf safetensors diffusers transformers accelerate

# Setup directories
RUN mkdir -p /workspace/loras /workspace/configs /workspace/outputs

# Copy batch generation script
COPY run_batch_generation.py /workspace/run_batch_generation.py
COPY utils/ /workspace/utils/

RUN chmod +x /workspace/run_batch_generation.py

WORKDIR /workspace
CMD ["python", "/workspace/run_batch_generation.py"]
```

**Batch generation script template**:
```python
#!/usr/bin/env python3
# /workspace/run_batch_generation.py

import os
import yaml
import torch
from pathlib import Path
from diffusers import StableDiffusionXLPipeline
from safetensors.torch import load_file
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class BatchGenerator:
    def __init__(self, config_path):
        self.config = self.load_config(config_path)
        self.pipeline = None
        self.output_dir = Path(self.config['generation_config']['output_dir'])
        self.output_dir.mkdir(parents=True, exist_ok=True)
    
    def load_config(self, config_path):
        with open(config_path, 'r') as f:
            return yaml.safe_load(f)
    
    def setup_pipeline(self):
        """Load base model and apply LoRAs"""
        cfg = self.config['generation_config']
        
        # Load base model
        logger.info(f"Loading base model: {cfg['model']['base']}")
        self.pipeline = StableDiffusionXLPipeline.from_pretrained(
            cfg['model']['base'],
            torch_dtype=torch.float16,
            use_safetensors=True
        ).to("cuda")
        
        # Load and apply LoRAs
        for lora_cfg in cfg['model'].get('loras', []):
            logger.info(f"Loading LoRA: {lora_cfg['path']} (weight: {lora_cfg['weight']})")
            self.pipeline.load_lora_weights(
                lora_cfg['path'],
                weight_name=os.path.basename(lora_cfg['path'])
            )
            # Apply weight (simplified - actual implementation depends on framework)
            # self.pipeline.fuse_lora(lora_scale=lora_cfg['weight'])
    
    def load_prompts(self):
        """Load prompts from file"""
        prompts_file = self.config['generation_config']['batch']['prompts_file']
        with open(prompts_file, 'r') as f:
            prompts = [line.strip() for line in f if line.strip() and not line.startswith('#')]
        return prompts
    
    def generate_images(self):
        """Main generation loop"""
        cfg = self.config['generation_config']
        gen_cfg = cfg['generation']
        batch_cfg = cfg['batch']
        
        prompts = self.load_prompts()
        logger.info(f"Loaded {len(prompts)} prompts")
        
        seed = gen_cfg.get('seed_start', 42)
        
        for prompt_idx, prompt in enumerate(prompts):
            logger.info(f"Processing prompt {prompt_idx+1}/{len(prompts)}: {prompt[:50]}...")
            
            # Generate multiple images per prompt
            for img_idx in range(batch_cfg['images_per_prompt']):
                current_seed = seed + (prompt_idx * 100) + img_idx
                generator = torch.Generator(device="cuda").manual_seed(current_seed)
                
                # Generate image
                image = self.pipeline(
                    prompt=prompt,
                    width=gen_cfg['width'],
                    height=gen_cfg['height'],
                    num_inference_steps=gen_cfg['steps'],
                    guidance_scale=gen_cfg['cfg_scale'],
                    generator=generator
                ).images[0]
                
                # Save with metadata
                filename = f"prompt_{prompt_idx:04d}_img_{img_idx:02d}_seed_{current_seed}.png"
                output_path = self.output_dir / filename
                image.save(output_path)
                
                logger.info(f"Saved: {filename}")
        
        logger.info(f"Generation complete. {len(prompts) * batch_cfg['images_per_prompt']} images saved to {self.output_dir}")

def main():
    # Find all config files
    config_dir = Path(os.getenv('CONFIG_DIR', '/workspace/configs'))
    config_files = sorted(config_dir.glob('experiment_*.yaml'))
    
    if not config_files:
        logger.error(f"No experiment configs found in {config_dir}")
        return
    
    logger.info(f"Found {len(config_files)} experiment configs")
    
    # Process each config
    for config_file in config_files:
        logger.info(f"\n{'='*60}")
        logger.info(f"Processing: {config_file.name}")
        logger.info(f"{'='*60}\n")
        
        try:
            generator = BatchGenerator(str(config_file))
            generator.setup_pipeline()
            generator.generate_images()
        except Exception as e:
            logger.error(f"Error processing {config_file.name}: {e}")
            continue
        
        # Clear GPU memory between experiments
        torch.cuda.empty_cache()
    
    logger.info("\nAll experiments completed!")

if __name__ == "__main__":
    main()
```

### Phase 2: Launch Spot Pod

#### 2.1 Deploy Spot Pod for Generation
```bash
# Deploy spot pod with network volume
runpodctl create pods \
  --name "bulk-gen-$(date +%Y%m%d-%H%M)" \
  --gpuType "NVIDIA RTX 4090" \
  --gpuCount 1 \
  --imageName "your-registry/comfyui-batch:latest" \
  --networkVolumeId "$NETWORK_VOLUME_ID" \
  --spot \
  --bid 0.40 \  # Max price (spot usually ~30-50% cheaper)
  --containerDiskSize 30 \
  --env "CONFIG_DIR=/workspace/configs,OUTPUT_DIR=/workspace/outputs" \
  --ports "22/tcp,8188/http" \
  --args "python /workspace/run_batch_generation.py"

# Save pod ID
export POD_ID="returned-pod-id"
```

**Via Python SDK**:
```python
import runpod
import os

runpod.api_key = os.getenv("RUNPOD_API_KEY")

pod = runpod.create_pod(
    name=f"bulk-gen-{datetime.now().strftime('%Y%m%d-%H%M')}",
    image_name="your-registry/comfyui-batch:latest",
    gpu_type_id="NVIDIA RTX 4090",
    cloud_type="COMMUNITY",  # Spot pods typically in Community Cloud
    network_volume_id=os.getenv("NETWORK_VOLUME_ID"),
    container_disk_in_gb=30,
    support_public_ip=False,  # Save costs
    bid_per_gpu=0.40,  # Spot bid
    env={
        "CONFIG_DIR": "/workspace/configs",
        "OUTPUT_DIR": "/workspace/outputs",
    },
    docker_args="python /workspace/run_batch_generation.py"
)
```

#### 2.2 Monitor Progress (Basic)
```bash
# Check container logs
runpodctl logs pod $POD_ID

# SSH and monitor
ssh pod
tail -f /workspace/logs/generation.log

# Check output directory
watch -n 30 'ls -lh /workspace/outputs/experiment_*/  | tail -20'
```

**Optional: Progress webhook** (for longer jobs):
```python
# Add to batch generation script
import requests

WEBHOOK_URL = os.getenv('PROGRESS_WEBHOOK')

def send_progress(current, total, experiment_name):
    if WEBHOOK_URL:
        requests.post(WEBHOOK_URL, json={
            'experiment': experiment_name,
            'progress': current,
            'total': total,
            'percent': (current / total) * 100
        })
```

### Phase 3: Output Management

#### 3.1 Organize Outputs on Volume
```bash
# Directory structure on network volume
/workspace/
├── loras/
│   ├── character-lora-v1.safetensors
│   ├── style-lora-v2.safetensors
│   └── anime-style-lora.safetensors
├── configs/
│   ├── experiment_001_character_portraits.yaml
│   ├── experiment_002_style_tests.yaml
│   └── prompts_*.txt
├── outputs/
│   ├── experiment_001/
│   │   ├── prompt_0000_img_00_seed_1000.png
│   │   ├── prompt_0000_img_01_seed_1001.png
│   │   └── ...
│   └── experiment_002/
│       └── ...
└── logs/
    └── generation.log
```

#### 3.2 Sync to B2 (During or After Generation)
```bash
# Option A: Background sync during generation
ssh pod
cat > /workspace/sync_outputs.sh << 'EOF'
#!/bin/bash
while true; do
  rclone sync /workspace/outputs b2:$B2_BUCKET/outputs \
    --transfers 4 \
    --checkers 8 \
    --progress
  sleep 300  # Sync every 5 minutes
done
EOF

chmod +x /workspace/sync_outputs.sh
nohup /workspace/sync_outputs.sh > /workspace/logs/sync.log 2>&1 &

# Option B: Sync after completion
rclone sync /workspace/outputs b2:$B2_BUCKET/outputs --progress
```

#### 3.3 Download Outputs Locally
```bash
# Option A: Direct B2 download (recommended for large batches)
b2 sync b2://YOUR_BUCKET_NAME/outputs ./local_outputs

# Option B: From pod via runpodctl
runpodctl send /workspace/outputs/experiment_001/*.png
# Then receive on local machine
runpodctl receive CODE

# Option C: Via rsync over SSH
rsync -avzP -e "ssh -p POD_SSH_PORT" \
  root@POD_IP:/workspace/outputs/ \
  ./local_outputs/
```

### Phase 4: Spot Pod Handling

#### 4.1 Spot Interruption Recovery
**Add checkpoint mechanism to generation script**:
```python
import json
from pathlib import Path

class CheckpointManager:
    def __init__(self, checkpoint_file='/workspace/outputs/.checkpoint.json'):
        self.checkpoint_file = Path(checkpoint_file)
        self.checkpoint = self.load_checkpoint()
    
    def load_checkpoint(self):
        if self.checkpoint_file.exists():
            with open(self.checkpoint_file, 'r') as f:
                return json.load(f)
        return {}
    
    def save_checkpoint(self, experiment, prompt_idx, img_idx):
        self.checkpoint[experiment] = {
            'prompt_idx': prompt_idx,
            'img_idx': img_idx,
            'timestamp': str(datetime.now())
        }
        with open(self.checkpoint_file, 'w') as f:
            json.dump(self.checkpoint, f, indent=2)
    
    def get_resume_point(self, experiment):
        if experiment in self.checkpoint:
            return (
                self.checkpoint[experiment]['prompt_idx'],
                self.checkpoint[experiment]['img_idx']
            )
        return (0, 0)

# Use in generation loop
checkpoint_mgr = CheckpointManager()
start_prompt, start_img = checkpoint_mgr.get_resume_point(cfg['name'])

for prompt_idx in range(start_prompt, len(prompts)):
    for img_idx in range(start_img, images_per_prompt):
        # Generate image
        # ...
        
        # Save checkpoint after each image
        checkpoint_mgr.save_checkpoint(cfg['name'], prompt_idx, img_idx)
```

**Redeploy after interruption**:
```bash
# Spot pod was terminated - redeploy with same volume
runpodctl create pods \
  --name "bulk-gen-resume" \
  --gpuType "NVIDIA RTX 4090" \
  --networkVolumeId "$NETWORK_VOLUME_ID" \  # Same volume!
  --spot \
  --bid 0.40 \
  --imageName "your-registry/comfyui-batch:latest" \
  --args "python /workspace/run_batch_generation.py"

# Script will resume from checkpoint
```

#### 4.2 Manual Termination
```bash
# Check if generation is complete
ssh pod
ls /workspace/outputs/experiment_*/  | wc -l

# Terminate pod when done
runpodctl stop pod $POD_ID
# OR fully remove
runpodctl remove pod $POD_ID
```

## ComfyUI Alternative Workflow

**Using ComfyUI GUI**:
```bash
# Deploy pod with ComfyUI + web interface
runpodctl create pods \
  --name "comfyui-interactive" \
  --gpuType "NVIDIA RTX 4090" \
  --networkVolumeId "$NETWORK_VOLUME_ID" \
  --imageName "runpod/stable-diffusion:comfy-ui-3.0.1" \
  --ports "8188/http,22/tcp" \
  --env "LORA_DIR=/workspace/loras"

# Access ComfyUI at: https://POD_ID-8188.proxy.runpod.net
# Load LoRAs from /workspace/loras
# Queue batch jobs via API or GUI
```

**ComfyUI API workflow**:
```python
import requests
import json

# ComfyUI API endpoint
COMFY_URL = f"https://{pod_id}-8188.proxy.runpod.net"

# Load workflow JSON
with open('/workspace/configs/comfy_workflow.json', 'r') as f:
    workflow = json.load(f)

# Queue prompts in batch
for prompt in prompts:
    # Update workflow with new prompt
    workflow['6']['inputs']['text'] = prompt
    
    # Queue job
    response = requests.post(
        f"{COMFY_URL}/prompt",
        json={"prompt": workflow}
    )
    job_id = response.json()['prompt_id']
    print(f"Queued: {job_id}")
```

## Optimization Tips

### GPU Selection for Inference
| GPU | VRAM | Cost/hr (Spot) | Best For |
|-----|------|----------------|----------|
| RTX 4090 | 24GB | ~$0.35 | SDXL + LoRAs, best value |
| A40 | 48GB | ~$0.45 | Multiple LoRAs, larger batches |
| A100 40GB | 40GB | ~$1.10 | Speed-critical, parallel batches |

### Batch Size Tuning
```python
# Measure throughput with different batch sizes
batch_sizes = [1, 2, 4, 8]
for bs in batch_sizes:
    start = time.time()
    # Generate batch
    elapsed = time.time() - start
    print(f"Batch size {bs}: {bs/elapsed:.2f} images/sec")
```

### Cost Estimation
```python
# Estimate generation cost
images_to_generate = 500
seconds_per_image = 25  # Measured from test run
gpu_cost_per_hour = 0.35  # Spot RTX 4090

total_hours = (images_to_generate * seconds_per_image) / 3600
estimated_cost = total_hours * gpu_cost_per_hour

print(f"Estimated cost: ${estimated_cost:.2f} for {images_to_generate} images")
print(f"Cost per image: ${estimated_cost/images_to_generate:.4f}")
```

## Troubleshooting

**Problem: Spot pod keeps getting interrupted**
```bash
# Solution: Increase bid price or switch to on-demand
--bid 0.50  # Higher bid = less likely to be interrupted
# OR
# Remove --spot flag for on-demand pricing
```

**Problem: Out of VRAM during generation**
```bash
# Reduce batch size
batch_size: 1

# Use model optimization
pipeline.enable_model_cpu_offload()
pipeline.enable_vae_slicing()
pipeline.enable_attention_slicing()
```

**Problem: Outputs not saving to volume**
```bash
# Verify mount
df -h | grep workspace

# Check permissions
chmod 755 /workspace/outputs

# Test write
echo "test" > /workspace/outputs/test.txt
```

## Next Steps

- **Fine-tuning workflow**: See [finetuning-workflow.md](finetuning-workflow.md)
- **Network volume management**: See [../storage/network-volumes.md](../storage/network-volumes.md)
- **Template customization**: See [../templates/inference-templates.md](../templates/inference-templates.md)
- **Cost optimization**: See [../reference/cost-optimization.md](../reference/cost-optimization.md)
