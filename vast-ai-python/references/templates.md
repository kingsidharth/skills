# Templates

A template is a reusable launch configuration wrapping `docker run`: image, env vars, ports, on-start commands, launch mode.

## Vast-provided base images

- `vastai/base-image` — minimal with Instance Portal, Caddy TLS, auth
- `vastai/pytorch` — PyTorch + CUDA + the above

Any public Docker Hub image works. Private registries supported via `image_login`.

## SDK template methods

```python
vast.search_templates(query="pytorch")         # find templates
vast.create_template(...)                       # create new
vast.update_template(id=ID, ...)               # modify
vast.delete_template(id=ID)                    # remove
```

## Template components

- **Image path:tag** — Docker image (e.g. `pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime`)
- **Launch mode** — `jupyter_direct` (recommended), `ssh_direct`, `ssh_proxy`, `args`
- **On-start script** — bash commands run after entrypoint initializes
- **Environment variables** — key-value pairs (avoid secrets in public templates)
- **Ports** — exposed container ports
- **Disk size default** — initial storage allocation suggestion

## Launch modes

| Mode | Access | Notes |
|---|---|---|
| Jupyter + SSH | Interactive notebooks + terminal | Best for dev/exploration |
| SSH only | Terminal access | Best for headless training |
| Entrypoint (args) | Container runs its own entrypoint | Best for services/inference servers |

## Creating via REST API

```
POST https://console.vast.ai/api/v0/templates/
{
  "name": "My PyTorch Template",
  "image": "pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime",
  "runtype": "jupyter_direct",
  "onstart": "pip install transformers",
  "env": {"HF_HOME": "/workspace/hf_cache"},
  "disk": 50
}
```

## Tips

- Always include a version tag on the Docker image (not just the path)
- Recommended templates cache on hosts → faster boot than custom images
- Use `onstart` for pip installs, model downloads, config — runs every boot
- For complex setup, build a custom Docker image instead of long onstart scripts
- Template cannot be changed after instance creation — need a new instance
