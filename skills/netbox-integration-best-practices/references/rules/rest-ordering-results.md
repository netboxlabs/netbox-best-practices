---
title: Use Ordering Parameter for Sorted Results
impact: LOW
category: rest
tags: [rest, ordering, sorting, queries]
netbox_version: "4.4+"
---

# rest-ordering-results: Use Ordering Parameter for Sorted Results

## Rationale

Use the `?ordering=` parameter to get sorted results from the server rather than sorting client-side. This is more efficient and consistent.

## Correct Pattern

```python
import requests

# Ascending order by name
response = requests.get(
    f"{API_URL}/dcim/devices/?ordering=name",
    headers=headers
)

# Descending order (prefix with -)
response = requests.get(
    f"{API_URL}/dcim/devices/?ordering=-created",
    headers=headers
)

# Multiple fields (comma-separated)
response = requests.get(
    f"{API_URL}/dcim/devices/?ordering=site,name",
    headers=headers
)
```

## Common Ordering Fields

- `name` - Alphabetical by name
- `created` - By creation date
- `-created` - Newest first
- `last_updated` - By modification date
- `id` - By database ID

## Related Rules

- [rest-pagination-required](./rest-pagination-required.md) - Pagination
- [rest-filtering-expressions](./rest-filtering-expressions.md) - Filtering
