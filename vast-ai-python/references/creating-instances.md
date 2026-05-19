# Creating Instances

Two-step process: search offers → accept an offer to create instance.

## SDK method

```python
result = vast.create_instance(
    id=OFFER_ID,
    image="pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime",
    disk=20,           # GB, fixed at creation, cannot resize
    onstart_cmd="echo hello && nvidia-smi",
    ssh=True,
    direct=True,
)
# Returns: {"success": true, "new_contract": 12345678}
instance_id = result["new_contract"]
```

Key params:
- `id`: offer ID from search_offers
- `image`: Docker image path:tag
- `disk`: storage in GB (default 10, cannot change later)
- `ssh`: enable SSH
- `direct`: use direct port mapping (lower latency than proxy)
- `onstart_cmd`: bash to run on boot
- `price`: bid $/hr for interruptible instances

## REST API

```
PUT https://console.vast.ai/api/v0/asks/{offer_id}/
Authorization: Bearer $VAST_API_KEY

{
  "image": "vllm/vllm-openai:latest",
  "disk": 50,
  "runtype": "ssh_direct",
  "env": {"MODEL_ID": "...", "-p 8000:8000": "1"},
  "onstart": "vllm serve $MODEL_ID --port 8000"
}
```

## From template

```
PUT /api/v0/asks/{offer_id}/
{"template_hash_id": "4e17788f74f075dd9aab7d0d4427968f"}
```

Override specific template values by including them alongside `template_hash_id`.

## Runtype options

| Runtype | Provisioned Ports | Description |
|---|---|---|
| `ssh_direct` | 22 | Direct SSH, lowest latency |
| `ssh_proxy` / `ssh` | none | SSH via Vast proxy |
| `jupyter_direct` | 8080 + 22 | **Recommended.** Jupyter + SSH, direct |
| `jupyter_proxy` / `jupyter` | none | Jupyter + SSH via proxy |
| `args` | none | Preserves image entrypoint; no SSH/Jupyter |

All Jupyter runtypes include SSH. Only `_direct` variants open ports on the instance.

## Interruptible instance creation

```python
result = vast.create_instance(
    id=OFFER_ID,
    image="pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime",
    disk=20,
    ssh=True,
    direct=True,
    price=0.25,  # bid in $/hr
)
```

## Polling for readiness

```python
import time

while True:
    info = vast.show_instance(id=instance_id)
    status = info.get("actual_status")
    if status == "running":
        break
    if status in ("exited", "unknown", "offline"):
        vast.destroy_instance(id=instance_id)
        raise RuntimeError(f"Instance failed: {status}")
    time.sleep(10)
```

Status progression: `loading` → `running`. Boot time typically 1–5 min. Handle `exited`/`unknown`/`offline` — they never reach `running` and accrue disk charges.

## Private Docker images

Pass registry credentials:
```json
{"image_login": "-u username -p access_token docker.io"}
```

## Attaching a volume at creation

```json
{"volume_info": {"volume_id": 12345, "mount_path": "/data"}}
```
