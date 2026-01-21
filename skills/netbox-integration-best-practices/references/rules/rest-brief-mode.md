---
title: Use Brief Mode for List Operations
impact: HIGH
category: rest
tags: [rest, performance, brief, optimization]
netbox_version: "4.4+"
---

# rest-brief-mode: Use Brief Mode for List Operations

## Rationale

Brief mode (`?brief=True`) returns a minimal representation of objects, typically reducing response size by 90% or more. This dramatically improves:
- Network transfer time
- JSON parsing time
- Memory usage
- Overall response latency

Use brief mode whenever you don't need full object details.

## Incorrect Pattern

```python
# WRONG: Full response when only ID and name needed
import requests

# Fetching device list for a dropdown menu
response = requests.get(
    f"{API_URL}/dcim/devices/",
    headers=headers
)
# Response: ~2KB per device with all fields
# For 1000 devices: ~2MB transfer

# But we only use:
options = [{"value": d["id"], "label": d["name"]} for d in response.json()["results"]]
```

**Problems with this approach:**
- Transferring unnecessary data
- Slower response times
- Higher memory usage
- Wasted bandwidth and processing

## Correct Pattern

```python
# CORRECT: Brief mode for dropdown population
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

response = requests.get(
    f"{API_URL}/dcim/devices/?brief=True",
    headers=headers
)
# Response: ~200 bytes per device
# For 1000 devices: ~200KB transfer (10x smaller)

devices = response.json()["results"]
options = [{"value": d["id"], "label": d["display"]} for d in devices]
```

**Benefits:**
- ~90% reduction in response size
- Faster response times
- Lower memory footprint
- Better scalability

## Brief Response Fields

Brief mode returns only essential fields:

```json
{
    "id": 123,
    "url": "https://netbox.example.com/api/dcim/devices/123/",
    "display": "switch-01",
    "name": "switch-01"
}
```

The exact fields vary by object type but typically include:
- `id`: Object identifier
- `url`: API URL for full object
- `display`: Human-readable display name
- Natural key fields (e.g., `name`, `slug`)

## Size Comparison

| Object Type | Full Response | Brief Response | Reduction |
|-------------|--------------|----------------|-----------|
| Device | ~2,000 bytes | ~200 bytes | 90% |
| Prefix | ~800 bytes | ~150 bytes | 81% |
| Site | ~1,200 bytes | ~180 bytes | 85% |
| Interface | ~600 bytes | ~120 bytes | 80% |

## Use Cases for Brief Mode

| Scenario | Use Brief? |
|----------|-----------|
| Dropdown/select menus | Yes |
| Autocomplete suggestions | Yes |
| Reference lists (for FK selection) | Yes |
| Existence checks | Yes |
| Relationship validation | Yes |
| Displaying object details | No |
| Bulk updates (need current values) | No |
| Reports needing all fields | No |

## Combining with Other Parameters

```python
# Brief mode with filters and pagination
response = requests.get(
    f"{API_URL}/dcim/devices/?brief=True&site=nyc-dc1&limit=100",
    headers=headers
)

# Brief mode with ordering
response = requests.get(
    f"{API_URL}/dcim/devices/?brief=True&ordering=name",
    headers=headers
)
```

## pynetbox Note

pynetbox doesn't have a direct brief mode parameter, but you can use the underlying request:

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# pynetbox always fetches full objects
# For brief mode, use requests directly or filter early

# Alternative: Use filter to limit what you fetch
sites = nb.dcim.sites.filter(status="active")
options = [{"value": s.id, "label": s.name} for s in sites]
```

## Exceptions

- **Detail views:** Need full object data
- **Editing forms:** Need all current field values
- **Nested serializer access:** Brief mode doesn't include nested objects

## Related Rules

- [rest-field-selection](./rest-field-selection.md) - More granular field control
- [rest-exclude-config-context](./rest-exclude-config-context.md) - Exclude heavy fields
- [perf-brief-mode-lists](./perf-brief-mode-lists.md) - Performance impact

## References

- [NetBox REST API - Brief Mode](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#brief-format)
