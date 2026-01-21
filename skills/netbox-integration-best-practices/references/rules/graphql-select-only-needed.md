---
title: Request Only Needed Fields
impact: HIGH
category: graphql
tags: [graphql, performance, fields, optimization]
netbox_version: "4.4+"
---

# graphql-select-only-needed: Request Only Needed Fields

## Rationale

GraphQL's power is selecting exactly the fields you need. Over-fetching wastes:
- Database query time (joining unnecessary tables)
- Serialization time
- Network bandwidth
- Client parsing time

Only request fields your application actually uses.

## Incorrect Pattern

```graphql
# WRONG: Requesting all available fields "just in case"
query {
  device_list(limit: 100) {
    id
    url
    display
    name
    device_type {
      id
      url
      display
      manufacturer {
        id
        url
        display
        name
        slug
        description
      }
      model
      slug
      part_number
      u_height
      is_full_depth
      subdevice_role
      airflow
    }
    role {
      id
      url
      display
      name
      slug
      color
      vm_role
      description
    }
    tenant {
      id
      url
      display
      name
      slug
    }
    platform {
      id
      url
      display
      name
      slug
    }
    serial
    asset_tag
    site {
      id
      url
      display
      name
      slug
      status
      region {
        id
        url
        display
        name
        slug
      }
    }
    # ... 20+ more fields
  }
}
```

**Problems with this approach:**
- Fetches data never used
- Joins many related tables unnecessarily
- Large response payload
- Slower query execution

## Correct Pattern

```graphql
# CORRECT: Request only what you need
query GetDeviceList {
  device_list(limit: 100) {
    name
    status
    primary_ip4 {
      address
    }
  }
}
```

**Benefits:**
- Minimal data transfer
- Faster query execution
- Smaller response to parse
- Clear intent of data usage

## Determine Required Fields

Before writing a query, ask:
1. What will the UI/code actually display or process?
2. Which fields are used in logic/filtering?
3. Are nested fields really needed, or just IDs?

### Example Analysis

**Task:** Display a device table with name, status, and IP

**Required fields:**
- `name` - display in table
- `status` - display in table
- `primary_ip4.address` - display in table

**Not required:**
- `id` - unless linking to detail pages
- `device_type.manufacturer.description` - not displayed
- `site.region.slug` - not displayed

## Progressive Enhancement

Start minimal, add fields as needed:

```graphql
# Version 1: Basic list
query GetDevices {
  device_list(limit: 100) {
    name
    status
  }
}

# Version 2: Added IP after realizing we need it
query GetDevices {
  device_list(limit: 100) {
    name
    status
    primary_ip4 {
      address
    }
  }
}

# Version 3: Added site for grouping feature
query GetDevices {
  device_list(limit: 100) {
    name
    status
    primary_ip4 {
      address
    }
    site {
      name
    }
  }
}
```

## Field Selection for Different Use Cases

### Dashboard Widget (Minimal)

```graphql
query DashboardDeviceCounts {
  device_list(limit: 1) {
    id
  }
  active: device_list(filters: {status: "active"}, limit: 1) {
    id
  }
}
```

### Device List Table

```graphql
query DeviceTable {
  device_list(limit: 50) {
    id           # For row key/linking
    name         # Display column
    status       # Display column
    site {
      name       # Display column
    }
    primary_ip4 {
      address    # Display column
    }
  }
}
```

### Device Detail View

```graphql
query DeviceDetail($id: Int!) {
  device(id: $id) {
    name
    status
    serial
    asset_tag
    device_type {
      model
      manufacturer {
        name
      }
    }
    site {
      name
    }
    rack {
      name
    }
    position
    comments
  }
}
```

## Complexity Impact

Field selection affects query complexity scores:

```graphql
# Score: ~50 (good)
query {
  device_list(limit: 100) {
    name
    status
  }
}

# Score: ~250 (higher due to nested objects)
query {
  device_list(limit: 100) {
    name
    status
    device_type {
      model
      manufacturer {
        name
      }
    }
    site {
      name
      region {
        name
      }
    }
  }
}
```

## Exceptions

- **Caching:** May fetch extra fields if query results are cached and reused
- **Unknown requirements:** During prototyping, over-fetching may be acceptable
- **GraphQL fragments:** Shared fragments may include more fields than one use case needs

## Related Rules

- [graphql-always-paginate](./graphql-always-paginate.md) - Limit result counts
- [graphql-max-depth](./graphql-max-depth.md) - Limit nesting
- [rest-field-selection](./rest-field-selection.md) - REST equivalent

## References

- [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
