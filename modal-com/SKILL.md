# Modal.com Skill

## Critical Rules

1. **ALWAYS READ REFERENCES FIRST**: Before writing Modal code, consult relevant reference file(s) in `references/`
2. **Python-First**: Primary language is Python; TypeScript/JavaScript via libmodal SDK for invocation only
3. **GPU Workloads**: ALWAYS enable memory snapshots for GPU functions (5-10x cold start improvement)
4. **Backblaze B2**: When using B2, note NO presigned URL support, must use S3-compatible API with endpoint URL
5. **Volumes for Persistence**: Container filesystems are ephemeral - use Volumes for persistent storage

## Navigation

**Starting new project** → `apps.md`  
**Need GPU/LLM inference** → `gpu-workloads.md`  
**Working with data/models** → `storage.md`  
**Building API** → `web-endpoints.md`  
**Slow cold starts** → `performance.md`  
**Something broken** → `errors.md`

### Reference Files

| Reference | When to Use | Key Topics |
|-----------|-------------|------------|
| `apps.md` | Creating/deploying apps, multi-file projects | Ephemeral vs deployed, environments, invocation patterns |
| `functions.md` | Writing functions, parallel execution | @function decorator, .map(), @enter/@exit lifecycle |
| `images.md` | Building containers, installing dependencies | pip_install, apt_install, run_commands, local files |
| `gpu-workloads.md` | GPU acceleration, LLM/image generation | GPU selection (T4→B200), CUDA, memory snapshots |
| `storage.md` | Data persistence, cloud storage, model weights | Volumes, B2/S3/GCS, HuggingFace downloads |
| `sandboxes.md` | Running untrusted/agent-generated code | Sandbox.create(), security isolation |
| `web-endpoints.md` | HTTP endpoints, APIs, streaming | FastAPI, ASGI/WSGI, WebSockets |
| `secrets-env.md` | Configuration, API keys, credentials | Secrets, environment variables, from_dotenv |
| `performance.md` | Optimization, cold start reduction | Memory snapshots, concurrent loading, warm pools |
| `errors.md` | Debugging, troubleshooting issues | Common errors, solutions, logs |

## Core Workflow

```python
import modal

# 1. Create app
app = modal.App("my-app")

# 2. Define image with dependencies
image = modal.Image.debian_slim() \
    .pip_install("torch", "transformers")

# 3. Create storage
volume = modal.Volume.from_name("models", create_if_missing=True)

# 4. Define function
@app.function(
    image=image,
    gpu="A10G",
    volumes={"/models": volume},
    enable_memory_snapshot=True  # CRITICAL for GPU
)
def process(data):
    # Your code runs in cloud with GPU
    return result

# 5. Run
# Development: modal run script.py
# Production: modal deploy script.py
```

## Common Patterns

### GPU Inference
See `gpu-workloads.md` for LLM inference, Stable Diffusion, multi-GPU patterns

### Backblaze B2 Storage
See `storage.md` for B2 mounting, HuggingFace downloads in tranches

### Web API
See `web-endpoints.md` for FastAPI integration, streaming responses

### Batch Processing
See `functions.md` for .map() parallelism, concurrent execution

## Languages Supported

**Python**: Primary language for defining Modal apps, functions, and all infrastructure code  
**TypeScript/JavaScript**: For invoking deployed Modal functions via libmodal SDK (not for defining functions)

```typescript
// Example: Calling deployed Modal function from TypeScript
import { Modal } from "@modalhq/libmodal";

const modal = new Modal({
  tokenId: process.env.MODAL_TOKEN_ID,
  tokenSecret: process.env.MODAL_TOKEN_SECRET
});

const result = await modal.function.call("my-app", "my-function", {arg: value});
```

## Key Value Propositions

**Modal's killer features**:
1. **Decorator-based deployment**: Just add `@app.function(gpu="H100")` - no infrastructure
2. **Memory snapshots**: 5-10x faster cold starts with `enable_memory_snapshot=True`
3. **Auto-scaling**: From 0 to hundreds of GPUs automatically
4. **Multi-cloud**: AWS/GCP/OCI pooling for availability and cost
5. **Volume persistence**: Distributed file system for model weights

