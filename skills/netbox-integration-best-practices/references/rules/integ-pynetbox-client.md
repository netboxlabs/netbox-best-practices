---
title: Use pynetbox for Python Integrations
impact: HIGH
category: integ
tags: [integration, python, pynetbox, client]
netbox_version: "4.4+"
---

# integ-pynetbox-client: Use pynetbox for Python Integrations

## Rationale

pynetbox is the official Python client for NetBox. It provides a Pythonic interface, automatic pagination, and handles many details for you.

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api(
    url="https://netbox.example.com",
    token="nbt_abc123.xxxxx"
)

# Query patterns
all_devices = nb.dcim.devices.all()  # Iterator with auto-pagination
filtered = nb.dcim.devices.filter(site="nyc-dc1", status="active")
single = nb.dcim.devices.get(name="switch-01")

# Create
new_device = nb.dcim.devices.create(
    name="new-switch",
    device_type=1,
    role=1,
    site=1
)

# Update
device = nb.dcim.devices.get(name="switch-01")
device.status = "planned"
device.save()

# Delete
device.delete()

# Bulk create
devices = [{"name": f"sw-{i}", "device_type": 1, "role": 1, "site": 1} for i in range(10)]
created = nb.dcim.devices.create(devices)
```

## Installation

```bash
pip install pynetbox
```

## Related Rules

- [auth-use-v2-tokens](./auth-use-v2-tokens.md) - Token format
- [integ-retry-strategies](./integ-retry-strategies.md) - Error handling

## References

- [pynetbox Documentation](https://pynetbox.readthedocs.io/)
- [pynetbox GitHub](https://github.com/netbox-community/pynetbox)
