# SDPA — PyTorch Scaled Dot-Product Attention

`torch.nn.functional.scaled_dot_product_attention` — built into PyTorch ≥2.0. Single API, multiple fused backends selected automatically.

## Backends

| Backend | Constant | GPU Requirement | Notes |
|---------|----------|-----------------|-------|
| FlashAttention | `SDPBackend.FLASH_ATTENTION` | SM80+ (A100, RTX 30xx+) | fp16/bf16, head dims ≤256 |
| Memory-Efficient | `SDPBackend.EFFICIENT_ATTENTION` | Broad CUDA | xFormers-derived, supports arbitrary masks |
| cuDNN | `SDPBackend.CUDNN_ATTENTION` | SM80+ | Newer PyTorch builds |
| Math | `SDPBackend.MATH` | Any | Fallback, no fusion |

PyTorch picks the fastest compatible backend at runtime. Flash is preferred when constraints are met.

## Usage

```python
import torch
import torch.nn.functional as F

# Basic — backend auto-selected
out = F.scaled_dot_product_attention(q, k, v, is_causal=True)

# Force a specific backend
from torch.nn.attention import sdpa_kernel, SDPBackend

with sdpa_kernel(SDPBackend.FLASH_ATTENTION):
    out = F.scaled_dot_product_attention(q, k, v, is_causal=True)
```

## Checking availability

```python
# Global check
print(torch.backends.cuda.is_flash_attention_available())

# Per-input check (advanced)
print(torch.backends.cuda.can_use_flash_attention(params, debug=True))
print(torch.backends.cuda.can_use_efficient_attention(params, debug=True))
```

## Constraints for Flash backend

- Input dtype: fp16 or bf16 (bf16 needs Ampere+)
- Head dimensions: ≤256
- No arbitrary `attn_mask` tensor (use `is_causal=True` instead, or fall back to memory-efficient)
- Dropout supported

When an `attn_mask` tensor is passed, SDPA falls back to memory-efficient or math backend. For custom mask patterns without this penalty, use [FlexAttention](FLEX_ATTENTION.md).

## torch.compile integration

SDPA composes with `torch.compile` — the compiler fuses surrounding ops (LayerNorm, residual adds) around the SDPA call:

```python
@torch.compile
def attention_block(x, q, k, v):
    x = F.layer_norm(x, x.shape[-1:])
    out = F.scaled_dot_product_attention(q, k, v, is_causal=True)
    return x + out  # residual fused into surrounding kernel
```

## DiT note

Standard DiT models using vanilla self-attention get FlashAttention for free through SDPA — no library installs, no code changes. Just ensure inputs are fp16/bf16 on Ampere+ hardware.
