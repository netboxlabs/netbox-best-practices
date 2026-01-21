---
title: Understand Nested vs Flat Serializers
impact: LOW
category: rest
tags: [rest, serializers, nested, response]
netbox_version: "4.4+"
---

# rest-nested-serializers: Understand Nested vs Flat Serializers

## Rationale

NetBox responses include nested object representations for related objects. Understanding this structure helps parse responses and construct requests correctly.

## Response Structure

```json
{
  "id": 123,
  "name": "switch-01",
  "site": {
    "id": 1,
    "url": "https://netbox.example.com/api/dcim/sites/1/",
    "display": "NYC-DC1",
    "name": "NYC-DC1",
    "slug": "nyc-dc1"
  },
  "device_type": {
    "id": 1,
    "url": "...",
    "display": "Catalyst 9300",
    "manufacturer": {
      "id": 1,
      "url": "...",
      "display": "Cisco",
      "name": "Cisco",
      "slug": "cisco"
    },
    "model": "Catalyst 9300"
  }
}
```

## Request Structure

When creating/updating, use integer IDs:

```python
# CORRECT: Use IDs for foreign keys
device_data = {
    "name": "switch-01",
    "site": 1,           # Integer ID, not nested object
    "device_type": 1,
    "role": 1
}

response = requests.post(f"{API_URL}/dcim/devices/", headers=headers, json=device_data)
```

## Related Rules

- [rest-brief-mode](./rest-brief-mode.md) - Minimal nested data
- [rest-field-selection](./rest-field-selection.md) - Control nested fields
