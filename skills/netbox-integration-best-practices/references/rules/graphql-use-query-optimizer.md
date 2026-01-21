---
title: Use Query Optimizer for All GraphQL Queries
impact: CRITICAL
category: graphql
tags: [graphql, performance, optimization, tooling]
netbox_version: "4.4+"
---

# graphql-use-query-optimizer: Use Query Optimizer for All GraphQL Queries

## Rationale

The [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer) is an essential tool for production GraphQL usage. It performs static analysis to detect query patterns that cause severe performance problems.

**Real-world impact:** Query complexity scores have been reduced from 20,500 to 17 (~1,200x improvement) using this tool.

Without query analysis, it's easy to write queries that:
- Trigger N+1 database queries
- Fetch unbounded result sets
- Create multiplicative fan-out patterns
- Exceed safe query depth

## Incorrect Pattern

```graphql
# WRONG: Query written without analysis
# This may have N+1 issues, unbounded lists, or fan-out problems
query {
  site_list {
    name
    devices {
      name
      interfaces {
        name
        ip_addresses {
          address
        }
      }
    }
  }
}
```

**Problems with this approach:**
- No pagination limits (could return thousands of objects)
- Nested lists create fan-out (sites × devices × interfaces × IPs)
- Deep nesting (4 levels)
- No visibility into actual complexity

## Correct Pattern

First, install and run the query optimizer:

```bash
pip install netbox-graphql-query-optimizer

# Analyze the query
netbox-query-optimizer analyze query.graphql
```

**Output example:**
```
Query Analysis Report
=====================
Complexity Score: 20,500 (CRITICAL - exceeds threshold of 1,000)

Issues Found:
- UNBOUNDED_LIST: site_list has no pagination
- UNBOUNDED_LIST: devices has no pagination
- UNBOUNDED_LIST: interfaces has no pagination
- UNBOUNDED_LIST: ip_addresses has no pagination
- FAN_OUT: Multiplicative nesting detected
- DEPTH_EXCEEDED: Query depth 4 exceeds recommended max of 3

Recommendations:
1. Add limit parameter to all list queries
2. Reduce nesting depth or split into multiple queries
3. Consider REST API for this use case
```

**Corrected query after analysis:**

```graphql
# CORRECT: Query optimized based on analyzer feedback
query GetSiteSummary {
  site_list(limit: 10) {
    id
    name
    device_count
  }
}

# Separate query for device details when needed
query GetSiteDevices($siteId: Int!) {
  device_list(site_id: $siteId, limit: 50) {
    name
    interface_count
  }
}

# Separate query for interface details when needed
query GetDeviceInterfaces($deviceId: Int!) {
  interface_list(device_id: $deviceId, limit: 100) {
    name
    ip_addresses(limit: 10) {
      address
    }
  }
}
```

**After optimization:**
```
Complexity Score: 17 (OK)
No issues found.
```

## Calibration for Production

Default scores are estimates. Calibrate against your actual data for accurate scoring:

```bash
netbox-query-optimizer analyze query.graphql \
  --calibrate \
  --url https://netbox.example.com \
  --token nbt_abc123.xxxxx
```

This fetches real object counts from your NetBox instance to compute realistic complexity.

## Integration into CI/CD

Add query analysis to your CI pipeline:

```yaml
# .github/workflows/check-queries.yml
name: Validate GraphQL Queries
on: [push, pull_request]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install netbox-graphql-query-optimizer
      - run: |
          for query in queries/*.graphql; do
            echo "Analyzing $query..."
            netbox-query-optimizer analyze "$query" --max-score 500
          done
```

## Complexity Budget Guidelines

| Query Type | Max Score | Use Case |
|------------|-----------|----------|
| Dashboard widgets | 50 | Real-time display |
| Detail views | 200 | Single object with relations |
| Reports | 500 | Batch data retrieval |
| ETL/sync | 1000 | Background processing |

## Exceptions

- **Prototype/development:** Quick testing without optimization is acceptable
- **One-off queries:** Manual queries for debugging don't need formal analysis

But any query used in production applications MUST be analyzed.

## Related Rules

- [graphql-always-paginate](./graphql-always-paginate.md) - Every list needs limits
- [graphql-pagination-at-each-level](./graphql-pagination-at-each-level.md) - Paginate nested lists
- [graphql-max-depth](./graphql-max-depth.md) - Keep depth ≤3
- [graphql-complexity-budgets](./graphql-complexity-budgets.md) - Establish budgets

## References

- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)
- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
