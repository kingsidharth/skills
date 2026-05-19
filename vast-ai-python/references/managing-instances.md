# Managing Instances

## Lifecycle

```
create → loading → running → stopped → destroyed
                          ↓ (interruptible)
                        paused → running (auto-resume)
```

## SDK methods

```python
vast.show_instances()                  # list all instances
vast.show_instance(id=ID)              # single instance details
vast.stop_instance(id=ID)              # pause — GPU billing stops, storage continues
vast.start_instance(id=ID)             # restart a stopped instance (GPU must be available)
vast.destroy_instance(id=ID)           # permanent delete, stops ALL billing
vast.reboot_instance(id=ID)            # restart container without recreating
vast.label_instance(id=ID, label="x")  # set display name
```

## Status values

| Status | Meaning |
|---|---|
| `creating` | Vast initiating instance |
| `loading` | Docker image downloading |
| `running` | Ready to use |
| `exited` | Container crashed |
| `offline` | Host disconnected |
| `inactive` | Stopped, data preserved |
| `scheduling` | Attempting restart, waiting for GPU |

## Billing rules

- **Running**: GPU compute + storage + bandwidth — all billed per second
- **Stopped**: storage only (may be higher rate than running)
- **Destroyed**: all billing stops, data is permanently lost
- Expired instances may be deleted 48h after expiration

## Connecting

```python
ssh_url = vast.ssh_url(id=ID)  # returns host:port for SSH

# Data transfer
vast.copy(src="local:./data/", dst=f"{ID}:/workspace/data/")
vast.copy(src=f"{ID}:/workspace/results/", dst="local:./results/")

# Instance to instance
vast.copy(src=f"{ID_A}:/workspace/", dst=f"{ID_B}:/workspace/")

# Cloud storage (requires configured cloud connection)
vast.cloud_copy(src=f"s3.CONN_ID:/bucket/data/", dst=f"{ID}:/workspace/")
```

## Interruptible-specific behavior

- When outbid or on-demand claims GPU: instance is **paused** (not destroyed)
- Data is preserved while paused
- Resumes automatically when priority returns
- During pause: storage charges continue, compute billing stops
- No user action needed for resume — happens automatically

## Restart caveats

After `stop_instance`, restarting requires the GPU to still be available on that host. If another renter took it, restart enters `scheduling` state and waits. For interruptible instances, the host may have rented the GPU to an on-demand user.
