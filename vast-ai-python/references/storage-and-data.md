# Storage & Data

## Container storage

- Fixed at instance creation — cannot resize
- Default 10 GB, set via `disk` param
- Persists while instance exists (including stopped state)
- Permanently deleted when instance is destroyed
- Billed continuously while instance exists, even when stopped

Allocate generously — running out mid-job requires creating a new instance and migrating data.

## Volumes

Local persistent storage that survives instance destruction.

### Constraints

- Tied to the **physical machine** where created — cannot migrate
- Can only attach to instances on the **same host**
- Fixed size at creation — cannot resize
- Must destroy attached instance before deleting volume
- Billed independently while volume exists

### SDK methods

```python
vast.create_volume(machine_id=MACHINE_ID, size=100, label="training-data")
vast.search_volumes()         # list your volumes
vast.show_volumes()           # detailed view
vast.delete_volume(id=VOL_ID)
vast.clone_volume(id=VOL_ID)  # duplicate on same machine
```

### Attaching to an instance

At creation time via REST API:
```json
{"volume_info": {"volume_id": 12345, "mount_path": "/data"}}
```

Or via the GUI's "Add volume" dropdown when creating an instance.

## Data movement

### SDK copy

```python
# Local → instance
vast.copy(src="local:./data/", dst=f"{ID}:/workspace/data/")

# Instance → local
vast.copy(src=f"{ID}:/results/", dst="local:./results/")

# Between instances
vast.copy(src=f"{A}:/workspace/", dst=f"{B}:/workspace/")
```

### Cloud sync

```python
# S3-compatible (requires configured cloud connection)
vast.cloud_copy(src=f"s3.CONN_ID:/bucket/path/", dst=f"{ID}:/workspace/")
vast.cloud_copy(src=f"{ID}:/workspace/output/", dst=f"s3.CONN_ID:/bucket/output/")
```

Supports Google Drive, S3, and other providers via cloud connections configured in the console.

### SCP/SFTP

Standard SSH-based transfer works with any running instance via the SSH connection details from `vast.ssh_url(id=ID)`.

## Billing summary

| Resource | When billed | How to stop |
|---|---|---|
| GPU compute | While running | Stop or destroy instance |
| Container storage | While instance exists | Destroy instance |
| Volume storage | While volume exists | Delete volume |
| Bandwidth | Per byte transferred | N/A |

Storage charges on stopped instances may be higher than on running instances (host-dependent).

## Data persistence strategy

- **Temp files / cache**: container storage — acceptable to lose
- **Models / datasets**: volumes (same-machine persistence) or cloud sync (off-machine)
- **Critical outputs**: cloud sync or `copy()` to local after each run
- For interruptible instances: checkpoint and sync frequently since pauses happen without warning
