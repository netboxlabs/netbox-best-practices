---
title: Calibrate Query Optimizer Against Production
impact: HIGH
category: graphql
tags: [graphql, optimization, calibration, performance]
netbox_version: "4.4+"
---

# graphql-calibrate-optimizer: Calibrate Query Optimizer Against Production

## Rationale

The [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer) provides complexity scores, but default scores are estimates based on typical data distributions. Calibrating against your actual NetBox instance gives accurate scores based on real object counts.

Without calibration:
- Scores may underestimate actual complexity
- Queries that seem acceptable may cause problems
- Over-cautious limits may reject valid queries

## Incorrect Pattern

```bash
# Using only default scores without calibration
netbox-query-optimizer analyze query.graphql

# Output with defaults:
# Complexity Score: 150 (OK)
# But actual data may result in score of 15,000
```

**Problems with this approach:**
- Default estimates may not match your data
- Sites with 500+ devices will have different scores than average
- May approve queries that will timeout in production

## Correct Pattern

```bash
# CORRECT: Calibrate against your actual NetBox instance
netbox-query-optimizer analyze query.graphql \
  --calibrate \
  --url https://netbox.example.com \
  --token nbt_abc123.xxxxx

# Output with real data:
# Complexity Score: 2,450 (WARNING - exceeds recommended threshold)
#
# Calibration Data:
#   - site_list: 45 sites
#   - avg devices per site: 120
#   - avg interfaces per device: 48
```

**Benefits:**
- Accurate complexity based on your data
- Identifies queries that will perform poorly in production
- Enables informed decisions about query design

## Calibration Process

### Step 1: Install the Optimizer

```bash
pip install netbox-graphql-query-optimizer
```

### Step 2: Create Calibration Token

Create a read-only token for calibration queries:

1. In NetBox, create a user with read-only permissions
2. Generate an API token for that user
3. Store token securely (don't commit to repo)

### Step 3: Run Calibrated Analysis

```bash
# Set token as environment variable
export NETBOX_TOKEN="nbt_abc123.xxxxx"

# Analyze with calibration
netbox-query-optimizer analyze query.graphql \
  --calibrate \
  --url https://netbox.example.com \
  --token "$NETBOX_TOKEN"
```

### Step 4: Interpret Results

```
Query Analysis Report (Calibrated)
==================================

Data Profile:
  Total sites: 45
  Total devices: 5,400
  Total interfaces: 259,200
  Avg devices per site: 120.0
  Avg interfaces per device: 48.0

Query: GetSiteOverview

Uncalibrated Score: 150
Calibrated Score: 12,960

Issues:
  - FAN_OUT: 45 sites × 120 devices × 48 interfaces = 259,200 potential objects
  - EXCEEDS_BUDGET: Score 12,960 exceeds threshold 1,000

Recommendations:
  1. Add stricter limits: devices(limit: 10), interfaces(limit: 10)
  2. Split into multiple queries
  3. Consider REST API for simpler access patterns
```

## Calibration in CI/CD

Create a calibration job that runs against staging/production-like data:

```yaml
# .github/workflows/query-validation.yml
name: Validate GraphQL Queries

on: [push, pull_request]

jobs:
  validate-queries:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - run: pip install netbox-graphql-query-optimizer

      - name: Analyze queries with calibration
        env:
          NETBOX_TOKEN: ${{ secrets.NETBOX_READONLY_TOKEN }}
          NETBOX_URL: ${{ secrets.NETBOX_STAGING_URL }}
        run: |
          for query in queries/*.graphql; do
            echo "Analyzing $query..."
            netbox-query-optimizer analyze "$query" \
              --calibrate \
              --url "$NETBOX_URL" \
              --token "$NETBOX_TOKEN" \
              --max-score 500 \
              --fail-on-warning
          done
```

## When to Recalibrate

Recalibrate when:
- Significant data growth (e.g., adding new sites)
- Changing data distribution (e.g., consolidating sites)
- Before production deployment
- Quarterly for ongoing projects

## Storing Calibration Data

For faster CI runs, cache calibration data:

```bash
# Export calibration data
netbox-query-optimizer calibrate \
  --url https://netbox.example.com \
  --token "$NETBOX_TOKEN" \
  --output calibration.json

# Use cached calibration
netbox-query-optimizer analyze query.graphql \
  --calibration-file calibration.json
```

## Exceptions

- **Development:** Default scores are fine for local testing
- **Empty databases:** Calibration is meaningless without data
- **Rapidly changing data:** May need frequent recalibration

## Related Rules

- [graphql-use-query-optimizer](./graphql-use-query-optimizer.md) - Basic optimizer usage
- [graphql-complexity-budgets](./graphql-complexity-budgets.md) - Setting thresholds
- [graphql-pagination-at-each-level](./graphql-pagination-at-each-level.md) - Fix high scores

## References

- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)
