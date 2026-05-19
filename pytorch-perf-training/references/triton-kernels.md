# Custom Triton Kernels with torch.compile

Use when Inductor's auto-generated kernels underperform for a specific operation (verified via profiling).

## Integration with torch.compile

Triton kernels are first-class citizens in torch.compile. Decorate with `@triton.jit` and call normally — Inductor traces through them.

```python
import triton
import triton.language as tl

@triton.jit
def fused_adaln_kernel(
    X_ptr, Scale_ptr, Shift_ptr, Out_ptr,
    N: tl.constexpr,
    BLOCK: tl.constexpr,
):
    pid = tl.program_id(0)
    offs = pid * BLOCK + tl.arange(0, BLOCK)
    mask = offs < N

    x = tl.load(X_ptr + offs, mask=mask)
    scale = tl.load(Scale_ptr + offs, mask=mask)
    shift = tl.load(Shift_ptr + offs, mask=mask)

    out = x * (1 + scale) + shift
    tl.store(Out_ptr + offs, out, mask=mask)
```

## Wrapping for torch.compile

```python
# Register as a custom op for torch.compile compatibility
@torch.library.custom_op("mylib::fused_adaln", mutates_args=())
def fused_adaln(x: torch.Tensor, scale: torch.Tensor, shift: torch.Tensor) -> torch.Tensor:
    out = torch.empty_like(x)
    N = x.numel()
    BLOCK = 1024
    grid = (triton.cdiv(N, BLOCK),)
    fused_adaln_kernel[grid](x, scale, shift, out, N, BLOCK)
    return out

# Register fake (meta) tensor implementation for tracing
@fused_adaln.register_fake
def fused_adaln_fake(x, scale, shift):
    return torch.empty_like(x)

# Now works inside compiled model
@torch.compile
def forward(x, t_emb):
    scale, shift = t_emb.chunk(2, dim=-1)
    return fused_adaln(x, scale, shift)
```

## When to write custom Triton kernels

- **Fused adaLN-Zero modulation**: scale + shift + gate in one kernel (DiT's hot path)
- **Fused bias + activation**: when Inductor doesn't fuse them
- **Custom attention variants**: sparse attention, windowed attention
- **RoPE encoding**: fused rotary position embedding
- **Custom loss functions**: when the default decomposition is suboptimal

## When NOT to write custom kernels

- Standard matmuls (cuBLAS / Triton autotuned by Inductor)
- Standard attention (SDPA already selects FlashAttention)
- Operations where Inductor's codegen matches or exceeds hand-written Triton

## Autotuning Triton kernels

```python
@triton.autotune(
    configs=[
        triton.Config({"BLOCK": 256}, num_warps=4),
        triton.Config({"BLOCK": 512}, num_warps=8),
        triton.Config({"BLOCK": 1024}, num_warps=8),
    ],
    key=["N"],
)
@triton.jit
def my_kernel(X_ptr, Out_ptr, N: tl.constexpr, BLOCK: tl.constexpr):
    ...
```

Autotune benchmarks each config and caches the best. Works with torch.compile.

## Debugging Triton kernels

```python
# Interpret mode — runs on CPU for debugging
os.environ["TRITON_INTERPRET"] = "1"

# Print generated PTX
os.environ["TRITON_PRINT_AUTOTUNING"] = "1"

# Verify correctness against PyTorch reference
torch.testing.assert_close(triton_output, pytorch_reference, atol=1e-3, rtol=1e-3)
```
