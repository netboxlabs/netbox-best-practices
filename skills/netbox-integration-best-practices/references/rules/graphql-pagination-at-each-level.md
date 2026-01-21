---
title: Paginate Nested Lists at Every Level
impact: HIGH
category: graphql
tags: [graphql, pagination, nesting, performance]
netbox_version: "4.4+"
---

# graphql-pagination-at-each-level: Paginate Nested Lists at Every Level

## Rationale

In GraphQL, pagination isn't just for the top-level query. Every nested list that returns multiple objects must also be paginated. Without nested pagination:
- Each parent can return unbounded children
- Object counts multiply (fan-out pattern)
- 10 sites × 100 devices × 50 interfaces = 50,000 objects

This multiplicative growth can overwhelm both server and client.

## Incorrect Pattern

```graphql
# WRONG: Only top level paginated, nested lists unbounded
query {
  site_list(limit: 10) {
    name
    devices {  # UNBOUNDED - could be hundreds per site
      name
      interfaces {  # UNBOUNDED - could be hundreds per device
        name
        ip_addresses {  # UNBOUNDED - multiple per interface
          address
        }
      }
    }
  }
}
```

**Problems with this approach:**
- 10 sites with 100 devices each = 1,000 devices
- 1,000 devices with 50 interfaces each = 50,000 interfaces
- 50,000 interfaces with 2 IPs each = 100,000 IP addresses
- Total: 151,010 objects from what looks like a "10 sites" query

## Correct Pattern

```graphql
# CORRECT: Every list has pagination limits
query GetSiteOverview {
  site_list(limit: 10) {
    name
    devices(limit: 20) {  # Limit devices per site
      name
      interfaces(limit: 50) {  # Limit interfaces per device
        name
        ip_addresses(limit: 5) {  # Limit IPs per interface
          address
        }
      }
    }
  }
}
```

**With proper limits:**
- 10 sites × 20 devices × 50 interfaces × 5 IPs = 50,000 max
- Still potentially large, but bounded and predictable

## Better: Use Multiple Targeted Queries

```graphql
# Query 1: Get sites with device counts
query GetSites {
  site_list(limit: 10) {
    id
    name
    device_count
  }
}

# Query 2: Get devices for a specific site
query GetSiteDevices($siteId: Int!) {
  device_list(site_id: $siteId, limit: 50) {
    id
    name
    interface_count
  }
}

# Query 3: Get interfaces for specific devices
query GetDeviceInterfaces($deviceId: Int!) {
  interface_list(device_id: $deviceId, limit: 100) {
    name
    ip_addresses(limit: 10) {
      address
    }
  }
}
```

**Benefits:**
- Each query is bounded
- Fetch only what you need when you need it
- Better for UI (fetch on demand)
- Easier to cache

## Implementing Nested Pagination in Code

```python
import requests

def fetch_site_with_devices(site_id, netbox_url, token, devices_limit=50):
    """Fetch site with paginated devices."""
    query = """
    query GetSiteDevices($siteId: Int!, $limit: Int!, $offset: Int!) {
      site(id: $siteId) {
        name
        devices(limit: $limit, offset: $offset) {
          id
          name
          status
        }
      }
    }
    """

    all_devices = []
    offset = 0

    while True:
        response = requests.post(
            f"{netbox_url}/graphql/",
            headers={
                "Authorization": f"Bearer {token}",
                "Content-Type": "application/json"
            },
            json={
                "query": query,
                "variables": {
                    "siteId": site_id,
                    "limit": devices_limit,
                    "offset": offset
                }
            }
        )

        data = response.json()["data"]["site"]
        devices = data["devices"]

        if not devices:
            break

        all_devices.extend(devices)

        if len(devices) < devices_limit:
            break

        offset += devices_limit

    return {
        "name": data["name"],
        "devices": all_devices
    }
```

## Query Optimizer Detection

The [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer) detects unbounded nested lists:

```bash
netbox-query-optimizer analyze query.graphql

# Output:
# Issues Found:
# - UNBOUNDED_LIST at line 4: 'devices' has no limit parameter
# - UNBOUNDED_LIST at line 6: 'interfaces' has no limit parameter
```

## Choosing Nested Limits

Consider the data distribution in your NetBox:

| Relationship | Typical Count | Suggested Limit |
|--------------|---------------|-----------------|
| Site → Devices | 10-500 | 50-100 |
| Device → Interfaces | 4-200 | 50-100 |
| Interface → IP Addresses | 1-5 | 10 |
| Prefix → Child Prefixes | 1-100 | 50 |
| VRF → Prefixes | 10-1000 | 100 |

## Exceptions

- **Single object:** `site(id: 123)` doesn't need pagination
- **Known small lists:** `tags` on an object (typically <10)
- **Scalar fields:** Non-list fields don't need pagination

## Related Rules

- [graphql-always-paginate](./graphql-always-paginate.md) - Top-level pagination
- [graphql-use-query-optimizer](./graphql-use-query-optimizer.md) - Detect issues
- [graphql-max-depth](./graphql-max-depth.md) - Limit nesting depth

## References

- [GraphQL Pagination Best Practices](https://graphql.org/learn/pagination/)
- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)
