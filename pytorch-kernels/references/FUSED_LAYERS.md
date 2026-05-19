# Fused Layer Kernels

Triton/CUDA kernels that fuse multiple pointwise ops into single kernel launches, reducing memory traffic and launch overhead.

## Liger Kernel (LinkedIn)

Drop-in Triton kernels for transformer training. ~20% throughput gain, ~60% memory reduction. Works with FlashAttention, FSDP, DeepSpeed.

```bash
pip install liger-kernel
```

### Available kernels

| Kernel | Speedup | Memory | What it fuses |
|--------|---------|--------|---------------|
| RMSNorm | ~7× | ~3× less | norm + scale in one pass, caches RMS for backward |
| LayerNorm | ~7× | ~3× less | same approach, caches inverse std |
| SwiGLU | speed parity | ~2× less | gate + activation + element-wise multiply |
| GeGLU | speed parity | ~2× less | same for GELU gate variant |
| RoPE | fused | in-place | rotary embedding, no intermediate allocs |
| CrossEntropy | fused | significant | online softmax + loss in one kernel |
| FusedLinearCrossEntropy | chunked | massive | skips full logit materialization entirely |

### HuggingFace integration (one-liner)

```python
from liger_kernel.transformers import apply_liger_kernel_to_llama
apply_liger_kernel_to_llama(
    rope=True, swiglu=True, cross_entropy=True,
    fused_linear_cross_entropy=False, rms_norm=True,
)
model = AutoModelForCausalLM.from_pretrained("path/to/model")
```

### Custom model integration

```python
from liger_kernel.ops.rms_norm import LigerRMSNorm
from liger_kernel.ops.swiglu import LigerSwiGLU
from liger_kernel.ops.cross_entropy import LigerCrossEntropyLoss

# Replace nn.LayerNorm / custom RMSNorm with:
norm = LigerRMSNorm(hidden_size)

# Replace MLP activation with:
swiglu = LigerSwiGLU()
```

### Post-training kernels

Up to 80% memory savings for alignment losses: DPO, CPO, ORPO, SimPO, KTO, JSD.

## Unsloth Triton Kernels

Focused on fine-tuning. Key innovations:

- **Fused QK-RoPE**: merges Q and K rotary embedding into one in-place kernel. 2.3× faster (long seq), 1.9× (short seq). Zero extra VRAM.
- **SwiGLU/GeGLU**: fused with int64 indexing for long-context support
- **Uncontaminated packing**: padding-free training with per-sample attention masking, compatible with FA3/xFormers/SDPA backends

Claims: ~3× faster training, 30–90% less VRAM. Note: independent benchmarks (Chronicals paper) dispute some throughput numbers — verify gradient flow when benchmarking.

## torch.compile (TorchInductor)

PyTorch's compiler auto-fuses eligible pointwise + reduction ops into Triton kernels. No manual kernel writing needed.

**What it fuses well:**
- LayerNorm + residual add
- Activation functions (GELU, SiLU, etc.)
- AdaLN modulation (scale/shift/gate) — common in DiT blocks
- Dropout + add sequences
- Elementwise chains

**What it delegates to existing optimized kernels:**
- Linear layers → cuBLAS
- Attention → FlashAttention/SDPA
- Conv layers → cuDNN

```python
model = torch.compile(model, mode="reduce-overhead")  # best for training
model = torch.compile(model, mode="max-autotune")      # tries more configs
```

## Fused Optimizers

PyTorch optimizer implementations, from slowest to fastest:

| Tier | Name | Mechanism |
|------|------|-----------|
| 1 | for-loop | Per-parameter kernel launches |
| 2 | foreach | Multi-tensor, horizontal fusion |
| 3 | fused | Single kernel, vertical + horizontal fusion |

```python
# Fused AdamW — single kernel per step
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4, fused=True)

# Or compile the step for additional fusion with surrounding ops
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)
compiled_step = torch.compile(optimizer.step)
```

`fused=True` requires CUDA tensors. `foreach=True` is the fallback default when available.

## Performance settings

```python
# Enable TF32 for matmuls (free ~3× on Ampere+, slight precision trade-off)
torch.backends.cuda.matmul.allow_tf32 = True
torch.backends.cudnn.allow_tf32 = True

# Enable cuDNN autotuner (finds fastest conv algorithms)
torch.backends.cudnn.benchmark = True

# Expandable memory segments (reduces fragmentation)
import os
os.environ['PYTORCH_CUDA_ALLOC_CONF'] = 'expandable_segments:True'
```
