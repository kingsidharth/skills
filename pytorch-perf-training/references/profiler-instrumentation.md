# Profiler Instrumentation

Fine-grained annotation of training code for precise bottleneck attribution.

## record_function — custom labels in traces

```python
from torch.profiler import record_function

def train_step(batch):
    with record_function("data_transfer"):
        x, y = batch[0].cuda(non_blocking=True), batch[1].cuda(non_blocking=True)

    with record_function("forward"):
        with torch.amp.autocast("cuda", dtype=torch.bfloat16):
            logits = model(x)
            loss = criterion(logits, y)

    with record_function("backward"):
        loss.backward()

    with record_function("optimizer"):
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
```

These labels appear as named blocks in TensorBoard trace view and Chrome `chrome://tracing`. Nest them for sub-step granularity:

```python
with record_function("forward"):
    with record_function("forward/dit_blocks"):
        x = self.blocks(x, t_emb)
    with record_function("forward/final_layer"):
        x = self.final_layer(x)
```

## NVTX ranges (for Nsight Systems)

```python
# Manual NVTX — visible in Nsight Systems
torch.cuda.nvtx.range_push("forward_pass")
output = model(x)
torch.cuda.nvtx.range_pop()

# Context manager form
with torch.cuda.nvtx.range("backward_pass"):
    loss.backward()
```

`record_function` also emits NVTX ranges automatically when profiling with CUDA activity.

## Profiler schedule for long training

```python
from torch.profiler import profile, schedule, tensorboard_trace_handler

prof = profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    schedule=schedule(
        wait=5,      # skip first 5 steps (data loading warmup)
        warmup=3,    # profiler warmup (CUDA context, caches)
        active=5,    # record these 5 steps
        repeat=2,    # repeat the cycle twice
    ),
    on_trace_ready=tensorboard_trace_handler("./tb_logs"),
    record_shapes=True,
    profile_memory=True,
    with_stack=True,
    with_flops=True,    # estimate FLOPs for matmul / conv2d
)

prof.start()
for step, batch in enumerate(loader):
    train_step(batch)
    prof.step()
    if step >= 25:
        break
prof.stop()
```

`with_flops=True` estimates FLOPs for matmul and conv2d — useful for computing arithmetic intensity (FLOPs / bytes) to determine if a kernel is compute-bound or memory-bound.

## Profiler output formats

```python
# Table — quick terminal summary
print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=20))

# Grouped by input shape — reveals padding/alignment issues
print(prof.key_averages(group_by_input_shape=True).table(
    sort_by="cuda_time_total", row_limit=20
))

# Grouped by stack — traces back to source code
print(prof.key_averages(group_by_stack_n=5).table(
    sort_by="self_cuda_time_total", row_limit=20
))

# Chrome trace JSON
prof.export_chrome_trace("trace.json")
# View: chrome://tracing or ui.perfetto.dev

# Stacks for flamegraph
prof.export_stacks("profiler_stacks.txt", metric="self_cuda_time_total")
# Visualise with: flamegraph.pl < profiler_stacks.txt > perf.svg
```

## Key columns to look at

| Column | Meaning |
|--------|---------|
| `self_cpu_time_total` | Time in this op excluding children (CPU side) |
| `self_cuda_time_total` | Time in this op excluding children (GPU side) |
| `cuda_time_total` | Total GPU time including children |
| `cpu_memory_usage` | CPU memory delta |
| `self_cuda_memory_usage` | GPU memory allocated by this op |
| `# of Calls` | Frequency — high-frequency small ops are fusion candidates |
| `Input Shapes` | Reveals alignment issues (non-multiples of 8) |

## Profiling torch.compile'd code

When profiled, compiled code shows:
- `torch._dynamo` tracing events on the first call
- `Compiled region` blocks on subsequent calls
- Individual Triton/cuBLAS kernels inside compiled regions

To see what Inductor generated:
```bash
TORCH_COMPILE_DEBUG=1 python train.py
# Outputs to /tmp/torchinductor_<user>/<hash>.debug/
# Contains: fx_graph_readable.py, output_code.py (generated Triton)
```

## Selective profiling

Profile only a specific component (avoids noise from data loading):

```python
# Don't profile data loading
for batch in loader:
    x, y = batch
    with profile(activities=[ProfilerActivity.CUDA]) as prof:
        train_step(x, y)
    print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=10))
    break  # just one step
```

## Memory timeline

```python
# Granular memory allocation timeline
with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    profile_memory=True,
    record_shapes=True,
) as prof:
    train_step(batch)

prof.export_memory_timeline("memory_timeline.html")
# Opens in browser — shows allocation/deallocation over time
```
