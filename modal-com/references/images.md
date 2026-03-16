# Container Images Reference

## Base Images

```python
# Debian
modal.Image.debian_slim(python_version="3.11")

# From Docker registry
modal.Image.from_registry("nvidia/cuda:12.4.0-devel-ubuntu22.04", add_python="3.11")

# Micromamba (conda alternative)
modal.Image.micromamba()
```

## Installing Dependencies

```python
image = modal.Image.debian_slim() \
    .pip_install("torch", "transformers") \
    .apt_install("ffmpeg", "git") \
    .run_commands("git clone https://...") \
    .env({"HF_HUB_ENABLE_HF_TRANSFER": "1"})
```

## Adding Local Code

```python
# Add directory
image = modal.Image.debian_slim() \
    .add_local_dir("./src", remote_path="/root/src")

# Add specific files
image = image.add_local_file("config.json", "/root/config.json")

# Add Python source (auto-discovered modules)
image = image.add_local_python_source()
```

## Build-Time Functions

```python
def download_model():
    from transformers import AutoModel
    AutoModel.from_pretrained("bert-base", cache_dir="/models")

image = modal.Image.debian_slim() \
    .pip_install("transformers") \
    .run_function(
        download_model,
        secrets=[modal.Secret.from_name("hf")],
        gpu="T4"  # Can use GPU during build
    )
```

## UV Package Manager (Faster)

```python
# Use uv instead of pip for faster installs
image = modal.Image.debian_slim() \
    .uv_pip_install("torch", "transformers")
```

## Caching

Images rebuild when definition changes. Layers are cached, so only changed layers rebuild.

**Tip**: Put stable dependencies first, volatile code last

