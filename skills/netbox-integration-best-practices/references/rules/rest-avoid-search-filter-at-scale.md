---
title: Avoid Search Filter at Scale
impact: HIGH
category: rest
tags: [rest, performance, search, filtering]
netbox_version: "4.4+"
---

# rest-avoid-search-filter-at-scale: Avoid Search Filter at Scale

## Rationale

The `q=` search parameter provides simple text search across multiple fields, but it becomes extremely slow with large datasets. This is especially severe for devices with primary IPs assigned.

Community reports indicate that `q=` queries can take orders of magnitude longer than equivalent specific filters on deployments with thousands of devices.

## Incorrect Pattern

```python
# WRONG: Using generic search with large datasets
import requests

# This can take 10-60+ seconds with thousands of devices
response = requests.get(
    f"{API_URL}/dcim/devices/?q=switch",
    headers=headers
)
```

**Problems with this approach:**
- `q=` searches across multiple fields simultaneously
- No index optimization for this search pattern
- Dramatically slower as dataset grows
- Especially slow when devices have primary IPs assigned

## Correct Pattern

```python
# CORRECT: Use specific filters instead of generic search
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

# Search by name (indexed, fast)
response = requests.get(
    f"{API_URL}/dcim/devices/?name__ic=switch",
    headers=headers
)

# Or by multiple specific fields
response = requests.get(
    f"{API_URL}/dcim/devices/?name__ic=switch&status=active",
    headers=headers
)
```

**Benefits:**
- Uses indexed database queries
- Orders of magnitude faster
- Predictable performance at scale

## Performance Comparison

| Query | 100 devices | 5000 devices | 20000 devices |
|-------|-------------|--------------|---------------|
| `q=switch` | 0.5s | 10-30s | 60s+ / timeout |
| `name__ic=switch` | 0.1s | 0.3s | 0.8s |

Times are approximate and vary by hardware and data characteristics.

## Specific Filter Alternatives

| Instead of `q=` for... | Use specific filter |
|------------------------|---------------------|
| Device name | `name__ic=` |
| Serial number | `serial__ic=` |
| Asset tag | `asset_tag__ic=` |
| Comments | `comments__ic=` |
| Site name | `site=` (exact) or filter site first |

## Multi-Field Search Alternative

If you need to search across multiple fields, do it in parallel with specific filters:

```python
import asyncio
import httpx

async def search_devices(term, api_url, headers):
    """Search devices across multiple fields with specific filters."""
    async with httpx.AsyncClient(headers=headers) as client:
        # Search different fields in parallel
        tasks = [
            client.get(f"{api_url}/dcim/devices/?name__ic={term}&limit=50"),
            client.get(f"{api_url}/dcim/devices/?serial__ic={term}&limit=50"),
            client.get(f"{api_url}/dcim/devices/?asset_tag__ic={term}&limit=50"),
        ]

        responses = await asyncio.gather(*tasks)

        # Combine and deduplicate results
        seen_ids = set()
        results = []
        for resp in responses:
            for device in resp.json()["results"]:
                if device["id"] not in seen_ids:
                    seen_ids.add(device["id"])
                    results.append(device)

        return results
```

## Client-Side Search

For small datasets or autocomplete, consider fetching filtered data and searching client-side:

```python
# For autocomplete with <1000 items, this may be faster overall
# Fetch once, cache, and filter client-side

devices = requests.get(
    f"{API_URL}/dcim/devices/?brief=True&status=active",
    headers=headers
).json()["results"]

# Client-side filtering (instant)
def search_devices(term, devices):
    term_lower = term.lower()
    return [d for d in devices if term_lower in d["name"].lower()]
```

## When `q=` is Acceptable

- **Small datasets:** < 500 total objects
- **One-off queries:** Manual exploration, not production code
- **Admin/debugging:** When you accept the performance cost

## Exceptions

- **Small NetBox instances:** With < 1000 devices, `q=` may be acceptable
- **Non-device endpoints:** Some object types may not have this issue

## Related Rules

- [rest-filtering-expressions](./rest-filtering-expressions.md) - Use lookup expressions
- [rest-exclude-config-context](./rest-exclude-config-context.md) - Another major performance issue
- [perf-parallel-requests](./perf-parallel-requests.md) - Parallelize searches

## References

- [NetBox GitHub Discussions - Search Performance](https://github.com/netbox-community/netbox/discussions)
- [NetBox Filtering](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#filtering)
