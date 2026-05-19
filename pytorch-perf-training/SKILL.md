---
name: pytorch-perf-training
description: Optimise PyTorch single-GPU training wall-clock time. torch.compile, CUDA graphs, Tensor Cores, AMP, profiling, memory pre-allocation, Triton kernels, compiled optimizers, gradient monitoring, model visualization, benchmarking. Use when training DiT, transformers, diffusion models on GPU and user mentions speed, throughput, compilation, graph breaks, reduce-overhead, profiling, nsight, kernel fusion, gradient flow, vanishing/exploding gradients, hooks, torchinfo, torchviz, record_function, benchmark, or training step timing. Also trigger for looped/recursive transformers with shared weights, weight tying, depth-wise LoRA, or routing in looped blocks.
---

# PyTorch Performance Training

Single-GPU training optimisation targeting wall-clock time. Primary context: DiT-class models (S3DiT and similar), but techniques apply broadly to any transformer training.

## Optimisation stack (apply top-down)

1. **Profiling & diagnostics first** — measure before changing. See [profiling-diagnostics.md](references/profiling-diagnostics.md)
2. **torch.compile** — kernel fusion, graph capture, mode selection. See [torch-compile.md](references/torch-compile.md)
3. **CUDA graphs** — eliminate CPU launch overhead. See [cuda-graphs.md](references/cuda-graphs.md)
4. **Tensor Cores & AMP** — hardware-native mixed precision. See [tensor-cores-amp.md](references/tensor-cores-amp.md)
5. **Memory & data pipeline** — pre-allocation, pinned memory, async loading. See [memory-data-pipeline.md](references/memory-data-pipeline.md)
6. **Compiled optimizer & scheduler** — fuse optimizer into the graph. See [compiled-optimizer.md](references/compiled-optimizer.md)
7. **Custom Triton kernels** — when Inductor's codegen isn't enough. See [triton-kernels.md](references/triton-kernels.md)

## Profiler instrumentation

`record_function`, NVTX ranges, schedule tuning, output formats, compiled-code profiling. See [profiler-instrumentation.md](references/profiler-instrumentation.md)

## Gradient monitoring & model visualization

Gradient health checks, hooks (tensor + module), gradient flow plots, torchinfo/torchviz/torchview, TensorBoard integration, anomaly detection. See [gradient-model-viz.md](references/gradient-model-viz.md)

## Benchmarking recipes

Reproducible A/B timing, throughput metrics, memory benchmarks, compilation time, reporting templates. See [benchmarking-recipes.md](references/benchmarking-recipes.md)

## Looped / Recursive transformers

Weight-shared blocks, routing, depth-wise adaptation for DiT-like architectures. See [looped-transformers.md](references/looped-transformers.md)

## Quick checklist (DiT single-GPU)

Before any custom work, apply these in your training script:

```python
import torch

# 1. AMP + bf16 (Ampere+)
scaler = torch.amp.GradScaler() # only needed for fp16; bf16 doesn't need scaler
# 2. Compile the model
model = torch.compile(model, mode="max-autotune")
# 3. Compile the optimizer
optimizer.step = torch.compile(optimizer.step)
# 4. DataLoader
loader = DataLoader(..., num_workers=N, pin_memory=True, persistent_workers=True)
# 5. cuDNN autotuner
torch.backends.cudnn.benchmark = True
# 6. Gradient set-to-none
optimizer.zero_grad(set_to_none=True)
```

Then profile → identify bottleneck → apply targeted fix from the reference files.
