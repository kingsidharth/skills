# Windowed & Neighborhood Attention

Localized attention patterns for vision and video transformers. Three main approaches with different trade-offs.

## NATTEN (Neighborhood Attention Extension)

Sliding-window attention operating pixel-wise (not window-partitioned). Custom CUDA kernels built on CUTLASS + xFormers FMHA.

**Advantages over Swin WSA:**
- Preserves translational equivariance (no shift trick needed)
- Receptive field grows naturally without extra ops
- 40% faster, 25% less memory than Swin WSA
- Fused kernels: up to 16× speedup over naive, constant global memory regardless of neighborhood size

**Supports**: 1D, 2D, 3D neighborhoods. Dilated variant (DiNA) for sparse global attention.

```bash
pip install natten
```

```python
from natten import NeighborhoodAttention2D

# 2D spatial neighborhood attention
na = NeighborhoodAttention2D(
    dim=256,
    num_heads=8,
    kernel_size=7,      # neighborhood window
    dilation=1,         # 1=local, >1=dilated/sparse
)
out = na(x)  # x: (B, H, W, C)
```

**DiT usage**: replaces global self-attention in spatial DiT blocks. Each token attends only to its local neighborhood, reducing quadratic complexity to linear. Stack with dilated layers for global reach.

**2025 update**: Generalized Neighborhood Attention paper adds multi-dimensional sparse patterns at near-peak bandwidth.

## Swin-style Window Attention

Non-overlapping window partition + shifted windows for cross-window interaction.

- Simpler implementation (window partition is just a reshape)
- Requires shift trick for inter-window communication
- Breaks translational equivariance
- Well-supported in PyTorch via `F.scaled_dot_product_attention` within each window

For new DiT work, NATTEN or FlexAttention-based windowed masks are generally preferred over Swin WSA.

## FlexAttention Windowed Masks

FlexAttention can express neighborhood/windowed attention in pure Python, compiled to fused kernels:

```python
from torch.nn.attention.flex_attention import flex_attention, create_block_mask

# 1D sliding window
def sliding_window_1d(b, h, q_idx, kv_idx):
    return abs(q_idx - kv_idx) <= WINDOW

# 2D neighborhood (for image patches in raster order)
def neighborhood_2d(b, h, q_idx, kv_idx):
    q_row, q_col = q_idx // W, q_idx % W
    k_row, k_col = kv_idx // W, kv_idx % W
    return (abs(q_row - k_row) <= R) & (abs(q_col - k_col) <= R)
```

Trade-off vs NATTEN: FlexAttention is more flexible and composable (add causal, document masking, etc.), but NATTEN's dedicated CUDA kernels can be faster for pure neighborhood patterns.

## Sliding Tile Attention (STA)

Purpose-built for 3D spatiotemporal attention in video DiTs. Operates on tiles (not individual tokens) with hardware-aware sliding window design.

- 2.8–17× over FlashAttention-2, 1.6–10× over FA3
- 58.79% MFU (memory-bandwidth utilization)
- Training-free: drop into pretrained video DiTs with no quality loss
- With fine-tuning: further latency reduction (HunyuanVideo 945s → 268s)

Key insight: attention in pretrained video DiTs concentrates within local 3D windows. STA exploits this without approximation.

Repo: `github.com/NUS-HPC-AI-Lab/STA`

See also: [DIT_KERNELS.md](DIT_KERNELS.md) for more video DiT–specific approaches.
