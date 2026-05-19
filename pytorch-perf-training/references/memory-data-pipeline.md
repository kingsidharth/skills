# Memory & Data Pipeline

## Create tensors on device

```python
# BAD — creates on CPU, then copies
x = torch.randn(B, C, H, W).cuda()

# GOOD — creates directly on GPU
x = torch.randn(B, C, H, W, device="cuda")
```

Same for `torch.zeros`, `torch.ones`, `torch.full`, `torch.empty`, `torch.arange`, etc.

## Pre-allocation for variable-length inputs

Variable-length sequences cause the caching allocator to fragment memory. Pre-allocate to the maximum expected size:

```python
# Pre-allocate once
max_seq_len = 256
static_tokens = torch.zeros(B, max_seq_len, D, device="cuda")
static_mask = torch.zeros(B, max_seq_len, device="cuda", dtype=torch.bool)

# Each iteration: copy actual data, update mask
actual_len = tokens.shape[1]
static_tokens[:, :actual_len] = tokens
static_mask[:, :actual_len] = True
```

This avoids repeated alloc/dealloc and is required for CUDA graph compatibility.

## Avoiding CPU-GPU synchronization

Operations that force the CPU to wait for GPU:

| Operation | Why it syncs | Fix |
|-----------|-------------|-----|
| `tensor.item()` | Copies scalar to CPU | Log every N steps, outside compiled region |
| `print(tensor)` | Copies to CPU for display | Use `print(tensor.shape)` instead |
| `tensor.cpu()` / `tensor.numpy()` | Explicit transfer | Defer to end of step |
| `if tensor > 0:` | Python needs the bool value | Rewrite as tensor ops |
| `torch.nonzero()` (data-dependent shape) | Output size depends on data | Use masks instead |
| `assert tensor.max() < 100` | Evaluates tensor on CPU | Remove from hot path |
| `len(tensor.unique())` | Data-dependent | Profile separately |

## DataLoader

```python
loader = DataLoader(
    dataset,
    batch_size=B,
    num_workers=4,           # tune: usually 2-4 per GPU
    pin_memory=True,         # enables async CPU→GPU transfer
    persistent_workers=True, # keeps workers alive between epochs
    prefetch_factor=2,       # number of batches pre-loaded per worker
    drop_last=True,          # avoids short last batch (recompilation trigger)
)
```

`drop_last=True` is critical for compiled training — a shorter final batch triggers recompilation.

## Gradient checkpointing (activation checkpointing)

Trades compute for memory. Re-computes intermediate activations during backward instead of storing them:

```python
from torch.utils.checkpoint import checkpoint

class DiTBlock(nn.Module):
    def forward(self, x, t):
        # Checkpoint this block — saves memory proportional to block count
        return checkpoint(self._forward, x, t, use_reentrant=False)

    def _forward(self, x, t):
        # actual computation
        ...
```

`use_reentrant=False` is required for torch.compile compatibility.

For DiT with N blocks: checkpointing all blocks roughly halves activation memory at ~30% compute overhead.

## Memory-efficient attention

```python
# PyTorch 2.0+ SDPA — auto-selects FlashAttention / MemEfficient / Math
from torch.nn.functional import scaled_dot_product_attention

out = scaled_dot_product_attention(q, k, v, attn_mask=mask, is_causal=False)
```

SDPA automatically selects the fastest kernel. torch.compile can fuse operations around it.

## Gradient accumulation

```python
accumulation_steps = 4
for i, batch in enumerate(loader):
    with torch.amp.autocast("cuda", dtype=torch.bfloat16):
        loss = model(batch) / accumulation_steps
    loss.backward()

    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
```

Note: `zero_grad(set_to_none=True)` avoids a memset kernel — uses assignment instead of addition in the next backward.

## CUDA memory pool configuration

```python
# Expand the memory pool fraction (if OOM on a mostly-free GPU)
torch.cuda.set_per_process_memory_fraction(0.95)

# Release unused cached memory (useful between training phases)
torch.cuda.empty_cache()
```
