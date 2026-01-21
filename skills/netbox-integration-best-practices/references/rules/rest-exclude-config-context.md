---
title: Exclude Config Context from Device Lists
impact: HIGH
category: rest
tags: [rest, performance, config-context, optimization]
netbox_version: "4.4+"
---

# rest-exclude-config-context: Exclude Config Context from Device Lists

## Rationale

Config context is the single most expensive field to compute for device queries. It requires:
- Traversing the device hierarchy (region → site → location → rack → device)
- Evaluating config context rules at each level
- Merging multiple context sources with precedence
- Serializing potentially large JSON structures

**Community-reported impact:** Device list requests can be **10-100x slower** with config context included. This is confirmed across multiple large NetBox deployments.

Always exclude config context from list operations unless specifically needed.

## Incorrect Pattern

```python
# WRONG: Fetching device list with config context (implicit)
import requests

response = requests.get(
    f"{API_URL}/dcim/devices/",
    headers=headers
)
# Response includes config_context for each device
# With 1000 devices: may take 30+ seconds instead of <1 second
```

**Problems with this approach:**
- Config context computed for every device
- Dramatically slower response times
- Often the config context isn't even used
- May cause request timeouts with large datasets

## Correct Pattern

```python
# CORRECT: Exclude config context from device lists
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

response = requests.get(
    f"{API_URL}/dcim/devices/?exclude=config_context",
    headers=headers
)
# Response is 10-100x faster
```

**Benefits:**
- Dramatically faster response times
- Reduced database load
- Better scalability
- No unnecessary computation

## Performance Comparison

| Scenario | With config_context | Without (excluded) |
|----------|--------------------|--------------------|
| 100 devices | 2-5 seconds | 0.1-0.2 seconds |
| 1000 devices | 20-60 seconds | 0.5-1 second |
| 5000 devices | Timeout likely | 2-5 seconds |

Actual times vary based on config context complexity and server resources.

## When Config Context IS Needed

Fetch config context only when specifically required:

```python
# For individual device where config context is needed
response = requests.get(
    f"{API_URL}/dcim/devices/123/",  # Single device, full details
    headers=headers
)
config = response.json()["config_context"]

# Or fetch for specific devices only
device_ids = [1, 2, 3]
for device_id in device_ids:
    response = requests.get(
        f"{API_URL}/dcim/devices/{device_id}/",
        headers=headers
    )
    # Process config context individually
```

## Combining with Other Optimizations

```python
# Maximum optimization for device lists
response = requests.get(
    f"{API_URL}/dcim/devices/"
    f"?exclude=config_context"  # Exclude heavy computation
    f"&brief=True"              # Minimal fields
    f"&limit=100",              # Paginate
    headers=headers
)
```

## Excluding Multiple Fields

The `exclude` parameter accepts comma-separated field names:

```python
# Exclude multiple heavy fields
response = requests.get(
    f"{API_URL}/dcim/devices/?exclude=config_context,local_context_data",
    headers=headers
)
```

## pynetbox Consideration

pynetbox doesn't directly support the `exclude` parameter. For performance-critical list operations, use requests directly:

```python
import requests
import pynetbox

# For performance-critical lists, use requests
response = requests.get(
    f"{API_URL}/dcim/devices/?exclude=config_context&limit=1000",
    headers=headers
)
devices = response.json()["results"]

# pynetbox for individual device operations
nb = pynetbox.api("https://netbox.example.com", token=TOKEN)
device = nb.dcim.devices.get(123)  # Single device is fine
config = device.config_context
```

## Virtual Machines

The same applies to virtual machines:

```python
response = requests.get(
    f"{API_URL}/virtualization/virtual-machines/?exclude=config_context",
    headers=headers
)
```

## Exceptions

- **Single device/VM fetch:** Config context overhead is acceptable for individual objects
- **Config rendering:** When you specifically need to render device configuration
- **Automation requiring context:** Ansible, Nornir workflows that use config context

## Related Rules

- [rest-brief-mode](./rest-brief-mode.md) - Use brief mode for lists
- [rest-field-selection](./rest-field-selection.md) - Select only needed fields
- [perf-exclude-config-context](./perf-exclude-config-context.md) - Performance impact details
- [rest-avoid-search-filter-at-scale](./rest-avoid-search-filter-at-scale.md) - Another major performance issue

## References

- [NetBox GitHub Discussion - Performance Issues](https://github.com/netbox-community/netbox/discussions)
- [NetBox Config Contexts](https://netboxlabs.com/docs/netbox/en/stable/features/context-data/)
