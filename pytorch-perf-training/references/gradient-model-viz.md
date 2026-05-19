# Gradient Monitoring & Model Visualization

## Gradient health check (after loss.backward())

```python
def log_gradient_stats(model, step, writer=None):
    """Call after loss.backward(), before optimizer.step()."""
    for name, param in model.named_parameters():
        if param.grad is None:
            continue
        grad = param.grad
        norm = grad.norm().item()
        # Detect problems
        if torch.isnan(grad).any():
            print(f"[step {step}] NaN gradient: {name}")
        if torch.isinf(grad).any():
            print(f"[step {step}] Inf gradient: {name}")
        if norm < 1e-7:
            print(f"[step {step}] Vanishing gradient: {name} (norm={norm:.2e})")
        if norm > 1e3:
            print(f"[step {step}] Exploding gradient: {name} (norm={norm:.2e})")
        # TensorBoard histograms
        if writer:
            writer.add_histogram(f"grad/{name}", grad, step)
            writer.add_scalar(f"grad_norm/{name}", norm, step)
```

## Total gradient norm (for clipping decisions)

```python
total_norm = torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
# total_norm is the pre-clip norm — log it to detect if clipping activates often
print(f"Grad norm: {total_norm:.4f}")
```

If gradient norm is frequently clipped, training is unstable. Track the ratio of clipped vs unclipped steps.

## Hooks for per-layer gradient inspection

### Tensor hooks (on parameter gradients)

```python
hooks = []
for name, param in model.named_parameters():
    def make_hook(n):
        def hook(grad):
            print(f"{n}: grad norm = {grad.norm().item():.4e}")
            return grad  # return None to not modify
        return hook
    h = param.register_hook(make_hook(name))
    hooks.append(h)

# Run forward + backward
loss.backward()

# Clean up
for h in hooks:
    h.remove()
```

### Module hooks (activation + gradient flow)

```python
activation_stats = {}
gradient_stats = {}

def fwd_hook(name):
    def hook(module, input, output):
        if isinstance(output, torch.Tensor):
            activation_stats[name] = {
                "mean": output.mean().item(),
                "std": output.std().item(),
                "max": output.abs().max().item(),
                "has_nan": torch.isnan(output).any().item(),
            }
    return hook

def bwd_hook(name):
    def hook(module, grad_input, grad_output):
        if grad_output[0] is not None:
            gradient_stats[name] = {
                "norm": grad_output[0].norm().item(),
                "max": grad_output[0].abs().max().item(),
            }
    return hook

# Register on all DiT blocks
for i, block in enumerate(model.blocks):
    block.register_forward_hook(fwd_hook(f"block_{i}"))
    block.register_full_backward_hook(bwd_hook(f"block_{i}"))
```

`register_full_backward_hook` is preferred over the deprecated `register_backward_hook` — it correctly handles multiple outputs and in-place operations.

## Hooks + torch.compile compatibility

Hooks registered **before** `torch.compile` generally work. Module forward hooks are compatible. Backward hooks may cause graph breaks in some cases.

```python
# Safe: register hooks, then compile
for block in model.blocks:
    block.register_forward_hook(my_hook)
model = torch.compile(model)

# Gate logging to avoid sync during compiled execution
def safe_fwd_hook(module, input, output):
    if not torch.compiler.is_compiling():
        # safe to call .item() etc here
        print(output.mean().item())
```

## Model architecture summary

### torchinfo (parameter counts, shapes, memory)

```python
from torchinfo import summary

summary(model, input_size=(B, C, H, W),
        col_names=["input_size", "output_size", "num_params", "mult_adds"],
        depth=3, verbose=2)
```

Shows per-layer parameter count, output shapes, MACs (multiply-accumulate). Useful for identifying unexpectedly large layers.

### Computational graph (torchviz)

```python
from torchviz import make_dot

x = torch.randn(1, C, H, W, device="cuda")
y = model(x)
dot = make_dot(y, params=dict(model.named_parameters()))
dot.render("model_graph", format="pdf")
```

Renders the autograd graph — shows which operations connect to which parameters. Useful for verifying skip connections, shared weights, and gradient paths.

### torchview (more detailed than torchviz)

```python
from torchview import draw_graph

graph = draw_graph(model, input_size=(1, C, H, W), depth=3,
                   expand_nested=True, save_graph=True,
                   filename="model_view")
```

Uses `__torch_function__` to trace through arbitrary functions, not just `nn.Module`. Shows actual tensor operations.

### Netron (ONNX viewer)

```python
torch.onnx.export(model, dummy_input, "model.onnx", opset_version=17)
# Open model.onnx at netron.app or install netron locally
```

Interactive graph viewer with shape inspection.

## TensorBoard integration

```python
from torch.utils.tensorboard import SummaryWriter

writer = SummaryWriter("runs/dit_training")

# Log scalars
writer.add_scalar("train/loss", loss.item(), step)
writer.add_scalar("train/lr", scheduler.get_last_lr()[0], step)

# Log gradient norms per layer
for name, param in model.named_parameters():
    if param.grad is not None:
        writer.add_scalar(f"grad_norm/{name}", param.grad.norm().item(), step)

# Log weight histograms (expensive — do every N steps)
if step % 500 == 0:
    for name, param in model.named_parameters():
        writer.add_histogram(f"weights/{name}", param.data, step)
        if param.grad is not None:
            writer.add_histogram(f"grads/{name}", param.grad, step)

# Log model graph (once)
writer.add_graph(model, dummy_input)
writer.close()
```

Launch: `tensorboard --logdir runs/`

## Anomaly detection (debugging NaN/Inf)

```python
# Enables autograd anomaly detection — slow but catches NaN source
torch.autograd.set_detect_anomaly(True)

# After finding the issue, DISABLE for training:
torch.autograd.set_detect_anomaly(False)
```

When anomaly detection is on, the first backward pass producing NaN will print the forward operation that caused it, with a full stack trace.

**Performance cost is high** — only use for debugging, never in production training.

## Gradient flow plot (matplotlib)

```python
def plot_grad_flow(named_parameters):
    """Call after loss.backward(). Visualises gradient magnitudes per layer."""
    import matplotlib.pyplot as plt
    ave_grads, max_grads, layers = [], [], []
    for n, p in named_parameters:
        if p.requires_grad and p.grad is not None and "bias" not in n:
            layers.append(n)
            ave_grads.append(p.grad.abs().mean().item())
            max_grads.append(p.grad.abs().max().item())
    plt.figure(figsize=(max(10, len(layers) * 0.3), 6))
    plt.bar(range(len(max_grads)), max_grads, alpha=0.3, color="c", label="max")
    plt.bar(range(len(ave_grads)), ave_grads, alpha=0.5, color="b", label="mean")
    plt.xticks(range(len(layers)), layers, rotation=90, fontsize=6)
    plt.xlabel("Layers")
    plt.ylabel("Gradient magnitude")
    plt.yscale("log")
    plt.legend()
    plt.tight_layout()
    plt.savefig("grad_flow.png", dpi=150)
    plt.close()
```
