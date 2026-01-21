---
title: Avoid Deeply Nested Filters
impact: HIGH
category: graphql
tags: [graphql, filtering, performance, optimization]
netbox_version: "4.5+"
---

# graphql-avoid-nested-filters: Avoid Deeply Nested Filters

## Rationale

NetBox 4.5.1+ added direct filter fields to many object types, allowing you to filter without deeply nesting through related objects. Using these local filters reduces filter depth and improves query performance.

Deeply nested filters create complex SQL queries with multiple JOINs. Flatter filter structures are more efficient.

## Incorrect Pattern

```graphql
# SUBOPTIMAL: Filter depth 3 - traverses device → site
query GetInterfaces {
  interface_list(
    limit: 100
    filters: {
      device: {
        site: {
          name: {exact: "NYC-DC1"}
        }
      }
    }
  ) {
    name
    device { name }
  }
}
```

**Why this is slower:**
- Filter depth of 3 requires multiple JOINs
- Each nesting level adds query complexity
- More expensive to execute

## Correct Pattern

```graphql
# OPTIMAL: Filter depth 2 - uses local site filter (NetBox 4.5.1+)
query GetInterfaces {
  interface_list(
    limit: 100
    filters: {
      site: {name: {exact: "NYC-DC1"}}
    }
  ) {
    name
    device { name }
  }
}
```

**Benefits:**
- Shallower filter path
- Fewer JOINs required
- Better query performance

> **Note:** The `site` filter on `interface_list` was added in NetBox 4.5.1. Check the GraphQL schema for available local filters on each object type.

## Common Local Filters

NetBox has added local filter shortcuts to avoid traversing relationships:

| Object | Instead of | Use |
|--------|-----------|-----|
| Interface | `device.site` | `site` |
| Interface | `device.role` | `device_role` |
| IP Address | `interface.device.site` | `site` |
| Cable | `a_terminations.device.site` | `site` |

Check the GraphQL introspection or NetBox release notes for the complete list.

## Filter Depth Comparison

```graphql
# Depth 4 (avoid)
filters: {
  interface: {
    device: {
      site: {
        region: {name: {exact: "US-East"}}
      }
    }
  }
}

# Depth 2 (preferred)
filters: {
  region: {name: {exact: "US-East"}}
}
```

## Discovering Available Filters

Use GraphQL introspection to find available filter fields:

```graphql
query {
  __type(name: "InterfaceFilter") {
    inputFields {
      name
      type { name }
    }
  }
}
```

Or use the NetBox GraphiQL interface to explore filter options.

## Version Compatibility

| Version | Available Filters |
|---------|-------------------|
| 4.4.x | Limited local filters |
| 4.5.0 | Added some shortcuts |
| 4.5.1+ | Many new local filters |

If targeting older versions, nested filters may be unavoidable. Document the workaround and plan to update when upgrading NetBox.

## Exceptions

- **Older NetBox versions:** Local filters may not be available
- **Complex filter logic:** Some relationships don't have shortcuts
- **One-off queries:** Optimization not worth the effort for rare queries

## Related Rules

- [graphql-filter-by-id](./graphql-filter-by-id.md) - Use IDs to avoid JOINs
- [graphql-prefer-filters](./graphql-prefer-filters.md) - Filter server-side
- [graphql-max-depth](./graphql-max-depth.md) - Keep query depth low

## References

- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
- [NetBox 4.5.1 Release Notes](https://github.com/netbox-community/netbox/releases/tag/v4.5.1)
