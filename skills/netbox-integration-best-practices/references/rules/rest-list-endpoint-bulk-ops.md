---
title: Use List Endpoints for Bulk Operations
impact: CRITICAL
category: rest
tags: [rest, bulk, create, update, delete, transactions]
netbox_version: "4.4+"
---

# rest-list-endpoint-bulk-ops: Use List Endpoints for Bulk Operations

## Rationale

NetBox does NOT have separate "bulk endpoints." Instead, bulk create, update, and delete operations use the standard list endpoints with JSON arrays. This is a common source of confusion.

Key characteristics:
- **Atomic:** All items succeed or all fail (transaction rollback)
- **Efficient:** Single HTTP request for multiple objects
- **Validated:** Each item is fully validated before commit
- **Signal-safe:** Triggers Django signals and webhooks for each object

Understanding this pattern is essential for efficient data population and management.

## Incorrect Pattern

```python
# WRONG: Creating objects one at a time
import requests

devices_to_create = [
    {"name": "switch-01", "device_type": 1, "role": 1, "site": 1},
    {"name": "switch-02", "device_type": 1, "role": 1, "site": 1},
    {"name": "switch-03", "device_type": 1, "role": 1, "site": 1},
]

# Inefficient: 3 HTTP requests
for device in devices_to_create:
    response = requests.post(
        f"{API_URL}/dcim/devices/",
        headers=headers,
        json=device  # Single object
    )
```

```python
# WRONG: Looking for non-existent bulk endpoint
response = requests.post(
    f"{API_URL}/dcim/devices/bulk-create/",  # This endpoint doesn't exist!
    headers=headers,
    json=devices_to_create
)
```

**Problems with this approach:**
- Multiple HTTP requests (network overhead)
- No atomicity (partial success possible)
- Slower overall execution
- Looking for endpoints that don't exist

## Correct Pattern

### Bulk Create (POST array to list endpoint)

```python
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

# Create multiple devices in one request
devices = [
    {"name": "switch-01", "device_type": 1, "role": 1, "site": 1, "status": "active"},
    {"name": "switch-02", "device_type": 1, "role": 1, "site": 1, "status": "active"},
    {"name": "switch-03", "device_type": 1, "role": 1, "site": 1, "status": "planned"},
]

response = requests.post(
    f"{API_URL}/dcim/devices/",  # Regular list endpoint
    headers=headers,
    json=devices  # JSON array, not single object
)

if response.status_code == 201:
    created = response.json()  # Returns array of created objects
    for device in created:
        print(f"Created: {device['name']} (ID: {device['id']})")
else:
    print(f"Error: {response.json()}")
```

### Bulk Update (PATCH array to list endpoint)

```python
# Update multiple devices - each object MUST include "id"
updates = [
    {"id": 1, "status": "active"},
    {"id": 2, "status": "active"},
    {"id": 3, "status": "staged"},
]

response = requests.patch(
    f"{API_URL}/dcim/devices/",  # Same list endpoint
    headers=headers,
    json=updates  # Array with "id" in each object
)

if response.status_code == 200:
    updated = response.json()  # Returns array of updated objects
    for device in updated:
        print(f"Updated: {device['name']} -> {device['status']}")
```

### Bulk Delete (DELETE array to list endpoint)

```python
# Delete multiple devices
deletions = [
    {"id": 1},
    {"id": 2},
    {"id": 3},
]

response = requests.delete(
    f"{API_URL}/dcim/devices/",  # Same list endpoint
    headers=headers,
    json=deletions  # Array of {"id": X} objects
)

if response.status_code == 204:
    print("All devices deleted successfully")
```

## Atomicity Behavior

**All bulk operations are atomic (all-or-none):**

```python
# If any item fails validation, entire operation is rolled back
devices = [
    {"name": "switch-01", "device_type": 1, "role": 1, "site": 1},  # Valid
    {"name": "switch-02", "device_type": 1, "role": 1, "site": 1},  # Valid
    {"name": "", "device_type": 1, "role": 1, "site": 1},           # INVALID: empty name
]

response = requests.post(f"{API_URL}/dcim/devices/", headers=headers, json=devices)

# Response: 400 Bad Request
# NONE of the devices are created because one failed validation
# {
#   "2": {"name": ["This field may not be blank."]}
# }
```

## Signals and Webhooks

Despite being called "bulk" operations, NetBox processes each object **sequentially** rather than using Django's `bulk_create()`. This is intentional for validation purposes.

**Key implications:**

- **Signals fire per-object:** Django's `post_save` and `post_delete` signals trigger for each object in the array
- **Webhooks fire per-object:** Each created/updated/deleted object generates its own webhook event
- **Change logging:** Each object gets its own ObjectChange record

This means bulk operations:
- Are **slower** than true database-level bulk inserts
- Are **safer** because all validation and signals execute properly
- Generate **N webhook calls** for N objects (not one aggregated call)

> **Note:** If the same object is modified multiple times within a single request, only the final state triggers the webhook (event deduplication).

## pynetbox Example

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Bulk create - pass list instead of dict
devices = [
    {"name": f"switch-{i:02d}", "device_type": 1, "role": 1, "site": 1}
    for i in range(10)
]
created = nb.dcim.devices.create(devices)

# Result is list of created Record objects
for device in created:
    print(f"{device.name}: {device.id}")
```

## Async Example

```python
import httpx
import asyncio

async def bulk_create_devices(devices, api_url, headers):
    """Create devices in bulk using async client."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{api_url}/dcim/devices/",
            headers=headers,
            json=devices,
            timeout=60  # Bulk operations may take longer
        )
        response.raise_for_status()
        return response.json()
```

## Chunking Large Bulk Operations

For very large bulk operations, consider chunking:

```python
def bulk_create_chunked(endpoint_url, items, headers, chunk_size=100):
    """Create items in chunks to avoid timeouts."""
    created = []

    for i in range(0, len(items), chunk_size):
        chunk = items[i:i + chunk_size]
        response = requests.post(endpoint_url, headers=headers, json=chunk)
        response.raise_for_status()
        created.extend(response.json())

    return created
```

## Exceptions

- **Single object operations:** Use single object JSON for individual creates/updates
- **Very large batches:** May need chunking to avoid timeouts (not atomic across chunks)

## Related Rules

- [rest-patch-vs-put](./rest-patch-vs-put.md) - Use PATCH for partial updates
- [rest-error-handling](./rest-error-handling.md) - Handle bulk operation errors
- [data-dependency-order](./data-dependency-order.md) - Create objects in order

## References

- [NetBox REST API - Creating Objects](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#creating-objects)
- [NetBox REST API - Updating Objects](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#updating-objects)
- [NetBox REST API - Deleting Objects](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#deleting-objects)
