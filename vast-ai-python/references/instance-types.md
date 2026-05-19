# Instance Types

Three types with different priority/pricing tradeoffs. Type is set at creation and cannot be changed (except on-demand → reserved conversion).

## On-demand

- Fixed host-set price, guaranteed resources, high priority
- Runs until max duration expires (shown on offer card) or you destroy it
- Expired instances may be deleted 48h after expiration — retrieve data before then
- Best for: production, continuous training, time-sensitive work

## Interruptible

- Bidding system — you set a max $/hr bid
- Lowest cost, often 50%+ cheaper than on-demand
- May be paused if outbid or if an on-demand renter claims the GPU
- Data is preserved when paused; instance resumes automatically when priority returns
- Priority rules: on-demand always wins → among interruptible, highest bid wins
- Best for: batch jobs, fault-tolerant training, dev/test

### SDK usage

```python
# Search interruptible offers
offers = vast.search_offers(type="bid", query="gpu_name=RTX_4090 rentable=true")

# Create with a bid price
result = vast.create_instance(id=OFFER_ID, image="pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime", disk=20, price=0.30)
```

### Interruptible best practices

- Checkpoint frequently to disk or cloud storage
- Use `cloud_copy()` or `copy()` to back up outputs after each epoch
- Expect interruptions — design for restartability
- Set bid slightly above market to reduce pause frequency

## Reserved

- Pre-paid discount on an existing on-demand instance (up to 50% off)
- Same high priority as on-demand
- Convert anytime: rent on-demand first, then convert via console discount badge
- Credits are locked to that specific instance
- Partial refunds available if cancelled early
- Best for: multi-day/week training, predictable workloads

## Decision criteria

| Scenario | Type | Reason |
|---|---|---|
| Production inference | On-demand | Guaranteed uptime |
| Multi-day fine-tuning | Reserved | Cost savings with commitment |
| Hyperparameter sweeps | Interruptible | Each run is short; restartable |
| Interactive development | On-demand | Need reliability for SSH sessions |
| Large batch preprocessing | Interruptible | Fault-tolerant, cost-sensitive |
