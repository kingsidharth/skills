# RunPod REST API — Pod Management Reference

Base URL: `https://rest.runpod.io/v1`
Auth: `Authorization: Bearer <RUNPOD_API_KEY>`

## Helper

```python
import requests, os

API_KEY = os.environ["RUNPOD_API_KEY"]
BASE = "https://rest.runpod.io/v1"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}

def rp_post(path, json=None):
    return requests.post(f"{BASE}{path}", headers=HEADERS, json=json).json()

def rp_get(path):
    return requests.get(f"{BASE}{path}", headers=HEADERS).json()

def rp_delete(path):
    return requests.delete(f"{BASE}{path}", headers=HEADERS).json()
```

---

## POST /pods — Create Pod

### Key Parameters

| Parameter | Type | Notes |
|---|---|---|
| `name` | string | Pod display name |
| `imageName` | string | Docker image (e.g. `runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04`) |
| `gpuTypeIds` | string[] | GPU types in priority order. E.g. `["NVIDIA GeForce RTX 4090", "NVIDIA RTX A5000"]` |
| `gpuCount` | int | Number of GPUs (default 1) |
| `containerDiskInGb` | int | Temp disk (cleared on stop). Default 20 |
| `volumeInGb` | int | Persistent volume disk (survives restarts, deleted on terminate) |
| `networkVolumeId` | string | Attach pre-existing network volume (replaces volumeInGb) |
| `volumeMountPath` | string | Mount path, default `/workspace` |
| `cloudType` | `SECURE` \| `COMMUNITY` | Default `SECURE` |
| `interruptible` | bool | `true` = spot instance. Default `false` |
| `templateId` | string | Use pre-configured template instead of inline config |
| `ports` | string[] | E.g. `["8888/http", "22/tcp"]` |
| `env` | object | `{"KEY": "value"}` — env vars inside container |
| `supportPublicIp` | bool | Request public IP for TCP access |
| `globalNetworking` | bool | Enable pod-to-pod private network |
| `dockerStartCmd` | string[] | Override container CMD |
| `dockerEntrypoint` | string[] | Override container ENTRYPOINT |
| `dataCenterIds` | string[] | Pin to specific DCs: `["US-TX-3", "EU-NL-1"]` |
| `allowedCudaVersions` | string[] | Filter by CUDA: `["12.4", "12.3"]` |
| `minRAMPerGPU` | int | Minimum RAM per GPU in GB |
| `minVCPUPerGPU` | int | Minimum vCPUs per GPU |
| `containerRegistryAuthId` | string | For private Docker registries |
| `gpuTypePriority` | `availability` \| `price` | How to pick among `gpuTypeIds` |
| `dataCenterPriority` | `availability` \| `price` | How to pick among `dataCenterIds` |
| `locked` | bool | Prevent accidental termination |

### Response (201)

Key fields in response:
- `id` — pod ID (use for all lifecycle calls)
- `desiredStatus` — `RUNNING`, `EXITED`
- `costPerHr` — hourly cost string
- `machine.dataCenterId` — assigned DC
- `publicIp` — if `supportPublicIp` was true
- `portMappings` — e.g. `{"22": 10341}` (external port for TCP)

---

## Pod Lifecycle Endpoints

| Action | Method | Path |
|---|---|---|
| List pods | GET | `/pods` |
| Get pod | GET | `/pods/{podId}` |
| Stop pod | POST | `/pods/{podId}/stop` |
| Start pod | POST | `/pods/{podId}/start` |
| Restart pod | POST | `/pods/{podId}/restart` |
| Reset pod | POST | `/pods/{podId}/reset` |
| Update pod | PATCH | `/pods/{podId}` |
| Delete pod | DELETE | `/pods/{podId}` |

### Stop vs Terminate

- **Stop**: Releases GPU. Volume disk at `/workspace` preserved. Charged for volume storage ($0.20/GB/mo). Network volumes unaffected.
- **Terminate** (DELETE): Destroys pod + volume disk. Network volumes survive independently.

Pods with network volumes attached cannot be stopped — only terminated. Data persists in the network volume.

---

## Network Volumes

| Action | Method | Path |
|---|---|---|
| Create | POST | `/networkvolumes` |
| List | GET | `/networkvolumes` |
| Get | GET | `/networkvolumes/{id}` |
| Update | PATCH | `/networkvolumes/{id}` |
| Delete | DELETE | `/networkvolumes/{id}` |

### Create Network Volume

```python
rp_post("/networkvolumes", {
    "name": "my-data",
    "size": 100,          # GB
    "dataCenterId": "US-TX-3"
})
```

Pricing: $0.07/GB/mo (< 1TB), $0.05/GB/mo (> 1TB). Can only increase size, not decrease.

---

## Templates

| Action | Method | Path |
|---|---|---|
| Create | POST | `/templates` |
| List | GET | `/templates` |
| Get | GET | `/templates/{id}` |
| Update | PATCH | `/templates/{id}` |
| Delete | DELETE | `/templates/{id}` |

### Create Template

```python
rp_post("/templates", {
    "name": "my-training-env",
    "imageName": "my-registry/my-image:v1",
    "containerDiskInGb": 50,
    "volumeInGb": 100,
    "volumeMountPath": "/workspace",
    "ports": ["8888/http", "22/tcp"],
    "env": {"MODEL_NAME": "my-model"},
    "dockerStartCmd": ["python", "main.py"],
    "isPublic": False,
})
```

Use `templateId` in pod creation to deploy from template.

---

## Secrets

Secrets are managed via console only (not REST API). Reference in templates/env vars:

```
{{ RUNPOD_SECRET_my_api_key }}
```

---

## Auto-provided Environment Variables

Available inside every pod:

| Variable | Description |
|---|---|
| `RUNPOD_POD_ID` | Unique pod ID |
| `RUNPOD_DC_ID` | Data center ID |
| `RUNPOD_GPU_COUNT` | Number of GPUs |
| `RUNPOD_PUBLIC_IP` | Public IP (if available) |
| `RUNPOD_TCP_PORT_22` | External SSH port |
| `RUNPOD_API_KEY` | Pod-scoped API key |
| `PUBLIC_KEY` | SSH public keys from account settings |

---

## GPU Type ID Strings

Use exact strings. Common ones:

- `NVIDIA GeForce RTX 4090`
- `NVIDIA GeForce RTX 3090`
- `NVIDIA RTX A5000`
- `NVIDIA RTX A6000`
- `NVIDIA A40`
- `NVIDIA A100-SXM4-80GB`
- `NVIDIA A100 80GB PCIe`
- `NVIDIA H100 80GB HBM3`
- `NVIDIA L40`
- `NVIDIA L40S`

Pass multiple in `gpuTypeIds` array for fallback. Set `gpuTypePriority` to `"availability"` or `"price"`.

---

## HTTP Proxy Access

Services exposed via HTTP ports are accessible at:

```
https://{POD_ID}-{INTERNAL_PORT}.proxy.runpod.net
```

100-second Cloudflare timeout applies. For long-running requests, use TCP or implement polling.

---

## Data Center IDs

Available for global networking:

`CA-MTL-3`, `EU-CZ-1`, `EU-FR-1`, `EU-NL-1`, `EU-RO-1`, `EU-SE-1`, `EUR-IS-2`, `OC-AU-1`, `US-CA-2`, `US-GA-1`, `US-GA-2`, `US-IL-1`, `US-KS-2`, `US-NC-1`, `US-TX-3`, `US-TX-4`, `US-WA-1`

Network volume and pod must be in same DC.
