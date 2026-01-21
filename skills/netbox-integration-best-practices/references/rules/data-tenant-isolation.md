---
title: Use Tenants for Logical Resource Separation
impact: MEDIUM
category: data
tags: [data-model, tenants, multi-tenancy, isolation]
netbox_version: "4.4+"
---

# data-tenant-isolation: Use Tenants for Logical Resource Separation

## Rationale

Tenants enable logical separation of resources for multi-tenant environments, customer segregation, or organizational boundaries. Resources can be assigned to tenants for filtering and reporting.

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Create tenant structure
tenant_group = nb.tenancy.tenant_groups.create(
    name="Customers",
    slug="customers"
)
tenant = nb.tenancy.tenants.create(
    name="ACME Corp",
    slug="acme-corp",
    group=tenant_group.id
)

# Assign resources to tenant
prefix = nb.ipam.prefixes.create(
    prefix="10.100.0.0/24",
    tenant=tenant.id,
    status="active"
)

device = nb.dcim.devices.create(
    name="acme-fw01",
    device_type=1,
    role=1,
    site=1,
    tenant=tenant.id
)

# Query by tenant
acme_resources = {
    "devices": list(nb.dcim.devices.filter(tenant="acme-corp")),
    "prefixes": list(nb.ipam.prefixes.filter(tenant="acme-corp")),
    "vlans": list(nb.ipam.vlans.filter(tenant="acme-corp")),
}
```

## Related Rules

- [data-site-hierarchy](./data-site-hierarchy.md) - Organizational structure
- [data-dependency-order](./data-dependency-order.md) - Creation order
