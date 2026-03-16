# Performance Optimization Reference

## Cold Start Optimization

### 1. Enable Memory Snapshots (CRITICAL)

```python
@app.function(
    enable_memory_snapshot=True,  # CPU snapshot
    experimental_options={"enable_gpu_snapshot": True}  # GPU snapshot
)
def optimized():
    pass
```

**Impact**: 5-10x faster cold starts

### 2. Concurrent Model Loading

```python
from concurrent.futures import ThreadPoolExecutor

def load_models():
    with ThreadPoolExecutor() as executor:
        futures = [
            executor.submit(load, "model-a"),
            executor.submit(load, "model-b")
        ]
        return [f.result() for f in futures]
```

### 3. Keep Containers Warm

```python
@app.function(
    container_idle_timeout=300,  # Keep warm 5 min
    allow_concurrent_inputs=10   # Reuse containers
)
def warm_function():
    pass
```

### 4. Use Volumes for Model Weights

Don't bake models into images - use Volumes to decouple from code changes.

## Scaling Configuration

```python
@app.function(
    allow_concurrent_inputs=10,      # Handle 10 requests per container
    min_containers=2,                # Always keep 2 warm
    max_concurrent_inputs_per_container=5  # Max concurrent per container
)
```

## Batch Processing

```python
# Good: Parallel execution
results = my_function.map(inputs, order_outputs=False)  # Return as they complete

# Bad: Sequential
for input in inputs:
    result = my_function.remote(input)
```

## Memory Snapshot Best Practices

1. Load models in @enter (before snapshot)
2. Avoid CUDA calls before snapshot
3. Use GPU snapshots for torch.compile
4. First deployment creates snapshot (slower), subsequent fast

