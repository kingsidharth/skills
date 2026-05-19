# FlashAttention Library (Dao-AILab)

Standalone package with hand-tuned CUDA kernels for attention + supplementary ops (FusedMLP, RoPE, LayerNorm, CrossEntropy).

## Versions

| Version | GPU | Key features |
|---------|-----|-------------|
| FA-2 | Ampere (A100) | fp16/bf16, head dims ≤256, 50–73% of A100 peak |
| FA-3 | Hopper (H100) | TMA/WGMMA async, FP8, 840 TFLOPS (85% utilization) |
| FA-4 | Hopper + Blackwell | CuTeDSL-based, H100/B200 |

Head dim >192 backward requires A100/A800 or H100/H800. Head dim 256 backward works on consumer GPUs (no dropout) since v2.5.5.

## Colab A100 install

`pip install flash-attn` compiles from source and can hang for hours. Two reliable alternatives:

**Option A — Pre-built wheel (fastest)**
```bash
# Check your environment
python -c "import torch; print(torch.__version__, torch.version.cuda)"
# e.g. 2.4.1 cu121

# Go to https://github.com/Dao-AILab/flash-attention/releases
# Download the wheel matching your torch + CUDA + Python version
pip install flash_attn-2.x.x+cu12xtorch2.4-cp310-...-linux_x86_64.whl
```

**Option B — Build from source (ensure ninja works)**
```bash
pip install ninja packaging
ninja --version && echo $?  # must return 0
pip install flash-attn --no-build-isolation
# ~5 min on 64-core, much longer without ninja
```

## FusedMLP

Fuses `Linear → Activation → Linear` into 2–3 kernels (vs 5+ naive). Eliminates intermediate activation materialization.

```python
from flash_attn.ops.fused_dense import FusedMLP

mlp = FusedMLP(
    in_features=d_model,
    hidden_features=4 * d_model,
    activation="gelu",       # or "silu" for SwiGLU-style
    checkpoint_lvl=1,         # 0=none, 1=recompute act, 2=recompute all
)
```

Supports: GELU, SiLU/SwiGLU, ReLU. Tensor-parallel variant `ParallelFusedMLP` overlaps all-gather with computation.

## Other bundled ops

| Op | Module | Notes |
|----|--------|-------|
| RoPE | `flash_attn.layers.rotary` | Fused rotary embeddings |
| LayerNorm / RMSNorm | `flash_attn.ops.layer_norm` | Fused with residual |
| CrossEntropy | `flash_attn.losses.cross_entropy` | Memory-efficient |
| Dropout + Add + LayerNorm | `flash_attn.ops.layer_norm` | Single fused kernel |

## Direct API (bypassing SDPA)

```python
from flash_attn import flash_attn_func, flash_attn_varlen_func

# Fixed-length
out = flash_attn_func(q, k, v, causal=True, dropout_p=0.1)

# Variable-length (packed sequences)
out = flash_attn_varlen_func(
    q, k, v,
    cu_seqlens_q, cu_seqlens_k,
    max_seqlen_q, max_seqlen_k,
    causal=True,
)
```

Variable-length API avoids padding waste — essential for training with mixed-length sequences.
