# FlexAttention — Composable Fused Attention

PyTorch ≥2.5. Compiler-driven API that generates fused FlashAttention kernels from Python `score_mod`/`mask_mod` functions via `torch.compile` + Triton codegen.

## Why it matters

Custom attention variants (sliding window, paged, neighborhood, document masking) previously required hand-written CUDA kernels. FlexAttention lets you define them in pure Python and compile to performance matching hand-tuned FA kernels. Patterns compose freely — sliding window + causal + document masking + paged in one fused kernel.

## Core API

```python
from torch.nn.attention.flex_attention import (
    flex_attention,
    create_block_mask,
)

# Define a mask_mod: (batch, head, q_idx, kv_idx) → bool
def causal_mask(b, h, q_idx, kv_idx):
    return q_idx >= kv_idx

block_mask = create_block_mask(causal_mask, B, H, SEQ_Q, SEQ_KV)
out = flex_attention(q, k, v, block_mask=block_mask)
```

`create_block_mask` precomputes a `BlockMask` — a sparse index structure that skips fully-masked blocks without loading them. Partially masked blocks still apply `mask_mod` element-wise.

## Common patterns

```python
# Sliding window
def sliding_window(b, h, q_idx, kv_idx):
    return abs(q_idx - kv_idx) <= WINDOW_SIZE

# Causal + sliding window
def causal_sliding(b, h, q_idx, kv_idx):
    return (q_idx >= kv_idx) & (q_idx - kv_idx <= WINDOW_SIZE)

# Document masking (sample packing)
def document_mask(b, h, q_idx, kv_idx):
    return document_ids[q_idx] == document_ids[kv_idx]

# Prefix LM (bidirectional prefix, causal suffix)
def prefix_lm(b, h, q_idx, kv_idx):
    return (kv_idx <= prefix_length) | (q_idx >= kv_idx)
```

## Score modifications

```python
# ALiBi bias
def alibi_score(score, b, h, q_idx, kv_idx):
    return score - slopes[h] * abs(q_idx - kv_idx)

# Tanh soft-capping
def soft_cap(score, b, h, q_idx, kv_idx):
    return CAP * torch.tanh(score / CAP)

out = flex_attention(q, k, v, score_mod=alibi_score, block_mask=block_mask)
```

## PagedAttention

FlexAttention natively supports paged KV cache via `BlockMask` index remapping. The `attention-gym` repo has a full implementation: page table management, logical→physical conversion, mask/score mod adapters.

Key repo: `github.com/meta-pytorch/attention-gym`

Patterns available in attention-gym: Causal, Sliding Window, Document Masking, Prefix LM, NATTEN-style Neighborhood, Dilated, Paged Attention, ALiBi, Tanh Soft-Capping, MLA RoPE.

## Performance

- Forward: 0.68–1.43× of FlashAttention-2
- Decoding: 0.93–1.45× of FAKV
- End-to-end training (LLaMA3): 2.4× speedup
- End-to-end inference (LLaMA3.1): 2.04× speedup
- PagedAttention overhead: <5% over non-paged FlexAttention; can be faster than FA2 without paging on long sequences

## Checkpointing caveat

Nested function definitions inside `forward()` break `torch.utils.checkpoint`. Define mask/score functions at module level or as static methods.
