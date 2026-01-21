---
title: Always Paginate List Requests
impact: HIGH
category: rest
tags: [rest, pagination, performance, lists]
netbox_version: "4.4+"
---

# rest-pagination-required: Always Paginate List Requests

## Rationale

NetBox list endpoints can return thousands of objects. Without pagination:
- Memory exhaustion on client and server
- Request timeouts
- Slow response serialization
- Poor user experience

NetBox defaults to 50 items per page with a maximum of 1000. Always specify pagination explicitly for predictable behavior.

## Incorrect Pattern

```python
# WRONG: No pagination specified
import requests

response = requests.get(
    f"{API_URL}/dcim/devices/",
    headers=headers
)
devices = response.json()["results"]
# Only gets first 50 devices (default limit)
```

```python
# WRONG: Assuming all results are returned
response = requests.get(f"{API_URL}/dcim/devices/", headers=headers)
all_devices = response.json()["results"]  # May be incomplete!
```

**Problems with this approach:**
- Relies on server defaults (may change)
- Gets incomplete data (only first page)
- No explicit control over page size

## Correct Pattern

```python
# CORRECT: Explicit pagination
import requests

def get_all_objects(api_url, endpoint, headers, limit=100):
    """Fetch all objects with proper pagination."""
    all_results = []
    url = f"{api_url}/{endpoint}/?limit={limit}"

    while url:
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        data = response.json()

        all_results.extend(data["results"])
        url = data.get("next")  # None when no more pages

    return all_results

# Usage
all_devices = get_all_objects(API_URL, "dcim/devices", headers)
```

**Benefits:**
- Complete data retrieval
- Explicit page size control
- Memory-efficient processing
- Handles any dataset size

## Response Format

```json
{
    "count": 1500,
    "next": "https://netbox.example.com/api/dcim/devices/?limit=100&offset=100",
    "previous": null,
    "results": [...]
}
```

- `count`: Total objects matching query
- `next`: URL for next page (null if last page)
- `previous`: URL for previous page (null if first page)
- `results`: Array of objects for current page

## Pagination Parameters

| Parameter | Description | Default | Maximum |
|-----------|-------------|---------|---------|
| `limit` | Items per page | 50 | 1000 |
| `offset` | Skip N items | 0 | N/A |

```python
# First page
response = requests.get(f"{API_URL}/dcim/devices/?limit=100&offset=0")

# Second page
response = requests.get(f"{API_URL}/dcim/devices/?limit=100&offset=100")
```

## pynetbox Example

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# pynetbox handles pagination automatically
# .all() returns an iterator that fetches pages as needed
all_devices = nb.dcim.devices.all()

# Process without loading all into memory
for device in all_devices:
    process_device(device)

# Or convert to list if you need random access
device_list = list(nb.dcim.devices.all())
```

## Async Parallel Pagination

```python
import asyncio
import httpx

async def get_all_objects_parallel(api_url, endpoint, headers, limit=100):
    """Fetch all pages concurrently for faster retrieval."""
    async with httpx.AsyncClient(headers=headers) as client:
        # Get total count first
        response = await client.get(f"{api_url}/{endpoint}/?limit=1")
        total = response.json()["count"]

        # Calculate pages
        pages = (total + limit - 1) // limit

        # Fetch all pages concurrently
        tasks = [
            client.get(f"{api_url}/{endpoint}/?limit={limit}&offset={i * limit}")
            for i in range(pages)
        ]
        responses = await asyncio.gather(*tasks)

        # Combine results
        all_results = []
        for resp in responses:
            all_results.extend(resp.json()["results"])

        return all_results
```

## Recommended Page Sizes

| Use Case | Limit |
|----------|-------|
| Interactive UI | 25-50 |
| Background processing | 100-250 |
| Bulk export | 500-1000 |
| Memory-constrained | 50-100 |

## Exceptions

- **Single object:** `GET /api/dcim/devices/123/` doesn't need pagination
- **Count only:** If you only need the count, use `?limit=1` and read `count`

## Related Rules

- [rest-brief-mode](./rest-brief-mode.md) - Reduce payload size
- [graphql-always-paginate](./graphql-always-paginate.md) - GraphQL pagination
- [perf-pagination-strategy](./perf-pagination-strategy.md) - Choosing page sizes

## References

- [NetBox REST API - Pagination](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#pagination)
