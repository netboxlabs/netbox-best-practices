---
title: Filter by Custom Fields
impact: MEDIUM
category: rest
tags: [rest, filtering, custom-fields]
netbox_version: "4.4+"
---

# rest-custom-field-filters: Filter by Custom Fields

## Rationale

Custom fields extend NetBox's data model. Filter by custom fields using the `cf_` prefix to efficiently query organization-specific data.

## Correct Pattern

```python
import requests

# Filter by custom field value
response = requests.get(
    f"{API_URL}/dcim/devices/?cf_environment=production",
    headers=headers
)

# Multiple custom field filters (AND logic)
response = requests.get(
    f"{API_URL}/dcim/devices/?cf_environment=production&cf_tier=1",
    headers=headers
)

# Custom field with lookup expression
response = requests.get(
    f"{API_URL}/dcim/devices/?cf_deployment_date__gte=2024-01-01",
    headers=headers
)
```

## Common Custom Field Types

| Type | Filter Example |
|------|---------------|
| Text | `cf_owner__ic=team` |
| Integer | `cf_priority__gte=3` |
| Boolean | `cf_monitored=true` |
| Date | `cf_expiry__lte=2024-12-31` |
| Selection | `cf_environment=production` |

## Related Rules

- [rest-filtering-expressions](./rest-filtering-expressions.md) - Lookup expressions
- [data-custom-fields](./data-custom-fields.md) - Custom field usage
