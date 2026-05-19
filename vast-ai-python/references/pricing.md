# Pricing

Marketplace model — hosts set prices, supply/demand drives rates. No fixed price quotes.

## Cost components

### GPU compute
- Billed per second while instance is running
- Varies by: GPU model, count, host reliability, region, market conditions
- H100/A100 command premium; consumer GPUs (4090, 3090) are cheaper

### Storage
- Billed continuously while instance exists, **including when stopped**
- Rate varies by host; stopped instances may cost more per GB than running
- Only way to stop storage billing: destroy the instance

### Bandwidth
- Per byte, both upload and download
- Rates vary by host — check during instance selection for data-heavy workloads

## Instance type pricing

| Type | Pricing model | Typical savings |
|---|---|---|
| On-demand | Fixed host price | Baseline |
| Reserved | Pre-paid commitment | Up to 50% off on-demand |
| Interruptible | Bidding (you set max $/hr) | Often 50%+ below on-demand |

## Finding cheapest options

```python
# Sort by total cost
vast.search_offers(query="gpu_name=RTX_4090 rentable=true", order="dph_total+")

# Interruptible for max savings
vast.search_offers(type="bid", query="gpu_name=RTX_4090 rentable=true", order="dph_total+")
```

Also consider: lower reliability scores, different regions, off-peak hours.

## Billing mechanics

- Credits required upfront (minimum deposit $5)
- Billed per second for actual usage
- When balance hits zero: instances auto-stop; if credit card on file, auto-charged; otherwise instances scheduled for deletion
- Enable autobilling to avoid interruptions
- Expired instances deleted 48h after expiration if not renewed

## Cost optimization tips

- Use interruptible for fault-tolerant work (biggest savings)
- Convert on-demand → reserved for multi-day jobs
- Destroy instances when done (don't just stop — storage still bills)
- Right-size disk at creation (can't resize, but don't over-allocate)
- Use `storage` param in `search_offers()` to include disk cost in price sorting
