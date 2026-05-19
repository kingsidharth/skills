# Compiled Optimizer

Compiling the optimizer step fuses it with the backward pass, reducing kernel launches and memory traffic.

## Basic usage

```python
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)
# Compile the step function
optimizer.step = torch.compile(optimizer.step)
```

Or wrap the entire training step:

```python
@torch.compile(mode="max-autotune")
def train_step(model, optimizer, input, target):
    with torch.amp.autocast("cuda", dtype=torch.bfloat16):
        output = model(input)
        loss = criterion(output, target)
    loss.backward()
    optimizer.step()
    optimizer.zero_grad(set_to_none=True)
    return loss
```

## foreach / fused optimizers

PyTorch optimizers support multi-tensor variants:

```python
# foreach=True — processes all params in one kernel launch per op
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4, foreach=True)

# fused=True — single kernel for entire update (CUDA-only, not all optimizers)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4, fused=True)
```

Priority: `fused` > `foreach` > default. `fused` does the entire Adam update (moment estimation + param update) in one kernel.

When compiled: torch.compile can often match or exceed `fused` by generating its own fused kernel.

## Compiled optimizer + LR scheduler

```python
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=1000)

# Compile both
@torch.compile
def step_and_schedule(optimizer, scheduler):
    optimizer.step()
    scheduler.step()
```

Note: some schedulers call `.item()` or use Python state — may cause graph breaks. Test with `fullgraph=True`.

## foreach_map (explicit horizontal fusion)

For custom parameter updates:

```python
from torch._foreach_utils import foreach_map

def custom_update(param, grad, momentum, lr):
    momentum.mul_(0.9).add_(grad)
    param.add_(momentum, alpha=-lr)

# Fuses across all parameters
foreach_map(custom_update, list(model.parameters()),
            [p.grad for p in model.parameters()],
            momentum_buffers, lr=1e-4)
```

## Interaction with CUDA graphs

Compiled optimizers work with CUDA graphs, but optimizer state must be pre-initialized:

```python
# Run one dummy step to initialize Adam's exp_avg, exp_avg_sq
with torch.no_grad():
    dummy_loss = model(dummy_input).sum()
    dummy_loss.backward()
    optimizer.step()
    optimizer.zero_grad(set_to_none=True)

# Now safe to capture CUDA graph
```
