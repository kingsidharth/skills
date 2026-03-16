# Sandboxes Reference

## Creating Sandboxes

```python
import modal

app = modal.App()

# Create sandbox
sandbox = modal.Sandbox.create(
    "bash", "-c", "echo hello && sleep 10",
    app=app,
    image=modal.Image.debian_slim(),
    timeout=60
)

# Wait for completion
sandbox.wait()

# Get output
print(sandbox.stdout.read())
```

## Running Commands

```python
# Execute in running sandbox
process = sandbox.exec("python", "-c", "print('test')")
process.wait()
print(process.stdout.read())
```

## With Volumes and GPUs

```python
sandbox = modal.Sandbox.create(
    "python", "train.py",
    app=app,
    image=image,
    gpu="A10G",
    volumes={"/data": volume},
    timeout=3600
)
```

## Use Cases

- Run untrusted user code
- Execute LLM-generated code safely
- Git clone and test
- Isolated environments per request

