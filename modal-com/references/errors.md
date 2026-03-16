# Common Errors and Troubleshooting

## GPU Errors

**"CUDA not available"**:
- Check gpu= parameter is set
- Ensure torch/framework installed correctly
- Avoid CUDA calls in global scope with memory snapshots

**"CUDA out of memory"**:
- Reduce batch size
- Use float16: `torch_dtype=torch.float16`
- Use gradient checkpointing
- Choose larger GPU

## Volume Errors

**"Can't find file on Volume"**:
```python
volume.reload()  # Always reload before reading
```

**"Busy Volume errors"**:
- Don't modify same files from multiple containers
- Use separate files per container
- Coordinate with Dicts/Queues

## Image Build Errors

**"Package not found"**:
- Check package name spelling
- Use `pip_install` not `run_commands("pip install")`
- Check Python version compatibility

**"Permission denied"**:
- Don't write to read-only directories in image build
- Use /tmp or create writable dirs

## Timeout Errors

**"Function timeout"**:
```python
@app.function(timeout=3600)  # Increase timeout
```

## Debugging

```bash
# View logs
modal app logs my-app

# Interactive shell
modal shell

# Container shell for debugging
modal container exec <container-id> /bin/bash
```

## Memory Issues

**Container OOM**:
```python
@app.function(memory=16384)  # Increase memory (MB)
```

