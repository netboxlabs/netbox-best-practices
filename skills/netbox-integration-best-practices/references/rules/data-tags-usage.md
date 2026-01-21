---
title: Use Tags for Cross-Cutting Classification
impact: MEDIUM
category: data
tags: [data-model, tags, classification]
netbox_version: "4.4+"
---

# data-tags-usage: Use Tags for Cross-Cutting Classification

## Rationale

Tags provide flexible, cross-object-type classification. Unlike custom fields (object-type specific), tags can be applied to any taggable object, enabling queries across different resources.

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Create a tag
tag = nb.extras.tags.create(
    name="PCI-Compliant",
    slug="pci-compliant",
    color="ff0000"
)

# Apply to various object types
device = nb.dcim.devices.get(name="server-01")
device.tags = [{"name": "PCI-Compliant"}]
device.save()

prefix = nb.ipam.prefixes.get(prefix="10.0.1.0/24")
prefix.tags = [{"name": "PCI-Compliant"}]
prefix.save()

# Query across object types
pci_devices = nb.dcim.devices.filter(tag="pci-compliant")
pci_prefixes = nb.ipam.prefixes.filter(tag="pci-compliant")
pci_vlans = nb.ipam.vlans.filter(tag="pci-compliant")
```

## Tags vs Custom Fields

| Feature | Tags | Custom Fields |
|---------|------|---------------|
| Cross-object | Yes | No (type-specific) |
| Multiple values | Yes | Some types |
| Structured data | No | Yes |
| Hierarchy | No | No |

## Related Rules

- [data-custom-fields](./data-custom-fields.md) - Structured data
- [rest-filtering-expressions](./rest-filtering-expressions.md) - Tag filtering
