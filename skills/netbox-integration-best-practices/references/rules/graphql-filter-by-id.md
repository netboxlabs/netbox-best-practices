---
title: Filter by ID to Avoid Joins
impact: HIGH
category: graphql
tags: [graphql, filtering, performance, optimization]
netbox_version: "4.4+"
---

# graphql-filter-by-id: Filter by ID to Avoid Joins

## Rationale

When filtering on related objects, use numeric IDs instead of name-based lookups where possible. Filtering by ID uses the local foreign key column directly, avoiding a SQL JOIN to the related table.

This can significantly improve query performance, especially for queries that filter on multiple related objects.

## Incorrect Pattern

```graphql
# SUBOPTIMAL: Filtering by site name requires JOIN to sites table
query GetDevices {
  device_list(
    limit: 100
    filters: {site: {name: {exact: "NYC-DC1"}}}
  ) {
    name
    status
  }
}
```

**Why this is slower:**
- Database must JOIN the sites table
- Then filter on the name column
- More complex query execution plan

## Correct Pattern

```graphql
# OPTIMAL: Filtering by site ID uses local column directly
query GetDevices($siteId: ID!) {
  device_list(
    limit: 100
    filters: {site_id: $siteId}
  ) {
    name
    status
  }
}
```

**Benefits:**
- No JOIN required - filters on device table's `site_id` column
- Database uses index on foreign key
- Simpler, faster query execution

## Practical Workflow

In practice, you may need to look up the ID first:

```python
# First request: get the site ID (fast, single object)
site_query = """
query GetSite($name: String!) {
  site_list(limit: 1, filters: {name: {exact: $name}}) {
    id
  }
}
"""

# Second request: filter devices by ID (fast, no JOIN)
devices_query = """
query GetDevices($siteId: ID!) {
  device_list(limit: 100, filters: {site_id: $siteId}) {
    name
    status
  }
}
"""
```

For repeated queries, cache the ID mapping to avoid the lookup overhead.

## When Name Filtering is Acceptable

- **One-off queries:** Development, debugging, ad-hoc analysis
- **Small datasets:** Performance difference is negligible
- **Dynamic input:** User provides name in search UI

## Multiple Filter Example

```graphql
# SUBOPTIMAL: Multiple JOINs
query {
  device_list(
    filters: {
      site: {name: {exact: "NYC-DC1"}}
      role: {name: {exact: "Access Switch"}}
      manufacturer: {name: {exact: "Cisco"}}
    }
  ) {
    name
  }
}

# OPTIMAL: Direct ID filters
query($siteId: ID!, $roleId: ID!, $mfrId: ID!) {
  device_list(
    filters: {
      site_id: $siteId
      role_id: $roleId
      manufacturer_id: $mfrId
    }
  ) {
    name
  }
}
```

## Exceptions

- **User-facing queries:** Names are more user-friendly for input
- **Infrequent queries:** Optimization overhead not worth it
- **Unknown IDs:** When you only have the name available

## Related Rules

- [graphql-prefer-filters](./graphql-prefer-filters.md) - Filter server-side
- [graphql-avoid-nested-filters](./graphql-avoid-nested-filters.md) - Flatten filter paths
- [graphql-select-only-needed](./graphql-select-only-needed.md) - Minimize response data

## References

- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
