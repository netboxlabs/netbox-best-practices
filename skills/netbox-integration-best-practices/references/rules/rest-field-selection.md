---
title: Use Field Selection for Specific Fields
impact: HIGH
category: rest
tags: [rest, performance, fields, optimization]
netbox_version: "4.4+"
---

# rest-field-selection: Use Field Selection for Specific Fields

## Rationale

The `?fields=` parameter allows selecting exactly which fields to include in the response. This provides more granular control than brief mode when you need specific fields that brief mode doesn't include.

Benefits:
- Reduce payload to only needed data
- Include specific nested fields
- Better than brief when you need certain non-brief fields

## Incorrect Pattern

```python
# WRONG: Fetching full object for just a few fields
import requests

response = requests.get(
    f"{API_URL}/dcim/devices/",
    headers=headers
)

# Full response with 50+ fields, but we only use 4:
for device in response.json()["results"]:
    print(f"{device['name']}: {device['status']} at {device['site']['name']}")
    if device["primary_ip4"]:
        print(f"  IP: {device['primary_ip4']['address']}")
```

**Problems with this approach:**
- Transferring 50+ fields when only 4 needed
- Larger response size
- More data to parse
- Wasted resources

## Correct Pattern

```python
# CORRECT: Request only needed fields
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

response = requests.get(
    f"{API_URL}/dcim/devices/?fields=id,name,status,site,primary_ip4",
    headers=headers
)

for device in response.json()["results"]:
    print(f"{device['name']}: {device['status']} at {device['site']['name']}")
    if device.get("primary_ip4"):
        print(f"  IP: {device['primary_ip4']['address']}")
```

**Benefits:**
- Response contains exactly what's needed
- Smaller payload
- Faster processing
- Clear intent in code

## Field Selection Syntax

```
?fields=field1,field2,field3
```

**Selecting nested fields:**
```
?fields=name,site.name,device_type.model
```

**Multiple fields from same nested object:**
```
?fields=name,site.name,site.slug,device_type.model,device_type.manufacturer.name
```

## Response Examples

**Full response (no field selection):**
```json
{
    "id": 123,
    "name": "switch-01",
    "status": {"value": "active", "label": "Active"},
    "site": {"id": 1, "url": "...", "display": "NYC-DC1", "name": "NYC-DC1", "slug": "nyc-dc1"},
    "device_type": {"id": 1, "url": "...", ...},
    "role": {"id": 1, "url": "...", ...},
    "tenant": null,
    "platform": null,
    ... // 40+ more fields
}
```

**With field selection (`?fields=id,name,status,site.name`):**
```json
{
    "id": 123,
    "name": "switch-01",
    "status": {"value": "active", "label": "Active"},
    "site": {"name": "NYC-DC1"}
}
```

## Common Field Selection Patterns

**Device inventory list:**
```python
fields = "id,name,status,site.name,rack.name,primary_ip4.address"
response = requests.get(f"{API_URL}/dcim/devices/?fields={fields}", headers=headers)
```

**Interface summary:**
```python
fields = "id,name,type,enabled,device.name"
response = requests.get(f"{API_URL}/dcim/interfaces/?fields={fields}", headers=headers)
```

**Prefix allocation status:**
```python
fields = "id,prefix,status,site.name,vlan.vid,vlan.name"
response = requests.get(f"{API_URL}/ipam/prefixes/?fields={fields}", headers=headers)
```

## Field Selection vs Brief Mode

| Feature | `?fields=` | `?brief=True` |
|---------|-----------|---------------|
| Control | Choose any fields | Fixed minimal set |
| Nested fields | Yes | No |
| Flexibility | High | Low |
| Use case | Specific field needs | Dropdowns/references |

**Use brief mode when:**
- Building dropdowns or select lists
- Only need ID and display name

**Use field selection when:**
- Need specific non-brief fields
- Need nested object fields
- Need custom field combinations

## Combining with Other Parameters

```python
# Field selection with filters and pagination
response = requests.get(
    f"{API_URL}/dcim/devices/"
    f"?fields=id,name,status,site.name"
    f"&status=active"
    f"&site=nyc-dc1"
    f"&limit=100",
    headers=headers
)
```

## Exceptions

- **Unknown field requirements:** May need full object for dynamic processing
- **Cache reuse:** Full objects may be more cache-efficient if reused
- **Audit/logging:** May want complete snapshots

## Related Rules

- [rest-brief-mode](./rest-brief-mode.md) - Simpler minimal response
- [rest-exclude-config-context](./rest-exclude-config-context.md) - Exclude heavy fields
- [graphql-select-only-needed](./graphql-select-only-needed.md) - GraphQL field selection

## References

- [NetBox REST API - Field Selection](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/)
