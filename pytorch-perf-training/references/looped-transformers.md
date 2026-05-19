# Looped / Recursive Transformers

Weight-shared transformer blocks that iterate a single block N times, achieving effective depth N with parameters for 1 block. Relevant for DiT variants exploring parameter efficiency.

## Core pattern: weight sharing

```python
class LoopedDiT(nn.Module):
    def __init__(self, dim, num_loops, num_unique_blocks=1):
        super().__init__()
        # Single block of unique layers
        self.blocks = nn.ModuleList([
            DiTBlock(dim) for _ in range(num_unique_blocks)
        ])
        self.num_loops = num_loops

    def forward(self, x, t_emb):
        for loop_idx in range(self.num_loops):
            for block in self.blocks:
                x = block(x, t_emb)
        return x
```

The same `block` parameters receive gradients from every loop iteration — gradients accumulate naturally through the shared weights.

## Sharing strategies

**Cycle**: blocks repeat in order `[0,1,2,0,1,2,...]`
```python
for i in range(total_depth):
    x = self.blocks[i % num_unique_blocks](x, t_emb)
```

**Sandwich**: mirror pattern `[0,1,2,2,1,0]`
```python
block_order = list(range(K)) + list(range(K-1, -1, -1))
for idx in block_order:
    x = self.blocks[idx](x, t_emb)
```

**Single block loop** (Universal Transformer style):
```python
for _ in range(self.num_loops):
    x = self.block(x, t_emb)
```

## Depth-wise LoRA (Relaxed Recursive Transformer)

Adds per-loop-iteration low-rank deltas to break the uniformity of shared weights while keeping the model compact:

```python
class LoopedBlockWithLoRA(nn.Module):
    def __init__(self, block, num_loops, lora_rank=16):
        super().__init__()
        self.block = block
        self.lora_downs = nn.ModuleList([
            nn.Linear(block.dim, lora_rank, bias=False)
            for _ in range(num_loops)
        ])
        self.lora_ups = nn.ModuleList([
            nn.Linear(lora_rank, block.dim, bias=False)
            for _ in range(num_loops)
        ])
        # Initialize LoRA near-zero
        for up in self.lora_ups:
            nn.init.zeros_(up.weight)

    def forward(self, x, t_emb, loop_idx):
        base_out = self.block(x, t_emb)
        delta = self.lora_ups[loop_idx](self.lora_downs[loop_idx](x))
        return base_out + delta
```

The rank controls capacity vs parameter trade-off. SVD initialization from the difference between pretrained layer weights is effective when converting a standard model to recursive.

## Routing in looped blocks

MoE-style routing selects which expert FFN to use at each loop depth:

```python
class RoutedLoopBlock(nn.Module):
    def __init__(self, dim, num_experts, num_loops):
        super().__init__()
        self.attn = MultiHeadAttention(dim)
        self.experts = nn.ModuleList([FFN(dim) for _ in range(num_experts)])
        self.router = nn.Linear(dim, num_experts)

    def forward(self, x, t_emb, loop_idx):
        x = x + self.attn(self.norm1(x))
        # Route to expert
        logits = self.router(x.mean(dim=1))  # [B, num_experts]
        weights = torch.softmax(logits, dim=-1)
        expert_outs = torch.stack([e(x) for e in self.experts], dim=1)
        x = x + (weights.unsqueeze(-1).unsqueeze(-1) * expert_outs).sum(dim=1)
        return x
```

## Gated recurrence (stabilising deep loops)

```python
class GatedLoopBlock(nn.Module):
    def __init__(self, block, gate_init_bias=2.0):
        super().__init__()
        self.block = block
        # High initial bias → gate starts near 1 → mostly pass-through
        self.gate = nn.Linear(block.dim, block.dim)
        nn.init.constant_(self.gate.bias, gate_init_bias)

    def forward(self, x, t_emb):
        out = self.block(x, t_emb)
        g = torch.sigmoid(self.gate(x))
        return g * out + (1 - g) * x  # interpolate between residual and output
```

Gating with high initial bias (~0.88 retention) stabilises training with many loops.

## torch.compile compatibility

Looped blocks compile well because:
- The loop is static (fixed `num_loops`) — no graph break
- Same block called repeatedly → kernels compiled once, reused
- Regional compilation on the shared block is optimal

```python
# Compile just the shared block — fastest compilation
model.block = torch.compile(model.block, mode="max-autotune")
```

**Pitfall**: if `loop_idx` is used to index into a Python list inside the compiled region, Dynamo unrolls the loop. For many loops, this bloats the graph. Fix:

```python
# BAD — Dynamo unrolls, graph size = O(num_loops)
for i in range(num_loops):
    x = self.lora_list[i](x)

# BETTER — use nn.ModuleList and index with a tensor
# Or compile the inner block only, keep the loop in eager
```

## Logging across loops

```python
class InstrumentedLoop(nn.Module):
    def forward(self, x, t_emb):
        norms = []
        for i in range(self.num_loops):
            x = self.block(x, t_emb)
            # Collect outside compiled region
            if not torch.compiler.is_compiling():
                norms.append(x.norm().item())
        return x, norms
```

`torch.compiler.is_compiling()` gates logging to avoid graph breaks during compilation.

## Per-loop LayerNorm

Each loop iteration should have its own LayerNorm to re-centre activations:

```python
class LoopedBlock(nn.Module):
    def __init__(self, dim, num_loops):
        super().__init__()
        self.block = TransformerBlock(dim)
        self.norms = nn.ModuleList([nn.LayerNorm(dim) for _ in range(num_loops)])

    def forward(self, x, t_emb):
        for i in range(len(self.norms)):
            x = self.norms[i](self.block(x, t_emb))
        return x
```

Shared weights with per-loop normalisation is more stable than fully shared normalisation, especially beyond 8 loops.

## Efficiency trade-offs

- **Parameter savings**: N-loop model with 1 unique block uses ~1/N parameters of a standard model
- **Compute cost**: same as an N-layer model (FLOPs are not reduced)
- **Wall-clock**: sequential loops cannot be parallelised in vanilla form (see Parallel Loop Transformer research for relaxations)
- **Gradient flow**: deeper effective depth means vanishing/exploding gradient risk — use gating + per-loop norm
- **Memory**: activation checkpointing across loops saves O(N) activation memory at O(N) recompute cost
