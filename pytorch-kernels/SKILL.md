---
name: pytorch-kernels
description: Custom and community GPU kernels for PyTorch transformer training and inference. Use when optimizing attention (FlashAttention, FlexAttention, SDPA, NATTEN, sliding window, paged, neighborhood), MLP fusion (Liger, SwiGLU, GeGLU, FusedMLP), normalization (RMSNorm, LayerNorm), RoPE, CrossEntropy, or optimizers (fused AdamW). Especially relevant for DiT, MMDiT, video DiT, S3DiT architectures. Also covers Colab A100 setup for flash-attn, torch.compile kernel fusion, and Triton kernel libraries like Unsloth and Liger Kernel.
---

# PyTorch Custom & Community Kernels

> Target: Ampere+ GPUs (A100, H100). PyTorch ≥2.0, ideally ≥2.5 for FlexAttention.

## When to use what

| Need | Go to |
|------|-------|
| Drop-in fast attention (any model) | [SDPA.md](references/SDPA.md) |
| Custom attention masks/patterns | [FLEX_ATTENTION.md](references/FLEX_ATTENTION.md) |
| Standalone FlashAttention library + FusedMLP | [FLASH_ATTENTION.md](references/FLASH_ATTENTION.md) |
| Spatial/windowed attention for vision/DiT | [WINDOWED_ATTENTION.md](references/WINDOWED_ATTENTION.md) |
| Fused norm/MLP/RoPE/loss kernels | [FUSED_LAYERS.md](references/FUSED_LAYERS.md) |
| DiT/video-DiT specific optimizations | [DIT_KERNELS.md](references/DIT_KERNELS.md) |
| Colab A100 install recipes | [FLASH_ATTENTION.md § Colab Install](references/FLASH_ATTENTION.md) |

## Decision flow

1. **Always start with SDPA** — `F.scaled_dot_product_attention` auto-dispatches to FlashAttention on Ampere+ with zero setup
2. **Need custom masks** (sliding window, paged, document packing, neighborhood)? → FlexAttention composes them in pure Python, compiles to fused Triton
3. **Need FusedMLP, RoPE, extra ops** from flash-attn library? → standalone `flash-attn` package
4. **Training throughput** — drop in Liger Kernel for fused RMSNorm/SwiGLU/CrossEntropy (~20% throughput, ~60% memory)
5. **Vision/video DiT** — NATTEN for 2D/3D neighborhood attention, STA for video DiTs, FlexAttention for composable spatial-temporal masks
