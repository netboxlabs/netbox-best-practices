---
title: Understand IPAM Hierarchy
impact: MEDIUM
category: data
tags: [data-model, hierarchy, ipam, prefixes]
netbox_version: "4.4+"
---

# data-ipam-hierarchy: Understand IPAM Hierarchy

## Rationale

NetBox's IPAM hierarchy (RIRs → Aggregates → Prefixes → IP Addresses) enables structured IP address management. Understanding relationships enables proper allocation and querying.

## Hierarchy Structure

```
RIR (Regional Internet Registry)
└── Aggregate (assigned block)
    └── Prefix (routable network, can be hierarchical)
        ├── IP Range (allocation range)
        └── IP Address (individual address)

VRF (optional, scopes prefixes/IPs)
└── Prefixes/IP Addresses within VRF
```

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Create RIR and aggregate
rir = nb.ipam.rirs.create(name="ARIN", slug="arin")
aggregate = nb.ipam.aggregates.create(
    prefix="10.0.0.0/8",
    rir=rir.id,
    description="Private network"
)

# Create prefix hierarchy
parent_prefix = nb.ipam.prefixes.create(
    prefix="10.0.0.0/16",
    status="container"
)
child_prefix = nb.ipam.prefixes.create(
    prefix="10.0.1.0/24",
    status="active"
)

# Create IP address
ip = nb.ipam.ip_addresses.create(
    address="10.0.1.1/24",
    status="active"
)
```

## Querying IPAM

```python
# Find prefixes within a container
children = nb.ipam.prefixes.filter(within="10.0.0.0/16")

# Find available IPs in a prefix
available = nb.ipam.prefixes.get(prefix="10.0.1.0/24").available_ips.list()

# Find IPs in a VRF
vrf_ips = nb.ipam.ip_addresses.filter(vrf="production")
```

## Related Rules

- [data-dependency-order](./data-dependency-order.md) - Creation order
- [data-site-hierarchy](./data-site-hierarchy.md) - Site structure
