# DiT-Specific Kernel Optimizations

Kernel strategies for Diffusion Transformer architectures (DiT, MMDiT, video DiTs, S3DiT-like models).

## Standard DiT block mapping

```
Input → LayerNorm → AdaLN Modulate → Self-Attention → Residual
      → LayerNorm → AdaLN Modulate → MLP → Residual
```

| Component | Recommended kernel | Source |
|-----------|-------------------|--------|
| LayerNorm / RMSNorm | Liger fused kernel | `liger-kernel` |
| AdaLN (scale + shift + gate) | `torch.compile` auto-fuses | PyTorch native |
| Self-Attention (vanilla) | SDPA → Flash backend | PyTorch native |
| Self-Attention (custom mask) | FlexAttention | PyTorch ≥2.5 |
| Spatial windowed attention | NATTEN or FlexAttention | `natten` / native |
| MLP (GELU) | Liger GeGLU or flash-attn FusedMLP | `liger-kernel` / `flash-attn` |
| MLP (SwiGLU) | Liger SwiGLU | `liger-kernel` |
| RoPE (if used) | Liger or Unsloth fused | `liger-kernel` / `unsloth` |
| Final loss | FusedLinearCrossEntropy | `liger-kernel` |

## Image DiT optimizations

**DyDiT (Dynamic DiT)** — plug-and-play modules:
- Timestep-wise Dynamic Width (TDW): learns to prune attention heads + MLP channels per timestep. Architecture is pre-determined offline, no runtime overhead.
- Spatial Dynamic Token (SDT): skips "easy" patches that bypass compute blocks. Hardware-friendly gather/scatter with minimal overhead.

**DiTFastAttn / DiTFastAttnV2** — headwise attention compression:
- Analyzes per-head attention patterns (some heads are local-spatial, some cross-shaped, some global)
- Applies headwise arrow attention + dynamic caching
- Custom fused kernel for compressed attention
- Result: 68% attention FLOP reduction, 1.5× end-to-end speedup on 2K generation

**Chipmunk** — column-sparse delta caching:
- Exploits temporal redundancy across denoising steps
- Fused kernels compute attention output + sparsity pattern simultaneously
- Works on both attention and MLP layers
- Triton kernels included for cross-GPU portability

## Video DiT / S3DiT optimizations

Video DiTs use 3D spatiotemporal attention over flattened frame sequences. At 720p, this means 100K+ tokens where attention dominates 68–77% of total latency.

### Sliding Tile Attention (STA)

Best training-free acceleration for video DiTs. See [WINDOWED_ATTENTION.md](WINDOWED_ATTENTION.md).

### Sparse VideoGen

Spatial-temporal sparsity exploitation:
- Identifies structured patterns: spatial-local heads, temporal-local heads, cross-shaped heads, global heads
- Per-head sparse attention schemes matched to discovered patterns
- Compatible with FlexAttention mask composition

### Compact Attention

Discovers periodic hierarchical attention patterns in video DiTs:
- Specialized head types with distinct functional roles
- Local spatial, cross-shaped spatial, frame-distance-aware temporal
- Adaptive sparse attention based on head classification

### FlexAttention for video DiTs

Compose spatial + temporal patterns in one fused kernel:

```python
# Spatial-local + temporal-causal composed
def video_attention(b, h, q_idx, kv_idx):
    q_frame = q_idx // (H * W)
    k_frame = kv_idx // (H * W)
    q_spatial = q_idx % (H * W)
    k_spatial = kv_idx % (H * W)

    # Temporal: causal or windowed
    temporal_ok = abs(q_frame - k_frame) <= T_WINDOW

    # Spatial: neighborhood within same frame
    q_row, q_col = q_spatial // W, q_spatial % W
    k_row, k_col = k_spatial // W, k_spatial % W
    spatial_ok = (abs(q_row - k_row) <= S_RADIUS) & (abs(q_col - k_col) <= S_RADIUS)

    # Cross-frame: only allow spatial overlap
    same_frame = q_frame == k_frame
    return (same_frame & spatial_ok) | (~same_frame & temporal_ok & spatial_ok)
```

### Separated spatial + temporal

Some architectures (pre-S3DiT era) separate spatial and temporal attention. Each can independently use:
- Spatial: NATTEN 2D neighborhood attention
- Temporal: standard causal or windowed via SDPA/FlexAttention
- Cross-attention (text conditioning): standard SDPA

### Sequence parallelism for long video

For sequences exceeding single-GPU memory, combine ring attention / context parallelism with FlexAttention. Causal-RoPE SP variants enable distributed 3D attention with localized computation and minimal cross-rank communication.

## Practical Colab A100 stack for DiT training

```python
import torch
import torch.nn.functional as F

# 1. Performance settings
torch.backends.cuda.matmul.allow_tf32 = True
torch.backends.cudnn.allow_tf32 = True

# 2. SDPA auto-selects flash on A100
attn_out = F.scaled_dot_product_attention(q, k, v)

# 3. Liger for fused norms + MLP
from liger_kernel.ops.rms_norm import LigerRMSNorm
from liger_kernel.ops.swiglu import LigerSwiGLU

# 4. Fused optimizer
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4, fused=True)

# 5. Compile the model
model = torch.compile(model)
```
