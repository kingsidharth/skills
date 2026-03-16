# Storage Reference

## Storage Types

| Type | Use Case | Persistence |
|------|----------|-------------|
| Volume | Model weights, datasets | Persistent |
| CloudBucketMount | S3/B2/GCS | External cloud |
| Dict | Key-value state | Persistent |
| Queue | Job queues | Persistent |
| Container FS | Temp files | Ephemeral |

## Volumes

```python
vol = modal.Volume.from_name("my-vol", create_if_missing=True)

@app.function(volumes={"/data": vol})
def save():
    with open("/data/file.txt", "w") as f:
        f.write("data")
    vol.commit()  # CRITICAL: persist changes

@app.function(volumes={"/data": vol})
def load():
    vol.reload()  # CRITICAL: get latest
    with open("/data/file.txt", "r") as f:
        return f.read()
```

**Volume v2** (beta): Unlimited files, concurrent writes

```python
vol = modal.Volume.from_name("v2-vol", create_if_missing=True, version=2)
```

## Backblaze B2 Integration

**CRITICAL**: B2 is S3-compatible BUT:
- NO presigned URLs support
- Must provide `bucket_endpoint_url`
- Regional endpoint: `s3.us-west-004.backblazeb2.com`

```python
b2_secret = modal.Secret.from_name("b2-creds")  # AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY

@app.function(
    volumes={
        "/data": modal.CloudBucketMount(
            bucket_name="my-b2-bucket",
            bucket_endpoint_url="https://s3.us-west-004.backblazeb2.com",  # REQUIRED
            secret=b2_secret,
            read_only=False
        )
    }
)
def process_b2():
    import os
    # Read/write files like local filesystem
    files = os.listdir("/data")
```

**B2 Application Key Setup**:
1. Create key in B2 dashboard
2. Enable "List All Bucket Names" capability
3. Note keyID and applicationKey
4. Create Modal secret with AWS_ACCESS_KEY_ID=keyID, AWS_SECRET_ACCESS_KEY=applicationKey

**B2 Endpoint by Region**:
- us-west-001: s3.us-west-001.backblazeb2.com
- us-west-004: s3.us-west-004.backblazeb2.com
- eu-central-003: s3.eu-central-003.backblazeb2.com

## HuggingFace Downloads in Tranches

```python
hf_image = modal.Image.debian_slim() \
    .pip_install("huggingface-hub[hf_transfer]") \
    .env({"HF_HUB_ENABLE_HF_TRANSFER": "1"})

@app.function(
    image=hf_image,
    volumes={"/cache": vol},
    timeout=3600
)
def download_model(repo_id: str):
    from huggingface_hub import snapshot_download
    
    snapshot_download(
        repo_id=repo_id,
        local_dir=f"/cache/{repo_id}",
        allow_patterns=["*.safetensors", "*.json"],  # Selective download
        ignore_patterns=["*.bin"],  # Skip PyTorch weights
        max_workers=8  # Parallel downloads (tranches)
    )
    vol.commit()
```

**Incremental downloads**:
```python
@app.function(volumes={"/cache": vol})
def download_incremental(repo_id: str):
    from huggingface_hub import hf_hub_download, HfApi
    
    api = HfApi()
    files = api.list_repo_files(repo_id)
    
    # Download in batches
    batch_size = 10
    for i in range(0, len(files), batch_size):
        for file in files[i:i+batch_size]:
            hf_hub_download(repo_id, filename=file, local_dir="/cache")
        vol.commit()  # Commit after each batch
```

## S3, GCS, R2 Mounts

**AWS S3**:
```python
s3_secret = modal.Secret.from_name("aws")  # AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION

@app.function(
    volumes={
        "/s3": modal.CloudBucketMount(bucket_name="my-bucket", secret=s3_secret)
    }
)
def read_s3():
    pass
```

**GCS**:
```python
gcs_secret = modal.Secret.from_name("gcp")  # GOOGLE_ACCESS_KEY_ID, GOOGLE_ACCESS_KEY_SECRET

@app.function(
    volumes={
        "/gcs": modal.CloudBucketMount(
            bucket_name="my-gcs-bucket",
            bucket_endpoint_url="https://storage.googleapis.com",
            secret=gcs_secret
        )
    }
)
```

**Cloudflare R2**:
```python
@app.function(
    volumes={
        "/r2": modal.CloudBucketMount(
            bucket_name="my-r2-bucket",
            bucket_endpoint_url="https://<ACCOUNT_ID>.r2.cloudflarestorage.com",
            secret=r2_secret
        )
    }
)
```

## Dicts and Queues

**Dict**:
```python
d = modal.Dict.from_name("config", create_if_missing=True)

@app.function()
def use_dict():
    d["key"] = "value"
    value = d["key"]
```

**Queue**:
```python
q = modal.Queue.from_name("jobs", create_if_missing=True)

@app.function()
def producer():
    q.put({"task": "data"})

@app.function()
def consumer():
    while not q.empty():
        task = q.get()
        process(task)
```

## Model Weight Storage Strategy

**Recommended**: Use Volume
```python
# Download once to Volume
@app.function(volumes={"/models": vol})
def download():
    download_weights_to("/models")
    vol.commit()

# Use in all functions
@app.function(gpu="A10G", volumes={"/models": vol})
def inference():
    model = load("/models/...")
```

**Alternative**: CloudBucketMount for existing cloud storage
**Avoid**: Baking into Image (rebuilds on code changes)

## Troubleshooting

**"Can't find file on Volume"** → Add `vol.reload()` before reading  
**"Busy Volume"** → Don't modify same files concurrently, use separate files  
**"Region detection failed"** → Add AWS_REGION to secret

