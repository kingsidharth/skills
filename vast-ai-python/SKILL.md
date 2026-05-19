---
name: vast-ai-python
description: Rent GPU instances on Vast.ai marketplace using Python SDK. Search offers, create on-demand/interruptible instances, manage lifecycle, templates, volumes, data transfer, cloud sync. Use when mentioning vast.ai, VastAI, GPU rental, spot GPU, interruptible GPU instances, marketplace GPU.
---

# Vast.ai Python SDK

GPU marketplace — rent from individual hosts and datacenters. Python SDK wraps the CLI as typed methods via `from vastai import VastAI`.

## Install & auth

```
pip install vastai
```

```python
from vastai import VastAI
vast = VastAI(api_key="YOUR_KEY")  # or omit to read ~/.config/vastai/vast_api_key
```

Constructor params: `api_key`, `server_url`, `retry=3`, `raw=False`, `quiet=False`.

## Core workflow

1. `search_offers(query=..., order=..., type=..., limit=...)` → find GPU
2. `create_instance(id=OFFER_ID, image=..., disk=..., ssh=True, direct=True)` → rent it
3. Poll `show_instance(id=...)` until `actual_status == "running"`
4. `ssh_url(id=...)` or `copy(src=..., dst=...)` → connect / transfer data
5. `stop_instance(id=...)` (pause, storage charges continue) or `destroy_instance(id=...)` (delete all)

## When to load references

- **Choosing instance type (on-demand vs interruptible vs reserved)**: see [instance-types.md](references/instance-types.md)
- **Searching & filtering offers, query syntax, DLPerf/reliability scores**: see [searching-offers.md](references/searching-offers.md)
- **Creating instances via SDK or REST API, runtype options**: see [creating-instances.md](references/creating-instances.md)
- **Instance lifecycle — start/stop/destroy, status states, billing**: see [managing-instances.md](references/managing-instances.md)
- **Templates — creating, Docker images, launch modes, on-start scripts**: see [templates.md](references/templates.md)
- **Storage — container disk, volumes, data movement, cloud sync**: see [storage-and-data.md](references/storage-and-data.md)
- **Pricing model, cost components, billing**: see [pricing.md](references/pricing.md)
