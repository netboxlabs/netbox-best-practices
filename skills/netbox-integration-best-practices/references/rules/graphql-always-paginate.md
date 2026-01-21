---
title: Always Paginate GraphQL List Queries
impact: CRITICAL
category: graphql
tags: [graphql, pagination, performance]
netbox_version: "4.4+"
---

# graphql-always-paginate: Always Paginate GraphQL List Queries

## Rationale

Unbounded GraphQL queries can return thousands or millions of objects, causing:
- Server memory exhaustion
- Request timeouts
- Client application crashes
- Database performance degradation

Every list query MUST include explicit pagination limits.

## Incorrect Pattern

```graphql
# WRONG: No pagination - could return entire database
query {
  device_list {
    name
    status
    site {
      name
    }
  }
}
```

**Problems with this approach:**
- No limit on returned objects
- Could return 10,000+ devices
- Server must serialize entire result set
- Client must process/store all results
- Network transfer of potentially megabytes of data

## Correct Pattern

```graphql
# CORRECT: Explicit pagination with limit and offset
query GetDevices($limit: Int!, $offset: Int!) {
  device_list(limit: $limit, offset: $offset) {
    name
    status
    site {
      name
    }
  }
}
```

**Variables:**
```json
{
  "limit": 100,
  "offset": 0
}
```

**Benefits:**
- Predictable response size
- Controlled memory usage
- Faster query execution
- Enables incremental data loading

## Pagination Patterns

### Basic Pagination

```graphql
query GetDevicesPage($limit: Int!, $offset: Int!) {
  device_list(limit: $limit, offset: $offset) {
    id
    name
    status
  }
}
```

### With Total Count

```graphql
query GetDevicesWithCount($limit: Int!, $offset: Int!) {
  device_list(limit: $limit, offset: $offset) {
    id
    name
  }
  # Get total count for pagination UI
  device_count: device_list {
    id
  }
}
```

Note: Getting count requires a separate query or including minimal fields.

### Full Pagination Implementation (Python)

```python
import requests

def fetch_all_devices_graphql(netbox_url, token, page_size=100):
    """Fetch all devices with proper pagination."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    query = """
    query GetDevices($limit: Int!, $offset: Int!) {
      device_list(limit: $limit, offset: $offset) {
        id
        name
        status
        site {
          name
        }
      }
    }
    """

    all_devices = []
    offset = 0

    while True:
        response = requests.post(
            f"{netbox_url}/graphql/",
            headers=headers,
            json={
                "query": query,
                "variables": {"limit": page_size, "offset": offset}
            }
        )
        response.raise_for_status()

        data = response.json()
        if "errors" in data:
            raise Exception(f"GraphQL errors: {data['errors']}")

        devices = data["data"]["device_list"]

        if not devices:
            break

        all_devices.extend(devices)

        if len(devices) < page_size:
            break

        offset += page_size

    return all_devices
```

## Recommended Page Sizes

| Use Case | Page Size |
|----------|-----------|
| Interactive UI | 25-50 |
| Background sync | 100-250 |
| Bulk export | 500 |
| Maximum (not recommended) | 1000 |

## Offset Pagination Limitations (Large Datasets)

NetBox's GraphQL API uses **offset-based pagination** (`limit` + `offset`), which has significant performance implications for large datasets.

**The Problem:** Offset pagination requires the database to scan all rows up to the offset position before returning results. As you paginate deeper into results, queries become progressively slower:

| Page | Offset | Performance Impact |
|------|--------|-------------------|
| 1 | 0 | Fast |
| 10 | 900 | Noticeable delay |
| 100 | 9,900 | Slow |
| 1000 | 99,900 | Very slow / timeout risk |

**Version-Specific Pagination Options:**

| NetBox Version | Pagination Method | Notes |
|----------------|-------------------|-------|
| 4.4.x | Offset only | `limit` + `offset` parameters |
| 4.5.x | Offset + ID range emulation | Filter by ID ranges as workaround |
| 4.6.0+ (planned) | Cursor-based | `start` + `limit` parameters |

### Workaround for 4.5.x: ID Range Filtering

In NetBox 4.5.x, you can emulate cursor-based pagination by filtering on ID ranges:

```graphql
# Instead of offset pagination for deep pages
query GetDevicesPage($minId: Int!, $limit: Int!) {
  device_list(
    limit: $limit
    filters: { id__gte: $minId }
  ) {
    id
    name
    status
  }
}
```

```python
def fetch_all_with_id_cursor(netbox_url, token, page_size=100):
    """Fetch all devices using ID-based cursor pagination (4.5.x)."""
    query = """
    query GetDevices($minId: Int!, $limit: Int!) {
      device_list(limit: $limit, filters: { id__gte: $minId }) {
        id
        name
        status
      }
    }
    """

    all_devices = []
    min_id = 0

    while True:
        response = graphql_request(
            netbox_url, token, query,
            {"minId": min_id, "limit": page_size}
        )
        devices = response["data"]["device_list"]

        if not devices:
            break

        all_devices.extend(devices)

        if len(devices) < page_size:
            break

        # Next page starts after the highest ID we received
        min_id = max(d["id"] for d in devices) + 1

    return all_devices
```

**Drawbacks of ID range emulation:**
- Inconsistent page sizes if IDs have gaps (deleted objects)
- Requires fetching `id` field in every query
- No built-in way to know total count or "last page"

### Future: Cursor-Based Pagination (4.6.0+)

NetBox 4.6.0 plans to introduce proper cursor-based pagination with a `start` parameter:

```graphql
# Planned syntax for 4.6.0+
query GetDevices {
  device_list(start: 1000, limit: 100) {
    id
    name
  }
}
```

This will use efficient primary key filtering (`pk >= start`) instead of offset scanning, providing consistent performance regardless of pagination depth.

See [GitHub Issue #21110](https://github.com/netbox-community/netbox/issues/21110) for implementation details and status.

## Exceptions

- **Single object queries:** `device(id: 123)` doesn't need pagination
- **Count-only queries:** Just counting objects doesn't return full data

But ANY query returning a list MUST be paginated.

## Related Rules

- [graphql-use-query-optimizer](./graphql-use-query-optimizer.md) - Analyzer detects unbounded queries
- [graphql-pagination-at-each-level](./graphql-pagination-at-each-level.md) - Paginate nested lists
- [rest-pagination-required](./rest-pagination-required.md) - REST pagination

## References

- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
