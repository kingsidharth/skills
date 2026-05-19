# Searching Offers

## SDK method

```python
VastAI.search_offers(
    type: Optional[str] = None,       # "bid" | "reserved" | "on-demand"
    no_default: bool = False,          # disable default filters (external=false rentable=true verified=true)
    limit: Optional[int] = None,
    disable_bundling: bool = False,    # show individual machine slots
    storage: Optional[float] = None,  # GiB to include in pricing calc
    order: Optional[str] = None,      # sort field; append "-" for desc
    query: Optional[str] = None       # filter string
) -> str
```

## Query string syntax

Space-separated `field=value` or `field>=value` / `field<=value` pairs.

### Key filter fields

| Field | Example | Meaning |
|---|---|---|
| `gpu_name` | `gpu_name=RTX_4090` | GPU model (underscore for spaces) |
| `num_gpus` | `num_gpus=4` or `num_gpus>=2` | GPU count |
| `gpu_ram` | `gpu_ram>=24` | Per-GPU VRAM in GB |
| `cpu_ram` | `cpu_ram>=32` | System RAM in GB |
| `verified` | `verified=true` | Only Vast-verified hosts |
| `rentable` | `rentable=true` | Currently available |
| `direct_port_count` | `direct_port_count>=1` | Has direct port access (needed for direct SSH) |
| `reliability` | `reliability>=0.95` | Historical uptime score (0–1) |
| `inet_down` | `inet_down>=500` | Download speed in Mbps |
| `disk_space` | `disk_space>=100` | Available disk in GB |
| `geolocation` | `geolocation=US` | Country code |

### Common order fields

- `dlperf_usd-` — best DL performance per dollar (descending)
- `dph_total+` — cheapest total $/hr first
- `dlperf-` — raw DL performance descending
- `score-` — composite score descending

## Machine tiers

- **Unverified**: new machines, filtered out by default
- **Verified**: passed Vast internal tests
- **Secure Cloud (Datacenter)**: verified + certified datacenter (Tier 2/3 or ISO 27001), blue label, recommended for production

## Scores

- **DLPerf**: Vast benchmark approximating real DL throughput for that GPU+host combo. Use to compare across GPU models.
- **Reliability**: historical uptime, starts at 60% for new machines, climbs with demonstrated availability.

## Example patterns

```python
# Cheapest 4090, verified, direct SSH
vast.search_offers(
    query="gpu_name=RTX_4090 num_gpus=1 verified=true direct_port_count>=1 rentable=true",
    order="dph_total+",
    limit=5,
)

# Multi-GPU A100 for training
vast.search_offers(
    query="gpu_name=A100_SXM4 num_gpus>=4 reliability>=0.95 rentable=true",
    order="dlperf_usd-",
    limit=10,
)

# Interruptible offers only
vast.search_offers(
    type="bid",
    query="gpu_name=RTX_4090 rentable=true",
    order="dph_total+",
)
```

## REST API equivalent

```
POST https://console.vast.ai/api/v0/bundles/
Authorization: Bearer $VAST_API_KEY

{
  "gpu_name": {"in": ["RTX 4090"]},
  "num_gpus": {"gte": 1},
  "reliability": {"gte": 0.99},
  "verified": {"eq": true},
  "rentable": {"eq": true},
  "type": "ondemand",
  "limit": 5
}
```
