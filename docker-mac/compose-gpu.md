# Compose GPU support

NVIDIA GPUs, via `deploy.resources.reservations.devices`. Requires the NVIDIA Container Toolkit on the host.

On Mac: **no GPU passthrough works**. Neither Docker Desktop nor OrbStack can give containers access to the Mac GPU (Metal isn't exposable). For ML on Mac, run natively or use cloud GPU (RunPod, Modal, etc.). GPU-in-container is a Linux-host story.

## Host setup (Linux)

```bash
# Install NVIDIA Container Toolkit (Ubuntu/Debian)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu22.04 nvidia-smi
```

## `docker run` direct

```bash
docker run --gpus all nvidia/cuda:12.6.0-base nvidia-smi
docker run --gpus '"device=0,1"' ...        # specific GPUs
docker run --gpus '"count=2"' ...           # any 2 GPUs
```

## Compose

```yaml
services:
  trainer:
    image: pytorch/pytorch:2.5-cuda12.4-cudnn9-runtime
    command: python train.py
    volumes:
      - ./data:/data
      - ./checkpoints:/checkpoints
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all         # or: count: 1, or capabilities only
              capabilities: [gpu]
```

Specific GPUs by index / UUID:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          device_ids: ["0", "1"]    # or ["GPU-abc123..."]
          capabilities: [gpu, compute, utility]
```

## `capabilities`

| Capability | Purpose |
|---|---|
| `gpu` | basic CUDA access |
| `compute` | CUDA compute |
| `utility` | nvidia-smi, monitoring |
| `graphics` | OpenGL, Vulkan |
| `video` | video encoding/decoding |

`[gpu]` alone is usually enough.

## Common patterns

### PyTorch training

```yaml
services:
  train:
    image: pytorch/pytorch:2.5-cuda12.4-cudnn9-runtime
    ipc: host                      # required for some dataloaders
    shm_size: 16gb                 # default 64M is tiny
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ./:/workspace
    working_dir: /workspace
    command: python train.py
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### vLLM inference

```yaml
services:
  vllm:
    image: vllm/vllm-openai:latest
    ports: ["8000:8000"]
    ipc: host
    volumes:
      - ~/.cache/huggingface:/root/.cache/huggingface
    environment:
      HF_TOKEN: ${HF_TOKEN}
    command: >
      --model meta-llama/Llama-3.1-8B-Instruct
      --gpu-memory-utilization 0.9
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### Ollama

```yaml
services:
  ollama:
    image: ollama/ollama
    ports: ["11434:11434"]
    volumes:
      - ollama:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

volumes:
  ollama:
```

## Alternative for Mac dev → Linux deploy

Workflow: write code against an Ollama/llama.cpp container on Mac (CPU or Metal natively), deploy the same Compose file to a Linux box with the GPU block. The GPU block is ignored silently on hosts without NVIDIA drivers — so one file works both places.

Even cleaner: put the GPU block under a `profile`:

```yaml
services:
  trainer:
    image: pytorch/pytorch:...
    profiles: [gpu]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

Then `docker compose --profile gpu up` on the GPU host, plain `docker compose up` on Mac.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `could not select device driver "nvidia"` | Container toolkit not installed or daemon not restarted |
| `nvidia-smi` works on host, not in container | Add `--gpus all` / GPU block |
| CUDA version mismatch errors | Image CUDA version must be ≤ host driver's supported CUDA |
| Out of shared memory in PyTorch | Set `shm_size: 16gb` and `ipc: host` |
| Slow data loading | Bind-mount dataset dir; avoid volumes for large datasets on some drivers |
