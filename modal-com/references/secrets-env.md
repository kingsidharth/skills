# Secrets and Environment Variables Reference

## Creating Secrets

**Dashboard**: modal.com/secrets (recommended)

**From code**:
```python
# From dict
secret = modal.Secret.from_dict({
    "API_KEY": "sk-...",
    "DATABASE_URL": "postgresql://..."
})

# From .env file
secret = modal.Secret.from_dotenv(__file__)  # Loads .env in same dir

# From environment
secret = modal.Secret.from_local_environ("API_KEY", "DATABASE_URL")
```

## Using Secrets

```python
@app.function(secrets=[modal.Secret.from_name("my-secret")])
def use_secret():
    import os
    api_key = os.environ["API_KEY"]  # Available as env var
```

## Modal Runtime Environment Variables

Available in all containers:
- `MODAL_ENVIRONMENT`: Environment name
- `MODAL_IS_REMOTE`: "1" if in cloud
- `MODAL_CLOUD_PROVIDER`: AWS/GCP/OCI
- `MODAL_REGION`: Cloud region
- `MODAL_IMAGE_ID`: Image identifier

## Conditional Secrets (Dev vs Prod)

```python
import modal

if modal.is_local():
    secret = modal.Secret.from_dict({"KEY": "dev-key"})
else:
    secret = modal.Secret.from_name("prod-secret")

@app.function(secrets=[secret])
def conditional():
    pass
```

