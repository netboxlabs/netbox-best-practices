# Rule Categories

This document defines the rule categories, their prefixes, and the rules within each category.

---

## Authentication (auth-)

Rules for API authentication, token management, and access control.

**Typical Impact Range:** CRITICAL - MEDIUM

| Rule | Impact | Description |
|------|--------|-------------|
| [auth-use-v2-tokens](./auth-use-v2-tokens.md) | CRITICAL | Use v2 tokens on NetBox 4.5+, migrate before 4.7 |
| [auth-provisioning-endpoint](./auth-provisioning-endpoint.md) | MEDIUM | Use provisioning endpoint for automated token creation |

---

## REST API (rest-)

Rules for REST API usage, including request patterns, filtering, and bulk operations.

**Typical Impact Range:** CRITICAL - LOW

| Rule | Impact | Description |
|------|--------|-------------|
| [rest-list-endpoint-bulk-ops](./rest-list-endpoint-bulk-ops.md) | CRITICAL | Use list endpoints for bulk create/update/delete |
| [rest-pagination-required](./rest-pagination-required.md) | HIGH | Always paginate list requests |
| [rest-patch-vs-put](./rest-patch-vs-put.md) | HIGH | Use PATCH for partial updates, not PUT |
| [rest-brief-mode](./rest-brief-mode.md) | HIGH | Use ?brief=True for list operations |
| [rest-field-selection](./rest-field-selection.md) | HIGH | Use ?fields= to select only needed fields |
| [rest-exclude-config-context](./rest-exclude-config-context.md) | HIGH | Exclude config_context from device lists |
| [rest-avoid-search-filter-at-scale](./rest-avoid-search-filter-at-scale.md) | HIGH | Avoid q= search filter with large datasets |
| [rest-filtering-expressions](./rest-filtering-expressions.md) | MEDIUM | Use lookup expressions for efficient filtering |
| [rest-custom-field-filters](./rest-custom-field-filters.md) | MEDIUM | Filter by custom fields using cf_ prefix |
| [rest-nested-serializers](./rest-nested-serializers.md) | LOW | Understand nested vs flat serializers |
| [rest-ordering-results](./rest-ordering-results.md) | LOW | Use ordering parameter for sorted results |
| [rest-options-discovery](./rest-options-discovery.md) | LOW | Use OPTIONS for endpoint discovery |

---

## GraphQL (graphql-)

Rules for GraphQL API usage, query optimization, and avoiding common pitfalls.

**Typical Impact Range:** CRITICAL - LOW

| Rule | Impact | Description |
|------|--------|-------------|
| [graphql-use-query-optimizer](./graphql-use-query-optimizer.md) | CRITICAL | Use netbox-graphql-query-optimizer for all queries |
| [graphql-always-paginate](./graphql-always-paginate.md) | CRITICAL | Every list query MUST have pagination limits |
| [graphql-pagination-at-each-level](./graphql-pagination-at-each-level.md) | HIGH | Paginate nested lists at every nesting level |
| [graphql-select-only-needed](./graphql-select-only-needed.md) | HIGH | Request only fields you need |
| [graphql-calibrate-optimizer](./graphql-calibrate-optimizer.md) | HIGH | Calibrate optimizer against production data |
| [graphql-max-depth](./graphql-max-depth.md) | HIGH | Keep query depth ≤3, never exceed 5 |
| [graphql-filter-by-id](./graphql-filter-by-id.md) | HIGH | Use IDs in filters to avoid SQL JOINs |
| [graphql-avoid-nested-filters](./graphql-avoid-nested-filters.md) | HIGH | Use local filter fields instead of nested paths |
| [graphql-prefer-filters](./graphql-prefer-filters.md) | MEDIUM | Filter server-side, not client-side |
| [graphql-vs-rest-decision](./graphql-vs-rest-decision.md) | MEDIUM | Know when to use GraphQL vs REST |
| [graphql-complexity-budgets](./graphql-complexity-budgets.md) | LOW | Establish complexity budgets for query types |

---

## Performance (perf-)

Rules for optimizing API performance at scale.

**Typical Impact Range:** HIGH - LOW

| Rule | Impact | Description |
|------|--------|-------------|
| [perf-exclude-config-context](./perf-exclude-config-context.md) | HIGH | Exclude config_context for better performance |
| [perf-brief-mode-lists](./perf-brief-mode-lists.md) | HIGH | Use brief mode for large list operations |

---

## Data Modeling (data-)

Rules for understanding and properly using NetBox's data model.

**Typical Impact Range:** CRITICAL - MEDIUM

| Rule | Impact | Description |
|------|--------|-------------|
| [data-dependency-order](./data-dependency-order.md) | CRITICAL | Create objects in correct dependency order |
| [data-site-hierarchy](./data-site-hierarchy.md) | MEDIUM | Understand the site/location hierarchy |
| [data-ipam-hierarchy](./data-ipam-hierarchy.md) | MEDIUM | Understand IPAM hierarchy and relationships |
| [data-custom-fields](./data-custom-fields.md) | MEDIUM | Properly use custom fields |
| [data-tags-usage](./data-tags-usage.md) | MEDIUM | Use tags for cross-cutting classification |
| [data-tenant-isolation](./data-tenant-isolation.md) | MEDIUM | Use tenants for logical resource separation |
| [data-natural-keys](./data-natural-keys.md) | MEDIUM | Use natural keys for human-readable queries |

---

## Integration (integ-)

Rules for integration patterns, tooling, and best practices.

**Typical Impact Range:** HIGH - LOW

| Rule | Impact | Description |
|------|--------|-------------|
| [integ-diode-ingestion](./integ-diode-ingestion.md) | HIGH | Use Diode for simplified data ingestion |
| [integ-pynetbox-client](./integ-pynetbox-client.md) | HIGH | Use pynetbox for Python integrations |
| [integ-webhook-configuration](./integ-webhook-configuration.md) | MEDIUM | Configure webhooks for event-driven integration |
| [integ-change-tracking](./integ-change-tracking.md) | LOW | Query object changes for audit trails |

---

---

## Rule Count Summary

| Category | Count | Impact Distribution |
|----------|-------|---------------------|
| Authentication | 2 | 1 CRITICAL, 1 MEDIUM |
| REST API | 12 | 1 CRITICAL, 6 HIGH, 2 MEDIUM, 3 LOW |
| GraphQL | 11 | 2 CRITICAL, 6 HIGH, 2 MEDIUM, 1 LOW |
| Performance | 2 | 2 HIGH |
| Data Modeling | 7 | 1 CRITICAL, 6 MEDIUM |
| Integration | 4 | 2 HIGH, 1 MEDIUM, 1 LOW |
| **Total** | **38** | **5 CRITICAL, 16 HIGH, 12 MEDIUM, 5 LOW** |

> **Note:** Generic software development best practices (token storage, retry strategies, connection pooling, caching, parallelization, etc.) are intentionally excluded. This skill focuses on NetBox and Diode-specific patterns.
