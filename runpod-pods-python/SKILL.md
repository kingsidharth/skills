---
name: runpod-pods-python
description: RunPod GPU pods — Python SDK, REST API, CLI. Create/start/stop/terminate pods, spot instances, network volumes, templates, storage, SSH, file transfer, Backblaze B2 sync, Docker images, environment variables, secrets.
---

# RunPod Pods — Python Automation

## Requirements

- `pip install runpod` / `uv add runpod` — Python 3.8+
- `RUNPOD_API_KEY` env var (Settings > API Keys in console)

## SDK vs REST API

| | Python SDK (`runpod`) | REST API (`rest.runpod.io/v1`) |
|---|---|---|
| Pod CRUD | ✓ | ✓ |
| Spot pods | ✗ | ✓ (`interruptible: true`) |
| Network volumes | ✗ | ✓ |
| Cloud type selection | ✗ | ✓ |
| Templates CRUD | ✗ | ✓ |
| Data center pinning | ✗ | ✓ |

Use SDK for simple scripts. Use REST for anything involving spot, network volumes, or templates. See [rest-api.md](references/rest-api.md) for full parameter reference.

## SDK Quick Start

```python
import runpod, os
runpod.api_key = os.environ["RUNPOD_API_KEY"]

pod = runpod.create_pod(
    name="my-pod",
    image_name="runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04",
    gpu_type_id="NVIDIA GeForce RTX 4090",
    gpu_count=1,
    volume_in_gb=50,
    container_disk_in_gb=20,
    ports="8888/http,22/tcp",
    volume_mount_path="/workspace",
)
pod_id = pod["id"]

runpod.get_pods()            # list all
runpod.get_pod(pod_id)       # get one
runpod.stop_pod(pod_id)      # release GPU, keep volume
runpod.resume_pod(pod_id)    # restart stopped pod
runpod.terminate_pod(pod_id) # destroy pod + volume disk
```

## REST API — Common Patterns

```python
import requests, os

HEADERS = {
    "Authorization": f"Bearer {os.environ['RUNPOD_API_KEY']}",
    "Content-Type": "application/json",
}
BASE = "https://rest.runpod.io/v1"
```

### On-Demand Pod with Network Volume

```python
pod = requests.post(f"{BASE}/pods", headers=HEADERS, json={
    "name": "training-run",
    "imageName": "runpod/pytorch:2.1.0-py3.10-cuda11.8.0-devel-ubuntu22.04",
    "gpuTypeIds": ["NVIDIA GeForce RTX 4090"],
    "gpuCount": 1,
    "networkVolumeId": "vol_abc123",
    "volumeMountPath": "/workspace",
    "containerDiskInGb": 50,
    "cloudType": "SECURE",
    "ports": ["8888/http", "22/tcp"],
    "env": {"MY_VAR": "value"},
}).json()
```

### Spot (Interruptible) Pod

5-second SIGTERM before kill. Save work to `/workspace` frequently.

```python
pod = requests.post(f"{BASE}/pods", headers=HEADERS, json={
    "name": "batch-job",
    "imageName": "my-registry/my-image:v1",
    "gpuTypeIds": ["NVIDIA GeForce RTX 4090", "NVIDIA RTX A5000"],
    "gpuCount": 1,
    "interruptible": True,
    "cloudType": "SECURE",
    "networkVolumeId": "vol_abc123",
    "volumeMountPath": "/workspace",
    "containerDiskInGb": 30,
    "supportPublicIp": True,
    "ports": ["22/tcp"],
}).json()
```

### From Template

```python
pod = requests.post(f"{BASE}/pods", headers=HEADERS, json={
    "name": "from-template",
    "templateId": "abc123tmpl",
    "gpuTypeIds": ["NVIDIA GeForce RTX 4090"],
    "gpuCount": 1,
}).json()
```

### Pod Lifecycle

```python
requests.post(f"{BASE}/pods/{pod_id}/stop", headers=HEADERS)     # stop
requests.post(f"{BASE}/pods/{pod_id}/start", headers=HEADERS)    # resume
requests.post(f"{BASE}/pods/{pod_id}/restart", headers=HEADERS)  # restart
requests.delete(f"{BASE}/pods/{pod_id}", headers=HEADERS)        # terminate
requests.get(f"{BASE}/pods", headers=HEADERS).json()             # list all
requests.get(f"{BASE}/pods/{pod_id}", headers=HEADERS).json()    # get one
```

## Network Volumes

Persistent storage independent of pods. Survives terminate. Shared across pods (one at a time). Must be in same DC as pod.

```python
# Create
vol = requests.post(f"{BASE}/networkvolumes", headers=HEADERS, json={
    "name": "my-data",
    "size": 100,
    "dataCenterId": "US-TX-3",
}).json()

# List
requests.get(f"{BASE}/networkvolumes", headers=HEADERS).json()

# Resize (increase only)
requests.patch(f"{BASE}/networkvolumes/{vol_id}", headers=HEADERS, json={
    "size": 200,
}).json()

# Delete
requests.delete(f"{BASE}/networkvolumes/{vol_id}", headers=HEADERS)
```

Pricing: $0.07/GB/mo (< 1TB), $0.05/GB/mo (> 1TB). Max 4TB via API (contact support for larger).

Attach to pod: set `networkVolumeId` during creation. Mounts at `volumeMountPath` (default `/workspace`), replacing volume disk. Cannot attach/detach after creation — must terminate and redeploy.

