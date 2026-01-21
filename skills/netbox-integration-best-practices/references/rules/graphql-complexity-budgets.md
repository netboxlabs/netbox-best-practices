---
title: Establish Complexity Budgets for Query Types
impact: LOW
category: graphql
tags: [graphql, performance, budgets, complexity]
netbox_version: "4.4+"
---

# graphql-complexity-budgets: Establish Complexity Budgets for Query Types

## Rationale

Different query types have different acceptable complexity levels. Establishing budgets helps teams write queries that perform well and catch problematic queries before deployment.

## Recommended Budgets

| Query Type | Max Score | Example Use Case |
|------------|-----------|------------------|
| Dashboard widgets | 50 | Real-time status displays |
| Autocomplete | 25 | Search suggestions |
| List views | 150 | Table/grid displays |
| Detail views | 200 | Single object with relations |
| Reports | 500 | Batch data retrieval |
| ETL/sync jobs | 1000 | Background data processing |

## Correct Pattern

```bash
# Enforce budgets in CI
netbox-query-optimizer analyze dashboard-query.graphql --max-score 50
netbox-query-optimizer analyze report-query.graphql --max-score 500
```

```python
# Runtime budget checking
QUERY_BUDGETS = {
    "dashboard": 50,
    "list_view": 150,
    "detail": 200,
    "report": 500
}

def validate_query_complexity(query_name, estimated_score):
    budget = QUERY_BUDGETS.get(query_name, 100)
    if estimated_score > budget:
        raise QueryTooComplexError(
            f"{query_name} score {estimated_score} exceeds budget {budget}"
        )
```

## Related Rules

- [graphql-use-query-optimizer](./graphql-use-query-optimizer.md) - Calculate scores
- [graphql-calibrate-optimizer](./graphql-calibrate-optimizer.md) - Accurate scoring
