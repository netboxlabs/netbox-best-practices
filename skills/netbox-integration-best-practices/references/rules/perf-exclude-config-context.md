---
title: Exclude Config Context for Performance
impact: HIGH
category: perf
tags: [performance, config-context, optimization]
netbox_version: "4.4+"
---

# perf-exclude-config-context: Exclude Config Context for Performance

## Rationale

Config context computation is the single most expensive operation for device queries. Excluding it from list operations can improve performance by 10-100x.

This is a critical optimization confirmed by community experience with large NetBox deployments.

## Correct Pattern

```python
# FAST: Exclude config_context
response = requests.get(
    f"{API_URL}/dcim/devices/?exclude=config_context",
    headers=headers
)
```

## Performance Impact

| Devices | With config_context | Without |
|---------|--------------------|---------|
| 100 | 2-5 seconds | 0.1-0.2 seconds |
| 1000 | 20-60 seconds | 0.5-1 second |
| 5000 | Timeout likely | 2-5 seconds |

See [rest-exclude-config-context](./rest-exclude-config-context.md) for detailed guidance.

## Related Rules

- [rest-exclude-config-context](./rest-exclude-config-context.md) - Full details
- [perf-brief-mode-lists](./perf-brief-mode-lists.md) - Brief mode
