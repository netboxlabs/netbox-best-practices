---
title: Filter Server-Side
impact: MEDIUM
category: graphql
tags: [graphql, filtering, performance, queries]
netbox_version: "4.4+"
---

# graphql-prefer-filters: Filter Server-Side

## Rationale

Always filter data in the GraphQL query rather than fetching everything and filtering client-side. Server-side filtering:
- Uses database indexes for efficient querying
- Reduces data transfer
- Lowers memory usage on both client and server
- Improves response times

## Incorrect Pattern

```graphql
# WRONG: Fetch all, filter client-side
query GetAllDevices {
  device_list(limit: 1000) {
    name
    status
    site {
      name
    }
  }
}
```

```python
# Client-side filtering (inefficient)
response = graphql_query(netbox_url, token, query)
devices = response["device_list"]

# Filter after fetching 1000 devices
active_nyc_devices = [
    d for d in devices
    if d["status"] == "active" and d["site"]["name"] == "NYC-DC1"
]
```

**Problems with this approach:**
- Transfers 1000 devices when only 50 are needed
- Database doesn't use indexes for filtering
- Wasted bandwidth and processing
- Slower overall response

## Correct Pattern

```graphql
# CORRECT: Filter in the query
query GetActiveNYCDevices {
  device_list(
    limit: 100
    filters: {
      status: "active"
      site: "nyc-dc1"
    }
  ) {
    name
    status
    site {
      name
    }
  }
}
```

```python
# Direct result - no client filtering needed
response = graphql_query(netbox_url, token, query)
active_nyc_devices = response["device_list"]  # Already filtered
```

**Benefits:**
- Only matching records transferred
- Database query optimized with indexes
- Smaller response payload
- Faster overall

## Available Filter Parameters

Most list queries support filter parameters:

```graphql
query {
  device_list(
    limit: 100
    filters: {
      status: "active"
      site: "nyc-dc1"
      role: "access-switch"
      has_primary_ip: true
    }
  ) {
    name
  }
}
```

Common filters:
- `status` - Object status
- `site` - Site name or slug
- `role` - Device role
- `tenant` - Tenant name
- `tag` - Tag name

## Nested Filtering

Filter related objects within the query:

```graphql
# Filter devices at site, and filter interfaces on those devices
query {
  device_list(limit: 50, filters: {site: "nyc-dc1"}) {
    name
    interfaces(filters: {enabled: true}) {
      name
      type
    }
  }
}
```

## Combining Filters

Multiple filters are combined with AND logic:

```graphql
query {
  device_list(
    filters: {
      status: "active"     # AND
      site: "nyc-dc1"       # AND
      role: "switch"        # AND
    }
    limit: 100
  ) {
    name
  }
}
# Returns devices that are active AND at nyc-dc1 AND have switch role
```

## When Client-Side Filtering is Acceptable

- **Complex logic:** Filter conditions not expressible in GraphQL
- **Cached data:** Filtering already-fetched cached results
- **Multiple filter passes:** Different views of same data

But even then, apply server-side filters first to minimize data:

```graphql
# Get active devices (server filter)
query {
  device_list(limit: 500, filters: {status: "active"}) {
    name
    device_type { model }
    custom_fields
  }
}
```

```python
# Then apply complex client-side logic
devices = response["device_list"]
filtered = [
    d for d in devices
    if d["custom_fields"].get("tier") == 1
    and "9300" in d["device_type"]["model"]
]
```

## Performance Impact

| Approach | 10,000 devices, 50 match filter |
|----------|--------------------------------|
| Fetch all, filter client | Transfer: 10,000, Time: 5-10s |
| Filter server-side | Transfer: 50, Time: 0.2s |

## Exceptions

- **Filters not available:** Some complex conditions can't be expressed
- **Aggregation needed:** May need full data for calculations
- **Comparison across results:** Cross-record logic requires all data

## Related Rules

- [rest-filtering-expressions](./rest-filtering-expressions.md) - REST filtering
- [graphql-select-only-needed](./graphql-select-only-needed.md) - Field selection
- [rest-avoid-search-filter-at-scale](./rest-avoid-search-filter-at-scale.md) - Search vs filters

## References

- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
