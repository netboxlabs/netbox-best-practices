---
title: Use Brief Mode for Large List Operations
impact: HIGH
category: perf
tags: [performance, brief, optimization, lists]
netbox_version: "4.4+"
---

# perf-brief-mode-lists: Use Brief Mode for Large List Operations

## Rationale

Brief mode reduces response size by ~90%, dramatically improving performance for large list operations like dropdowns, autocomplete, or relationship lookups.

## Correct Pattern

```python
# 90% smaller responses
response = requests.get(
    f"{API_URL}/dcim/devices/?brief=True&limit=100",
    headers=headers
)
```

## Size Comparison

| Object | Full | Brief | Reduction |
|--------|------|-------|-----------|
| Device | ~2000 bytes | ~200 bytes | 90% |
| Prefix | ~800 bytes | ~150 bytes | 81% |

See [rest-brief-mode](./rest-brief-mode.md) for detailed guidance.

## Related Rules

- [rest-brief-mode](./rest-brief-mode.md) - Full details
- [rest-field-selection](./rest-field-selection.md) - Custom field selection
