# Benchmarking Recipes

Reproducible timing patterns for comparing optimisation changes.

## Standard training step benchmark

```python
import torch
import time

def benchmark_train_step(model, optimizer, input_data, target, 
                         warmup=10, iterations=100):
    """Returns median ms/step using CUDA events."""
    model.train()
    start_evt = torch.cuda.Event(enable_timing=True)
    end_evt = torch.cuda.Event(enable_timing=True)

    # Warmup (triggers compilation, CUDA graph recording, cuDNN autotuning)
    for _ in range(warmup):
        with torch.amp.autocast("cuda", dtype=torch.bfloat16):
            out = model(input_data)
            loss = torch.nn.functional.mse_loss(out, target)
        loss.backward()
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)

    torch.cuda.synchronize()

    # Timed iterations
    times = []
    for _ in range(iterations):
        start_evt.record()
        with torch.amp.autocast("cuda", dtype=torch.bfloat16):
            out = model(input_data)
            loss = torch.nn.functional.mse_loss(out, target)
        loss.backward()
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
        end_evt.record()
        torch.cuda.synchronize()
        times.append(start_evt.elapsed_time(end_evt))

    times.sort()
    median = times[len(times) // 2]
    p10 = times[int(len(times) * 0.1)]
    p90 = times[int(len(times) * 0.9)]
    print(f"Median: {median:.2f} ms | P10: {p10:.2f} ms | P90: {p90:.2f} ms")
    return median
```

Always use CUDA events, never `time.time()`. CPU timers don't account for GPU async execution.

## torch.utils.benchmark (micro-benchmarks)

```python
from torch.utils.benchmark import Timer

t = Timer(
    stmt="model(x)",
    setup="import torch; model = model.cuda(); x = torch.randn(B, C, H, W, device='cuda')",
    globals={"model": model, "B": 32, "C": 4, "H": 32, "W": 32},
)
print(t.timeit(100))
# Reports: mean, median, IQR, with proper CUDA sync
```

For comparing two implementations:

```python
from torch.utils.benchmark import Compare

results = []
for name, fn in [("eager", model_eager), ("compiled", model_compiled)]:
    t = Timer(stmt="fn(x)", globals={"fn": fn, "x": x})
    results.append(t.blocked_autorange(min_run_time=2))
    results[-1].title = name

compare = Compare(results)
compare.print()
```

## A/B comparison template

```python
def ab_compare(model, input_data, target, configs):
    """
    configs: list of dicts with 'name' and 'setup' callable.
    setup(model) should return the modified model.
    """
    results = {}
    for cfg in configs:
        m = cfg["setup"](copy.deepcopy(model).cuda())
        opt = torch.optim.AdamW(m.parameters(), lr=1e-4)
        ms = benchmark_train_step(m, opt, input_data, target)
        results[cfg["name"]] = ms
        del m, opt
        torch.cuda.empty_cache()

    baseline = list(results.values())[0]
    for name, ms in results.items():
        speedup = baseline / ms
        print(f"{name}: {ms:.2f} ms ({speedup:.2f}x vs baseline)")

# Usage
ab_compare(model, x, y, [
    {"name": "eager",    "setup": lambda m: m},
    {"name": "compiled", "setup": lambda m: torch.compile(m, mode="max-autotune")},
    {"name": "reduce-overhead", "setup": lambda m: torch.compile(m, mode="reduce-overhead")},
])
```

## Throughput metrics

```python
batch_size = 32
ms_per_step = benchmark_train_step(model, opt, x, y)

samples_per_sec = batch_size / (ms_per_step / 1000)
print(f"Throughput: {samples_per_sec:.1f} samples/sec")

# For image generation (DiT)
images_per_sec = batch_size / (ms_per_step / 1000)
time_per_image = 1000 / images_per_sec
print(f"{images_per_sec:.1f} img/s | {time_per_image:.1f} ms/img")
```

## Memory benchmark

```python
def measure_peak_memory(fn, *args, **kwargs):
    """Measure peak GPU memory for a function call."""
    torch.cuda.reset_peak_memory_stats()
    torch.cuda.synchronize()

    fn(*args, **kwargs)
    torch.cuda.synchronize()

    peak_mb = torch.cuda.max_memory_allocated() / 1e6
    reserved_mb = torch.cuda.max_memory_reserved() / 1e6
    print(f"Peak allocated: {peak_mb:.1f} MB | Reserved: {reserved_mb:.1f} MB")
    return peak_mb
```

## Compilation time benchmark

```python
import time

model = MyDiT().cuda()
x = torch.randn(B, C, H, W, device="cuda")

start = time.time()
compiled = torch.compile(model, mode="max-autotune")
# First call triggers actual compilation
with torch.no_grad():
    compiled(x)
torch.cuda.synchronize()
compile_time = time.time() - start
print(f"Compilation time: {compile_time:.1f}s")
```

Track this — long compile times suggest too many graph variants or dynamic shapes. Regional compilation can reduce this 5-10x for repeated blocks.

## Reproducibility checklist

```python
# Fixed seeds for reproducible benchmarks
torch.manual_seed(42)
torch.cuda.manual_seed_all(42)
torch.backends.cudnn.deterministic = True   # slight perf cost
torch.backends.cudnn.benchmark = False       # disable for reproducibility
                                              # enable for speed
```

For benchmarking *speed*, disable deterministic mode and enable benchmark. For benchmarking *correctness*, do the opposite.

## GPU state before benchmarking

```python
# Ensure clean GPU state
torch.cuda.empty_cache()
torch.cuda.reset_peak_memory_stats()
torch.cuda.synchronize()

# Check no other processes are using the GPU
# nvidia-smi should show 0 MiB used by other processes
```

## Reporting template

```
Configuration:
  GPU: NVIDIA A100 80GB
  PyTorch: 2.5.0+cu124
  CUDA: 12.4
  Model: DiT-XL/2 (675M params)
  Batch size: 32
  Resolution: 256×256
  Precision: bf16

Results (median of 100 steps, 10 warmup):
  Eager:           142.3 ms/step  (224.8 img/s)
  + AMP bf16:       89.1 ms/step  (359.2 img/s) — 1.60x
  + torch.compile:  61.4 ms/step  (521.2 img/s) — 2.32x
  + CUDA graphs:    54.8 ms/step  (583.9 img/s) — 2.60x
  + compiled optim: 51.2 ms/step  (625.0 img/s) — 2.78x

Peak memory:
  Eager:     42.1 GB
  Compiled:  38.7 GB (compiled fuses intermediates)
```
