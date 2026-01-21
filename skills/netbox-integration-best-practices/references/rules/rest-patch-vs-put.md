---
title: Use PATCH for Partial Updates
impact: HIGH
category: rest
tags: [rest, update, patch, put]
netbox_version: "4.4+"
---

# rest-patch-vs-put: Use PATCH for Partial Updates

## Rationale

`PUT` replaces the entire object; omitted fields may be reset to defaults or cleared. `PATCH` updates only the specified fields, leaving others unchanged.

Using `PUT` when you intend `PATCH` can cause data loss or unexpected field resets.

## Incorrect Pattern

```python
# WRONG: Using PUT for partial update
import requests

# Original device has many fields: name, status, comments, tenant, rack, etc.
# We only want to change status

response = requests.put(
    f"{API_URL}/dcim/devices/123/",
    headers=headers,
    json={"status": "active"}  # All other fields omitted!
)
# DANGER: Other fields may be cleared or reset to defaults
```

**Problems with this approach:**
- Omitted required fields cause validation errors
- Omitted optional fields may be cleared
- Must send complete object for PUT
- Easy to accidentally lose data

## Correct Pattern

```python
# CORRECT: Using PATCH for partial update
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

# Only update the fields you want to change
response = requests.patch(
    f"{API_URL}/dcim/devices/123/",
    headers=headers,
    json={"status": "active"}  # Only status changes
)

if response.status_code == 200:
    updated_device = response.json()
    # All other fields are preserved
```

**Benefits:**
- Only specified fields are modified
- Other fields remain unchanged
- No risk of data loss
- Simpler request bodies

## When to Use PUT

Use `PUT` only when you intentionally want to replace the entire object:

```python
# CORRECT use of PUT: Replacing entire object
complete_device = {
    "name": "switch-01",
    "device_type": 1,
    "role": 1,
    "site": 1,
    "status": "active",
    "rack": 5,
    "position": 10,
    "comments": "Replaced device"
}

response = requests.put(
    f"{API_URL}/dcim/devices/123/",
    headers=headers,
    json=complete_device  # Complete object
)
```

## Method Comparison

| Aspect | PATCH | PUT |
|--------|-------|-----|
| Fields sent | Only changed | All fields |
| Omitted fields | Preserved | May be cleared |
| Use case | Partial update | Full replacement |
| Risk | Low | Data loss if incomplete |
| Payload size | Smaller | Larger |

## pynetbox Example

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# pynetbox uses PATCH by default
device = nb.dcim.devices.get(123)
device.status = "active"
device.save()  # PATCH request with only changed fields
```

## Updating Multiple Fields

```python
# PATCH supports updating multiple fields
response = requests.patch(
    f"{API_URL}/dcim/devices/123/",
    headers=headers,
    json={
        "status": "active",
        "comments": "Enabled in maintenance window",
        "custom_fields": {
            "last_maintenance": "2024-01-15"
        }
    }
)
```

## Bulk Updates with PATCH

```python
# Bulk PATCH to list endpoint
updates = [
    {"id": 1, "status": "active"},
    {"id": 2, "status": "active"},
    {"id": 3, "status": "planned"}
]

response = requests.patch(
    f"{API_URL}/dcim/devices/",  # List endpoint
    headers=headers,
    json=updates
)
```

## Exceptions

- **Full object sync:** When syncing from an external system that provides complete objects
- **Object reset:** When intentionally clearing optional fields

## Related Rules

- [rest-list-endpoint-bulk-ops](./rest-list-endpoint-bulk-ops.md) - Bulk update patterns
- [rest-error-handling](./rest-error-handling.md) - Handle update errors
- [rest-idempotency](./rest-idempotency.md) - Idempotent operations

## References

- [RFC 5789 - PATCH Method for HTTP](https://datatracker.ietf.org/doc/html/rfc5789)
- [NetBox REST API - Updating Objects](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#updating-objects)
