# torch.compile

## Modes

| Mode | What it does | When to use |
|------|-------------|-------------|
| `default` | Balanced fusion + codegen | First pass, safe baseline |
| `reduce-overhead` | Adds CUDA graphs automatically | Many small kernels, CPU-bound launch overhead |
| `max-autotune` | Benchmarks multiple kernel implementations (e.g. GEMM algorithms) | Compute-bound models, production training |
| `max-autotune-no-cudagraphs` | Autotune without CUDA graphs | When CUDA graphs cause issues (dynamic shapes, in-place mutations) |

For DiT training, start with `max-autotune`. It benchmarks Triton vs cuBLAS for matmuls and selects the fastest. If shapes are static (typical for DiT), `reduce-overhead` stacks on top.

## Compilation scope

```python
# Compile the whole model (recommended starting point)
model = torch.compile(model, mode="max-autotune")

# Regional: compile only the repeated block (faster cold start)
# Useful for DiT where the same block repeats N times
model.blocks = torch.compile(model.blocks, mode="max-autotune")

# Compile a single DiT block — kernels reused across all blocks
# Cuts compile time ~Nx for N blocks
for block in model.blocks:
    block.forward = torch.compile(block.forward, mode="max-autotune")
```

Regional compilation is especially valuable for DiT: identical transformer blocks share compiled kernels, so compile time drops from O(N_blocks) to O(1).

## fullgraph=True

Use during development to catch all graph breaks. Remove for production only if you intentionally accept breaks.

```python
model = torch.compile(model, fullgraph=True)  # errors on graph break
```

## Compilation caching

```python
# Persistent cache across runs (avoids re-compilation)
import torch._inductor.config
torch._inductor.config.fx_graph_cache = True

# Or via env var
# TORCHINDUCTOR_FX_GRAPH_CACHE=1
```

Cache location: `~/.cache/torch/inductor/`. Clear with `torch._inductor.utils.fresh_inductor_cache()`.

## Common DiT graph-break patterns and fixes

**Conditional logging/debug prints:**
```python
# BAD — graph break
if step % 100 == 0:
    print(f"loss: {loss.item()}")  # .item() syncs + break

# FIX — log outside compiled region
```

**Adaptive normalization with data-dependent branching:**
```python
# BAD — if tensor value determines path
if timestep > 500:
    x = self.norm1(x)

# FIX — keep both paths, select via mask/multiply
scale = (timestep > 500).float()
x = scale * self.norm1(x) + (1 - scale) * x
```

**Dynamic shape from variable sequence length:**
- Pad to fixed size, mask out padding
- Or use `torch.compile(dynamic=True)` at a speed cost

## Inductor options

```python
torch.compile(model, options={
    "triton.cudagraphs": True,           # auto CUDA graphs
    "epilogue_fusion": True,             # fuse pointwise into matmul epilogues
    "max_autotune": True,                # benchmark kernel variants
    "shape_padding": True,               # pad tensors for better alignment
    "coordinate_descent_tuning": True,   # extra tuning passes
})
```

## Compiled autograd

Captures the backward graph too, enabling cross-fwd-bwd fusion:

```python
# Enable compiled autograd (PyTorch 2.4+)
torch._dynamo.config.compiled_autograd = True
model = torch.compile(model)
```

## Warm-up

First 1-3 iterations trigger compilation + CUDA graph recording. Exclude these from timing:

```python
# Warm-up
for _ in range(3):
    with torch.no_grad():
        model(dummy_input)
# Now time
```

## Stance control

```python
# Switch between eager and compiled mid-training
torch.compiler.set_stance("eager")     # disable compile temporarily
torch.compiler.set_stance("default")   # re-enable
```

Useful for debugging: if a bug appears only in compiled mode, toggle stance to isolate.
