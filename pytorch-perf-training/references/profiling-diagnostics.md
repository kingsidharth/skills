# Profiling & Diagnostics

Profile first, optimise second. Every change should be validated by a before/after trace.

## PyTorch Profiler

```python
from torch.profiler import profile, ProfilerActivity, schedule

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    schedule=schedule(wait=1, warmup=3, active=5, repeat=1),
    on_trace_ready=torch.profiler.tensorboard_trace_handler("./log"),
    record_shapes=True,
    profile_memory=True,
    with_stack=True,
) as prof:
    for step, batch in enumerate(loader):
        train_step(batch)
        prof.step()
        if step >= 9:
            break
```

Key things to look for in the trace:

- **GPU idle gaps** — CPU is bottleneck (data loading, Python overhead, sync points)
- **Many small kernels** — candidate for CUDA graphs or torch.compile fusion
- **No Tensor Core kernels** (look for `sm80_xmma` or `ampere_` prefixes) — AMP not engaged or shapes not aligned
- **cudaMemcpy / cudaStreamSynchronize** — unwanted CPU-GPU sync

## TORCH_LOGS for torch.compile

```bash
# Graph breaks
TORCH_LOGS="+dynamo" python train.py 2>&1 | grep "graph break"

# Full compilation debug dump
TORCH_COMPILE_DEBUG=1 python train.py

# Parse logs with tlparse
pip install tlparse
TORCH_LOGS="+dynamo,+inductor" python train.py 2>&1 | tlparse
```

## Checking graph breaks

```python
# Fail hard on any graph break — use during development
model = torch.compile(model, fullgraph=True)

# Softer: count breaks
import torch._dynamo as dynamo
dynamo.config.log_level = logging.DEBUG
```

Common graph-break causes in DiT models:
- Python control flow dependent on tensor values (`if x.sum() > 0`)
- In-place ops on views in some patterns
- Custom Python logging/print inside forward
- List comprehensions (pre-3.12)
- Data-dependent shapes

Fix: `torch._dynamo.config.ignore_logging_functions.add(your_debug_fn)` or refactor control flow to be tensor-independent.

## Compile profiler

```python
from torch._dynamo.utils import CompileProfiler

with CompileProfiler() as prof:
    model = torch.compile(model, backend=prof)
    train_step(batch)
print(prof.report())
```

Reports recompilation causes, cache hits/misses, graph break locations.

## NVIDIA Nsight Systems

```bash
nsys profile -o trace --trace=cuda,nvtx \
    python train.py --steps=20

# View in Nsight Systems GUI
nsys-ui trace.nsys-rep
```

Look for:
- Kernel occupancy < 50% — block size / register pressure issue
- Memory throughput near HBM bandwidth limit — compute-bound optimisations won't help
- NCCL calls (if single-GPU, these shouldn't exist)

## Measuring wall-clock correctly

```python
# Use CUDA events, not time.time()
start = torch.cuda.Event(enable_timing=True)
end = torch.cuda.Event(enable_timing=True)

torch.cuda.synchronize()
start.record()
for _ in range(N):
    train_step(batch)
end.record()
torch.cuda.synchronize()

ms_per_step = start.elapsed_time(end) / N
```

## Memory diagnostics

```python
# Snapshot for memory visualiser
torch.cuda.memory._record_memory_history()
# ... run training ...
torch.cuda.memory._dump_snapshot("mem_snapshot.pickle")
torch.cuda.memory._record_memory_history(enabled=None)
# View at pytorch.org/memory_viz
```

```python
# Quick stats
print(torch.cuda.memory_summary())
print(f"Allocated: {torch.cuda.memory_allocated() / 1e9:.2f} GB")
print(f"Reserved:  {torch.cuda.memory_reserved() / 1e9:.2f} GB")
```

## Diagnostic decision tree

```
Slow training step
├── GPU utilisation < 80%?
│   ├── Yes → CPU-bound. Check data loading, Python overhead, sync points.
│   └── No  → GPU-bound. Continue ↓
├── Many small kernels visible in trace?
│   ├── Yes → Apply torch.compile or CUDA graphs
│   └── No  → Continue ↓
├── Kernels not using Tensor Cores?
│   ├── Yes → Enable AMP, check dtype alignment (multiples of 8)
│   └── No  → Continue ↓
├── Memory-bound kernels (low arithmetic intensity)?
│   ├── Yes → Fusion via torch.compile, channels_last, activation checkpointing
│   └── No  → Compute-bound. Custom Triton kernels, algorithmic changes.
└── Data loading visible in trace gaps?
    └── Yes → Increase num_workers, use pin_memory, persistent_workers
```
