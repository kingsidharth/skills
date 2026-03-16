# Functions Reference

## Function Decorator

```python
@app.function(
    image=modal.Image.debian_slim(),
    gpu="A10G",
    cpu=2.0,
    memory=2048,  # MB
    timeout=300,  # seconds
    secrets=[modal.Secret.from_name("api-keys")],
    volumes={"/data": volume},
    enable_memory_snapshot=True,
    container_idle_timeout=60,
    allow_concurrent_inputs=10
)
def my_function():
    pass
```

## Container Lifecycle

```python
class Model:
    @modal.build()  # Runs when image is built
    def download_model(self):
        download_weights()
    
    @modal.enter()  # Runs when container starts
    def load_model(self):
        self.model = load_from_disk()
    
    @modal.method()  # Regular method
    def predict(self, x):
        return self.model(x)
    
    @modal.exit()  # Runs when container stops
    def cleanup(self):
        self.model.cleanup()
```

## Parallel Execution

```python
# Map (ordered results)
results = my_function.map([1, 2, 3, 4])

# Starmap (spread arguments)
results = my_function.starmap([(1, 2), (3, 4)])

# For_each (no results)
my_function.for_each([1, 2, 3])

# Spawn (async, get handle)
call = my_function.spawn(arg)
result = call.get()  # Wait for result

# Generator (stream results)
for result in my_function.map(large_list):
    process(result)  # Process as they complete
```

## Retries and Error Handling

```python
from modal import Retries

@app.function(
    retries=Retries(
        max_retries=3,
        backoff_coefficient=2.0,
        initial_delay=1.0
    )
)
def flaky_function():
    # Will retry on failure
    pass
```

## Batching

```python
@app.function(allow_concurrent_inputs=10)
def batch_process(items: list):
    # Process up to 10 requests concurrently in one container
    return [process(item) for item in items]
```

## Best Practices

1. Use @enter for expensive setup (model loading)
2. Use memory snapshots for GPU functions
3. Batch processing with .map() for parallelism
4. Set appropriate timeouts (default 5 min)
5. Use allow_concurrent_inputs for I/O bound workloads

