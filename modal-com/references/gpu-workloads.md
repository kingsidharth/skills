# GPU Workloads Reference

## GPU Selection

**Available GPUs**:
- T4 (16GB) - Cheapest, development
- A10G (24GB) - Mid-tier, good for most inference
- L4 (24GB) - Efficient, Ada Lovelace
- L40S (48GB) - Large models, FP8 support
- H100 (80GB) - Training, frontier models
- H200 (141GB) - Massive models, 1.4x bandwidth
- B200/B300 - Cutting edge (limited software support)

**Selection**:
```python
@app.function(gpu="A10G")  # Single GPU
@app.function(gpu=modal.gpu.H100(count=4))  # Multi-GPU
```

## CUDA Setup

Modal has NVIDIA driver pre-installed. You need to install CUDA toolkit:

```python
# Via pip (recommended)
image = modal.Image.debian_slim() \
    .pip_install("torch")  # Bundles CUDA runtime

# From NVIDIA base
image = modal.Image.from_registry(
    "nvidia/cuda:12.4.0-devel-ubuntu22.04",
    add_python="3.11"
)
```

## Memory Snapshots (CRITICAL)

**Enable for 5-10x faster cold starts**:

```python
@app.function(
    gpu="A10G",
    enable_memory_snapshot=True,  # CPU snapshot
    experimental_options={"enable_gpu_snapshot": True}  # GPU snapshot (alpha)
)
def inference(prompt: str):
    # Model loading happens once, then snapshotted
    return model.generate(prompt)
```

**GPU Snapshots** capture compiled kernels, CUDA graphs, torch.compile results.

## LLM Inference Pattern

```python
model_vol = modal.Volume.from_name("llm-models", create_if_missing=True)

# Download model (once)
@app.function(volumes={"/models": model_vol}, timeout=3600)
def download_model(repo_id: str):
    from huggingface_hub import snapshot_download
    snapshot_download(repo_id, local_dir="/models")
    model_vol.commit()

# Inference with GPU snapshot
@app.function(
    gpu="H100",
    volumes={"/models": model_vol},
    enable_memory_snapshot=True,
    experimental_options={"enable_gpu_snapshot": True}
)
def generate(prompt: str):
    from transformers import AutoModelForCausalLM
    model = AutoModelForCausalLM.from_pretrained("/models", device_map="auto")
    return model.generate(...)
```

## Image Generation Pattern

```python
@app.function(
    gpu="A10G",
    enable_memory_snapshot=True,
    experimental_options={"enable_gpu_snapshot": True}
)
def generate_image(prompt: str):
    from diffusers import FluxPipeline
    pipe = FluxPipeline.from_pretrained(...).to("cuda")
    return pipe(prompt).images[0]
```

## Concurrent Model Loading

**Fast** (concurrent):
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor() as executor:
    models = [
        executor.submit(load, "model-a"),
        executor.submit(load, "model-b")
    ]
    return [f.result() for f in models]
```

**Slow** (sequential) - avoid this!

## Multi-GPU Training

```python
@app.function(gpu=modal.gpu.H100(count=4))
def train():
    # PyTorch DDP auto-detects multiple GPUs
    pass
```

## GPU Monitoring

```python
@app.function(gpu="A10G")
def monitor():
    import subprocess
    result = subprocess.run(["nvidia-smi"], capture_output=True)
    print(result.stdout.decode())
```

**Dashboard**: View GPU metrics at modal.com (utilization, memory, temperature)

