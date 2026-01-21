---
title: Use Lookup Expressions for Efficient Filtering
impact: MEDIUM
category: rest
tags: [rest, filtering, expressions, queries]
netbox_version: "4.4+"
---

# rest-filtering-expressions: Use Lookup Expressions for Efficient Filtering

## Rationale

NetBox provides powerful filtering with lookup expressions beyond simple equality. Using these expressions:
- Reduces data transfer (filter server-side)
- Leverages database indexes
- Enables complex queries in single requests
- Avoids inefficient client-side filtering

## Incorrect Pattern

```python
# WRONG: Fetching all, filtering client-side
import requests

response = requests.get(f"{API_URL}/dcim/devices/", headers=headers)
devices = response.json()["results"]

# Client-side filtering (inefficient)
active_switches = [
    d for d in devices
    if d["status"]["value"] == "active"
    and "switch" in d["name"].lower()
]
```

**Problems with this approach:**
- Downloads all devices (potentially thousands)
- Filtering happens after full transfer
- Wastes bandwidth and memory
- Slower overall

## Correct Pattern

```python
# CORRECT: Server-side filtering with expressions
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

# Filter on server, get only matching results
response = requests.get(
    f"{API_URL}/dcim/devices/?status=active&name__ic=switch",
    headers=headers
)
devices = response.json()["results"]
```

**Benefits:**
- Only matching objects transferred
- Database-optimized queries
- Lower bandwidth usage
- Faster overall response

## Available Lookup Expressions

### String Expressions

| Expression | Description | Example |
|------------|-------------|---------|
| (none) | Exact match | `name=switch-01` |
| `__n` | Not equal | `name__n=test-device` |
| `__ic` | Contains (case-insensitive) | `name__ic=switch` |
| `__nic` | Not contains | `name__nic=test` |
| `__isw` | Starts with | `name__isw=core-` |
| `__nisw` | Not starts with | `name__nisw=temp-` |
| `__iew` | Ends with | `name__iew=-prod` |
| `__niew` | Not ends with | `name__niew=-dev` |
| `__ie` | Exact (case-insensitive) | `name__ie=Switch-01` |
| `__nie` | Not exact (case-insensitive) | `name__nie=Router-01` |
| `__empty` | Is empty | `comments__empty=true` |

### Numeric Expressions

| Expression | Description | Example |
|------------|-------------|---------|
| `__gte` | Greater than or equal | `vlan_id__gte=100` |
| `__gt` | Greater than | `vlan_id__gt=99` |
| `__lte` | Less than or equal | `vlan_id__lte=200` |
| `__lt` | Less than | `vlan_id__lt=201` |

### Null Expressions

| Expression | Description | Example |
|------------|-------------|---------|
| `__isnull` | Is null | `primary_ip4__isnull=false` |

## Common Use Cases

### Find devices by name pattern

```python
# Devices with "core" in the name
response = requests.get(
    f"{API_URL}/dcim/devices/?name__ic=core",
    headers=headers
)

# Devices starting with "sw-"
response = requests.get(
    f"{API_URL}/dcim/devices/?name__isw=sw-",
    headers=headers
)
```

### VLAN range queries

```python
# VLANs between 100 and 200
response = requests.get(
    f"{API_URL}/ipam/vlans/?vid__gte=100&vid__lte=200",
    headers=headers
)
```

### Devices with IP addresses assigned

```python
# Devices that have a primary IPv4
response = requests.get(
    f"{API_URL}/dcim/devices/?primary_ip4__isnull=false",
    headers=headers
)

# Devices without primary IPv4
response = requests.get(
    f"{API_URL}/dcim/devices/?primary_ip4__isnull=true",
    headers=headers
)
```

### Exclude certain values

```python
# All devices NOT in "offline" status
response = requests.get(
    f"{API_URL}/dcim/devices/?status__n=offline",
    headers=headers
)

# Devices not at test sites
response = requests.get(
    f"{API_URL}/dcim/devices/?site__n=test-site-1&site__n=test-site-2",
    headers=headers
)
```

### Multiple values (OR logic)

```python
# Devices that are active OR planned
response = requests.get(
    f"{API_URL}/dcim/devices/?status=active&status=planned",
    headers=headers
)
```

### Combining filters (AND logic)

```python
# Active devices with "core" in name at specific site
response = requests.get(
    f"{API_URL}/dcim/devices/?status=active&name__ic=core&site=nyc-dc1",
    headers=headers
)
```

### Prefix queries

```python
# All prefixes with /24 or smaller
response = requests.get(
    f"{API_URL}/ipam/prefixes/?prefix_length__gte=24",
    headers=headers
)

# Prefixes within a container
response = requests.get(
    f"{API_URL}/ipam/prefixes/?within=10.0.0.0/8",
    headers=headers
)
```

## pynetbox Examples

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Filter with lookup expressions
devices = nb.dcim.devices.filter(
    name__ic="switch",
    status="active"
)

# VLAN range
vlans = nb.ipam.vlans.filter(
    vid__gte=100,
    vid__lte=200
)

# Negation
non_offline = nb.dcim.devices.filter(
    status__n="offline"
)
```

## Exceptions

- **Complex logic:** Some queries may require multiple requests or client-side processing
- **Regex patterns:** NetBox doesn't support regex; use multiple filters

## Related Rules

- [rest-custom-field-filters](./rest-custom-field-filters.md) - Filter by custom fields
- [rest-avoid-search-filter-at-scale](./rest-avoid-search-filter-at-scale.md) - Avoid `q=` at scale
- [graphql-prefer-filters](./graphql-prefer-filters.md) - GraphQL filtering

## References

- [NetBox Filtering](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#filtering)
