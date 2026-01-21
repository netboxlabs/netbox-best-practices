---
title: Choose GraphQL vs REST Appropriately
impact: MEDIUM
category: graphql
tags: [graphql, rest, api, architecture, decision]
netbox_version: "4.4+"
---

# graphql-vs-rest-decision: Choose GraphQL vs REST Appropriately

## Rationale

Both REST and GraphQL have strengths. Choosing the right API for each use case improves performance, simplifies code, and reduces errors.

There's no universal "better" choice - it depends on the specific task.

## Decision Matrix

| Use Case | Recommended | Reasoning |
|----------|-------------|-----------|
| Single object by ID | REST | Simpler, cacheable, one request |
| Simple list with filters | REST | Well-optimized, pagination built-in |
| Multiple related object types | GraphQL | Single request vs multiple |
| Deeply nested data | Either carefully | May need multiple queries anyway |
| Dashboard/real-time data | REST | Easier HTTP caching |
| Flexible field selection | GraphQL | Dynamic field selection |
| Bulk create/update/delete | REST | Native bulk operation support |
| CI/CD scripts | REST | Simpler shell scripting |
| Complex reporting | GraphQL | Flexible queries, single request |
| Webhook-driven updates | REST | Simpler integration |

## When to Use REST

### Single Object Operations

```python
# REST is simpler for single object fetch
response = requests.get(
    f"{API_URL}/dcim/devices/123/",
    headers=headers
)
device = response.json()
```

vs GraphQL:
```graphql
query {
  device(id: 123) {
    name
    status
    # Must specify all fields
  }
}
```

### Simple Filtered Lists

```python
# REST with filters is straightforward
response = requests.get(
    f"{API_URL}/dcim/devices/?site=nyc-dc1&status=active&limit=100",
    headers=headers
)
```

### Bulk Operations

```python
# REST has native bulk support
devices = [
    {"name": "sw-01", "device_type": 1, "role": 1, "site": 1},
    {"name": "sw-02", "device_type": 1, "role": 1, "site": 1},
]
response = requests.post(f"{API_URL}/dcim/devices/", headers=headers, json=devices)
```

GraphQL is read-only in NetBox - bulk writes require REST.

### CI/CD and Scripting

```bash
# REST works well with curl in scripts
curl -H "Authorization: Bearer $TOKEN" \
  "https://netbox.example.com/api/dcim/devices/?name=switch-01"
```

## When to Use GraphQL

### Fetching Related Data

```graphql
# Single request for device with related objects
query {
  device(id: 123) {
    name
    status
    site {
      name
      region { name }
    }
    device_type {
      model
      manufacturer { name }
    }
    interfaces(limit: 10) {
      name
      ip_addresses(limit: 5) {
        address
      }
    }
  }
}
```

vs REST (multiple requests):
```python
device = requests.get(f"{API_URL}/dcim/devices/123/").json()
site = requests.get(device["site"]["url"]).json()
device_type = requests.get(device["device_type"]["url"]).json()
interfaces = requests.get(f"{API_URL}/dcim/interfaces/?device_id=123").json()
# ... more requests for nested data
```

### Custom Field Selection

```graphql
# Request exactly the fields needed
query {
  device_list(limit: 100) {
    name
    primary_ip4 { address }
    # Only 2 fields vs full device object
  }
}
```

### Cross-Object Queries

```graphql
# Fetch different object types in one request
query DashboardData {
  devices: device_list(limit: 1) { id }
  sites: site_list(limit: 1) { id }
  prefixes: prefix_list(limit: 1) { id }
}
```

## Hybrid Approaches

Combine both APIs based on the operation:

```python
import pynetbox
import requests

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Use REST for writes
device = nb.dcim.devices.create(
    name="new-switch",
    device_type=1,
    role=1,
    site=1
)

# Use GraphQL for complex reads
query = """
query GetDeviceWithRelations($id: Int!) {
  device(id: $id) {
    name
    site { name region { name } }
    interfaces(limit: 50) {
      name
      ip_addresses(limit: 5) { address }
    }
  }
}
"""
data = graphql_query(NETBOX_URL, TOKEN, query, {"id": device.id})
```

## Performance Considerations

| Aspect | REST | GraphQL |
|--------|------|---------|
| Caching | HTTP caching works well | Harder to cache (POST only) |
| N+1 queries | May need multiple requests | Can fetch in one (but may cause N+1 on server) |
| Over-fetching | May get extra data | Request exact fields |
| Under-fetching | May need extra requests | One request for related data |
| Complexity | Simple per request | Complexity analysis needed |

## Exceptions

- **Team familiarity:** Use what the team knows well
- **Existing patterns:** Maintain consistency in a codebase
- **Third-party integrations:** Some tools only support REST

## Related Rules

- [graphql-use-query-optimizer](./graphql-use-query-optimizer.md) - Analyze GraphQL complexity
- [rest-pagination-required](./rest-pagination-required.md) - REST patterns
- [integ-pynetbox-client](./integ-pynetbox-client.md) - Python REST client

## References

- [NetBox REST API](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/)
- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
