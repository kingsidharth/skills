# CUDA Graphs

CUDA graphs record a sequence of GPU operations and replay them as a single CPU launch. Eliminates per-kernel CPU launch overhead. Most impactful when the model has many small kernels or when CPU is the bottleneck.

## Automatic via torch.compile

```python
# Recommended: let torch.compile handle it
model = torch.compile(model, mode="reduce-overhead")
# Equivalent:
model = torch.compile(model, options={"triton.cudagraphs": True})
```

This uses **CUDAGraph Trees** internally — handles graph breaks by creating tree-shaped paths, shares memory pools between forward and backward.

## Requirements for CUDA graph capture

- **Static shapes**: all tensor shapes must be identical across iterations
- **No CPU synchronization** inside the graph (no `.item()`, `.cpu()`, `print(tensor)`)
- **No host-side control flow** dependent on tensor values
- **Static memory addresses**: tensors must live at the same addresses (use in-place copy for inputs)
- **Pre-initialized optimizer state**: run a dummy step before capture so momentum buffers exist

## Manual CUDA graphs (when torch.compile isn't enough)

```python
# Pre-allocate static tensors
static_input = torch.randn(B, C, H, W, device="cuda")
static_target = torch.randint(0, num_classes, (B,), device="cuda")

# Warm up
for _ in range(3):
    output = model(static_input)
    loss = criterion(output, static_target)
    loss.backward()
    optimizer.step()
    optimizer.zero_grad(set_to_none=True)

# Capture
g = torch.cuda.CUDAGraph()
with torch.cuda.graph(g):
    output = model(static_input)
    loss = criterion(output, static_target)
    loss.backward()
    optimizer.step()
    optimizer.zero_grad(set_to_none=True)

# Training loop — replay
for batch_input, batch_target in loader:
    static_input.copy_(batch_input)
    static_target.copy_(batch_target)
    g.replay()
    # loss is valid here — read from static tensor
```

## CUDAGraph Trees (torch.compile internals)

When `torch.compile(mode="reduce-overhead")` encounters graph breaks, it builds a **tree** of CUDA graph segments:
- Each path through the model becomes a branch
- Forward and backward share the same memory pool
- Generation tracking prevents premature memory reuse

Manual iteration boundary marking (if heuristics fail):
```python
torch.compiler.cudagraph_mark_step_begin()
```

## DiT-specific considerations

DiT models are excellent CUDA graph candidates because:
- Fixed spatial resolution per noise level (static shapes)
- Repeating identical blocks (high kernel count, small kernels)
- No data-dependent control flow in the core forward pass

Watch out for:
- **Classifier-free guidance**: doubles the batch (still static, just 2x)
- **Timestep conditioning**: ensure the conditioning path is tensor-only, no Python branching on timestep values
- **Variable sequence length** in text conditioning: pad to max length

## Gotchas

- **RNG in CUDA graphs**: PyTorch handles dropout correctly, but custom RNG needs `CUDAGraph.register_generator_state()`
- **Gradient accumulation**: graph captures one backward; for accumulation over N steps, capture N steps or use `no_sync()` patterns
- **Memory overhead**: captured graph pins memory addresses. For large models, this can increase peak memory. Monitor with `torch.cuda.memory_reserved()`
- **25% of CUDA graphs can hurt performance** (PyGraph research) — mostly from parameter-copy overhead. Profile to confirm speedup.
