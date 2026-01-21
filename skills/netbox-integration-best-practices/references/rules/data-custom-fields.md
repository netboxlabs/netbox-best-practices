---
title: Use Custom Fields Properly
impact: MEDIUM
category: data
tags: [data-model, custom-fields, extensibility]
netbox_version: "4.4+"
---

# data-custom-fields: Use Custom Fields Properly

## Rationale

Custom fields extend NetBox's data model for organization-specific needs. Proper usage enables consistent data capture and effective querying.

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Create with custom fields
device = nb.dcim.devices.create(
    name="server-01",
    device_type=1,
    role=1,
    site=1,
    custom_fields={
        "environment": "production",
        "cost_center": "IT-001",
        "maintenance_window": "sunday-0200"
    }
)

# Update custom fields
device.custom_fields["environment"] = "staging"
device.save()

# Filter by custom fields
production = nb.dcim.devices.filter(cf_environment="production")
```

## Custom Field Types

| Type | Example Value |
|------|---------------|
| Text | `"production"` |
| Integer | `42` |
| Boolean | `true` |
| Date | `"2024-01-15"` |
| URL | `"https://wiki.example.com/device"` |
| Selection | `"tier-1"` |
| Multi-select | `["web", "database"]` |
| Object | `{"id": 123}` |

## Related Rules

- [rest-custom-field-filters](./rest-custom-field-filters.md) - Filter by custom fields
- [data-tags-usage](./data-tags-usage.md) - Alternative classification
