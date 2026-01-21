---
title: Limit Query Depth
impact: HIGH
category: graphql
tags: [graphql, depth, nesting, performance]
netbox_version: "4.4+"
---

# graphql-max-depth: Limit Query Depth

## Rationale

Deep nesting in GraphQL queries causes exponential complexity growth. Each nesting level can multiply the number of database queries and objects returned.

**Guidelines:**
- Keep depth at **3 or below** for most queries
- **Never exceed 5** levels of nesting
- Consider REST or multiple queries for deeper data needs

## Incorrect Pattern

```graphql
# WRONG: Excessive nesting depth (5 levels)
query {
  site_list(limit: 10) {           # Level 1
    name
    devices(limit: 50) {            # Level 2
      name
      interfaces(limit: 100) {       # Level 3
        name
        ip_addresses(limit: 10) {    # Level 4
          address
          vrf {                      # Level 5 - TOO DEEP
            name
            route_targets {          # Level 6 - DANGER
              name
            }
          }
        }
      }
    }
  }
}
```

**Problems with this approach:**
- Complexity grows exponentially with depth
- N+1 query patterns at each level
- Difficult to optimize database queries
- Large memory footprint
- Risk of timeouts

## Correct Pattern

```graphql
# CORRECT: Depth limited to 3 levels
query GetSiteDeviceSummary {
  site_list(limit: 10) {           # Level 1
    name
    devices(limit: 20) {            # Level 2
      name
      interface_count              # Scalar, not nested
      primary_ip4 {                # Level 3 (max)
        address
      }
    }
  }
}
```

**Benefits:**
- Predictable performance
- Manageable complexity
- Easier to optimize

## Split Deep Queries

Instead of one deep query, use multiple shallow queries:

```graphql
# Query 1: Get sites and devices (depth: 2)
query GetSitesAndDevices {
  site_list(limit: 10) {
    id
    name
    devices(limit: 20) {
      id
      name
    }
  }
}

# Query 2: Get interfaces for specific device (depth: 2)
query GetDeviceInterfaces($deviceId: Int!) {
  interface_list(device_id: $deviceId, limit: 100) {
    id
    name
    ip_addresses(limit: 10) {
      address
    }
  }
}

# Query 3: Get VRF details if needed (depth: 2)
query GetVRFDetails($vrfId: Int!) {
  vrf(id: $vrfId) {
    name
    description
    import_targets(limit: 10) {
      name
    }
    export_targets(limit: 10) {
      name
    }
  }
}
```

## Depth Counting

Count from the root list/object:

```graphql
query {
  site_list {        # Level 1 (root)
    name
    devices {         # Level 2
      name
      interfaces {     # Level 3
        name
        ip_addresses { # Level 4 (AVOID)
          address
        }
      }
    }
  }
}
```

**Note:** Scalar fields (name, status, etc.) don't add depth. Only nested objects/lists count.

## When Depth is Acceptable

Some shallow depth-4 queries may be acceptable with proper pagination:

```graphql
# Acceptable: depth 4 but well-constrained
query {
  site_list(limit: 5) {
    name
    devices(limit: 10) {
      name
      interfaces(limit: 5) {  # Only 5 interfaces
        name
        ip_addresses(limit: 2) {  # Only 2 IPs
          address
        }
      }
    }
  }
}
# Max objects: 5 × 10 × 5 × 2 = 500 (manageable)
```

## REST Alternative for Deep Data

When you need deeply nested data, REST may be simpler:

```python
import requests

# Fetch devices
devices_resp = requests.get(
    f"{API_URL}/dcim/devices/?site=nyc-dc1&limit=20",
    headers=headers
)
devices = devices_resp.json()["results"]

# For each device, fetch interfaces
for device in devices:
    interfaces_resp = requests.get(
        f"{API_URL}/dcim/interfaces/?device_id={device['id']}&limit=100",
        headers=headers
    )
    device["interfaces"] = interfaces_resp.json()["results"]

    # For each interface, IP addresses are already included
    # Or fetch separately if needed
```

## Query Optimizer Detection

The query optimizer detects depth violations:

```bash
netbox-query-optimizer analyze query.graphql

# Output:
# Issues Found:
# - DEPTH_EXCEEDED: Query depth 5 exceeds recommended max of 3
# - DEPTH_EXCEEDED: Query depth 6 at 'route_targets' is excessive
```

## Exceptions

- **Specific single-object queries:** `device(id: 123)` with nested details is often fine
- **Aggregate/count fields:** These don't add the same complexity as nested lists
- **Well-paginated queries:** Strict limits at every level can make depth-4 acceptable

## Related Rules

- [graphql-pagination-at-each-level](./graphql-pagination-at-each-level.md) - Limit nested counts
- [graphql-use-query-optimizer](./graphql-use-query-optimizer.md) - Detect depth issues
- [graphql-vs-rest-decision](./graphql-vs-rest-decision.md) - When to use REST instead

## References

- [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)
