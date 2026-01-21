---
title: Use OPTIONS for Endpoint Discovery
impact: LOW
category: rest
tags: [rest, discovery, options, schema]
netbox_version: "4.4+"
---

# rest-options-discovery: Use OPTIONS for Endpoint Discovery

## Rationale

The HTTP OPTIONS method returns endpoint schema including available fields, types, and choices. This is useful for dynamic integrations and validation.

## Correct Pattern

```python
import requests

response = requests.options(f"{API_URL}/dcim/devices/", headers=headers)
schema = response.json()

# Discover available fields
for field, meta in schema["actions"]["POST"].items():
    field_type = meta.get("type", "unknown")
    required = meta.get("required", False)
    choices = meta.get("choices", [])
    print(f"{field}: {field_type}, required={required}")
    if choices:
        print(f"  Choices: {[c['value'] for c in choices]}")
```

## Use Cases

- Dynamic form generation
- Runtime field validation
- API documentation tools
- Choice field discovery

## Related Rules

- [rest-error-handling](./rest-error-handling.md) - Handle API responses