## Templates

Reusable pod configurations. Define image, ports, env, storage, startup command once.

```python
# Create
tmpl = requests.post(f"{BASE}/templates", headers=HEADERS, json={
    "name": "my-training-env",
    "imageName": "my-registry/my-image:v1",
    "containerDiskInGb": 50,
    "volumeInGb": 100,
    "volumeMountPath": "/workspace",
    "ports": ["8888/http", "22/tcp"],
    "env": {"MODEL_NAME": "my-model"},
    "dockerStartCmd": ["python", "main.py"],
    "isPublic": False,
}).json()

# Use in pod creation
pod = requests.post(f"{BASE}/pods", headers=HEADERS, json={
    "name": "from-template",
    "templateId": tmpl["id"],
    "gpuTypeIds": ["NVIDIA GeForce RTX 4090"],
    "gpuCount": 1,
}).json()

# List / Update / Delete
requests.get(f"{BASE}/templates", headers=HEADERS).json()
requests.patch(f"{BASE}/templates/{tmpl_id}", headers=HEADERS, json={...}).json()
requests.delete(f"{BASE}/templates/{tmpl_id}", headers=HEADERS)
```

### Secrets in Templates

Secrets managed via console only. Reference in env vars:

```python
"env": {"API_KEY": "{{ RUNPOD_SECRET_my_api_key }}"}
```

## Storage

| Type | Persists | Mount | Cost | Notes |
|---|---|---|---|---|
| Container disk | Cleared on stop | System-managed | $0.10/GB/mo | Temp files, OS |
| Volume disk | Until pod terminated | `/workspace` | $0.10 running, $0.20 stopped /GB/mo | Models, checkpoints |
| Network volume | Independent of pod | `/workspace` (replaces volume disk) | $0.07/GB/mo | Portable, shared |

## Connecting to Pods

### HTTP Proxy

Services on exposed HTTP ports accessible at:
```
https://{POD_ID}-{INTERNAL_PORT}.proxy.runpod.net
```
100s Cloudflare timeout. HTTPS only.

### SSH

Basic (proxied, no SCP/SFTP):
```bash
ssh {POD_ID_PREFIX}@ssh.runpod.io -i ~/.ssh/id_ed25519
```

Full SSH via public IP (supports SCP/SFTP) — requires TCP port 22 exposed + `supportPublicIp: true`:
```bash
ssh root@{PUBLIC_IP} -p {EXTERNAL_PORT} -i ~/.ssh/id_ed25519
```

External port from `portMappings` in pod response or `RUNPOD_TCP_PORT_22` env var inside pod.

### Pod-to-Pod (Global Networking)

Set `globalNetworking: true`. Access other pods at `{POD_ID}.runpod.internal`. 100 Mbps. NVIDIA GPU pods only.

## File Transfer

| Method | Best for | Requires |
|---|---|---|
| `runpodctl send/receive` | Quick one-off transfers | Pre-installed on pods |
| SCP | Standard file ops | SSH + public IP |
| rsync | Large datasets, incremental sync | SSH + rsync installed |
| Cloud Sync | Backup to S3/GCS/Azure/B2/Dropbox | Cloud provider creds |

### runpodctl

```bash
# On source
runpodctl send myfile.tar.gz
# Output: Code is: 8338-galileo-collect-fidel

# On destination
runpodctl receive 8338-galileo-collect-fidel
```

### Backblaze B2

Console: Pod page > Cloud Sync > Backblaze B2 (needs Account ID + App Key + bucket path).

Programmatic from inside pod: see [b2-sync.md](references/b2-sync.md).

## Environment Variables

Set via `env` field in pod creation, template, or console.

Auto-provided inside every pod:

| Variable | Description |
|---|---|
| `RUNPOD_POD_ID` | Pod ID |
| `RUNPOD_DC_ID` | Data center |
| `RUNPOD_GPU_COUNT` | GPU count |
| `RUNPOD_PUBLIC_IP` | Public IP (if available) |
| `RUNPOD_TCP_PORT_22` | External SSH port |
| `RUNPOD_API_KEY` | Pod-scoped API key |
| `PUBLIC_KEY` | SSH public keys |
| `CUDA_VERSION` | CUDA version |

## Docker / Custom Templates

Extend RunPod base images (`runpod/pytorch:*`). Build with `--platform linux/amd64`. Push to Docker Hub or private registry (`containerRegistryAuthId` for private).

Startup: default = Jupyter + SSH. Override with `dockerStartCmd` (run after services) or `dockerEntrypoint` (replace everything).

## Decision Tables

| | On-demand | Spot |
|---|---|---|
| Interruptible | No | Yes (5s SIGTERM) |
| Cost | Standard | ~50-70% cheaper |
| Use for | Training, dev | Batch inference, fault-tolerant |

| | Secure Cloud | Community Cloud |
|---|---|---|
| Reliability | T3/T4 data centers | Peer-to-peer |
| Price | Higher | Lower |
| Use for | Production | Experimentation |

## Reference Files

- [rest-api.md](references/rest-api.md) — Full REST API parameters, GPU type IDs, data center IDs
- [b2-sync.md](references/b2-sync.md) — Backblaze B2 sync from pods (console + rclone + b2 CLI)
- [tailscale-setup.md](references/tailscale-setup.md) — Optional: Tailscale private networking inside pods
