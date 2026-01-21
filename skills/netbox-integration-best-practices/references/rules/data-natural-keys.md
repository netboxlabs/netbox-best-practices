---
title: Use Natural Keys for Human-Readable Queries
impact: MEDIUM
category: data
tags: [data-model, natural-keys, querying]
netbox_version: "4.4+"
---

# data-natural-keys: Use Natural Keys for Human-Readable Queries

## Rationale

Natural keys (name, slug) provide human-readable identification without requiring numeric database IDs. Use them for more maintainable and readable code.

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Query by natural key instead of ID
device = nb.dcim.devices.get(name="switch-01")
site = nb.dcim.sites.get(slug="nyc-dc1")

# Filter by related object natural keys
devices = nb.dcim.devices.filter(
    site="nyc-dc1",           # By slug
    role="access-switch",      # By slug
    status="active"            # By value
)

# Multiple fields for unique identification
interface = nb.dcim.interfaces.get(device="switch-01", name="Gi0/1")
```

## Natural Key Fields

| Object Type | Natural Key |
|-------------|-------------|
| Site | name, slug |
| Device | name |
| Device Type | model (+ manufacturer) |
| Prefix | prefix |
| VLAN | vid (+ group) |

## Related Rules

- [data-dependency-order](./data-dependency-order.md) - Object relationships
- [rest-filtering-expressions](./rest-filtering-expressions.md) - Query patterns
