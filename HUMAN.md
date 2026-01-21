# NetBox API Best Practices Guide

A concise guide for engineers building integrations with NetBox REST and GraphQL APIs.

**NetBox Version:** 4.4+ (4.5+ for v2 tokens)

---

## Executive Summary

### Authentication
- **Use v2 tokens** on NetBox 4.5+ (`Bearer nbt_<key>.<token>`)
- Migrate from v1 tokens before NetBox 4.7 (deprecation planned)

### REST API
- Always paginate list requests (max 1000 per page)
- Use `?brief=True` for minimal representations, or `?fields=` for specific fields
- Use `?exclude=config_context` when querying devices/VMs
- Use `PATCH` for partial updates, not `PUT`
- Use list endpoints with JSON arrays for bulk operations

### GraphQL
- Use [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer) for all queries
- Every list query MUST include pagination limits at every nesting level
- Keep query depth ≤3, never exceed 5
- Request only fields you need
- Filter by ID where possible to avoid SQL JOINs
- Use local filter fields instead of deeply nested filter paths

### Data Ingestion
- Use [Diode](https://github.com/netboxlabs/diode) for simplified ingestion (no dependency order, reference by name)
- Use REST/GraphQL for reading; use Diode for writing/populating

---

## Authentication

### v1 vs v2 Tokens

| Version | Format | Security | Status |
|---------|--------|----------|--------|
| v1 | `Token <40-char-hex>` | Plaintext in DB | Deprecated in 4.7 |
| v2 | `Bearer nbt_<key>.<secret>` | HMAC-SHA256 hashed | Recommended |

```python
# v2 token usage
headers = {"Authorization": f"Bearer {TOKEN}"}

# pynetbox handles it automatically
nb = pynetbox.api("https://netbox.example.com", token=TOKEN)
```

### Provisioning Endpoint

Use `/api/users/tokens/provision/` for automated token creation in CI/CD pipelines.

---

## REST API

### Key Patterns

| Do | Don't |
|----|-------|
| `PATCH` for partial updates | `PUT` (replaces entire object) |
| `?brief=True` for minimal data | Fetch full objects when you only need IDs/names |
| `?fields=foo,bar` for specific fields | Fetch full objects when you only need certain fields |
| `?exclude=config_context` | Include config_context in device/VM lists when not needed |
| Specific filters (`name__ic=`) | Generic search (`q=`) at scale |
| Bulk operations via arrays | Individual requests in loops |

### Pagination

```python
# Always paginate - NetBox defaults to 50 items
def get_all(endpoint):
    results, url = [], f"{API_URL}/{endpoint}/?limit=100"
    while url:
        data = requests.get(url, headers=headers).json()
        results.extend(data["results"])
        url = data.get("next")
    return results

# pynetbox handles pagination automatically
all_devices = list(nb.dcim.devices.all())
```

### Bulk Operations

```python
# Bulk create - POST array to list endpoint
requests.post(f"{API_URL}/dcim/devices/", json=[device1, device2, device3])

# Bulk update - PATCH array with IDs
requests.patch(f"{API_URL}/dcim/devices/", json=[
    {"id": 1, "status": "active"},
    {"id": 2, "status": "active"}
])

# Bulk delete - DELETE array with IDs
requests.delete(f"{API_URL}/dcim/devices/", json=[{"id": 1}, {"id": 2}])
```

All bulk operations are atomic (all-or-none). Signals and webhooks fire for **each object** (not once per batch).

### Filter Expressions

| Suffix | Meaning | Example |
|--------|---------|---------|
| (none) | Exact match | `status=active` |
| `__n` | Not equal | `status__n=offline` |
| `__ic` | Contains (case-insensitive) | `name__ic=core` |
| `__isw` | Starts with | `name__isw=sw-` |
| `__gte/__lte` | Greater/less than | `vlan_id__gte=100` |
| `__isnull` | Null check | `primary_ip__isnull=false` |

Custom fields: use `cf_` prefix (e.g., `cf_environment=production`)

---

## GraphQL

### Critical Rules

1. **Use the query optimizer** - Scores have been reduced from 20,500 to 17 using this tool
2. **Always paginate** - Every list, including nested lists
3. **Limit depth** - Keep ≤3 levels, never exceed 5
4. **Select only needed fields** - Don't over-fetch
5. **Filter by ID where possible** - Avoids SQL JOINs for related objects
6. **Use local filter fields** - Avoid deeply nested filter paths (see below)

```graphql
# WRONG: Unbounded, deep, over-fetching
query {
  site_list {
    devices {
      interfaces {
        ip_addresses { vrf { name } }
      }
    }
  }
}

# CORRECT: Paginated, shallow, selective
query {
  device_list(limit: 100, filters: {site_id: 123}) {
    name
    status
    primary_ip4 { address }
  }
}
```

### Complexity Budgets

| Query Type | Max Score |
|------------|-----------|
| Dashboard widgets | 50 |
| List views | 150 |
| Detail views | 200 |
| Reports | 500 |

### Filtering Considerations

**Filter by ID to avoid JOINs:**

```graphql
# SUBOPTIMAL: Requires JOIN to match site name
device_list(filters: {site: {name: {exact: "NYC-DC1"}}})

# OPTIMAL: Uses local site_id column directly
device_list(filters: {site_id: 123})
```

**Use local filter fields (4.5.1+):**

```graphql
# SUBOPTIMAL: Filter depth 3
interface_list(filters: {device: {site: {name: {exact: "NYC-DC1"}}}})

# OPTIMAL: Filter depth 2 (uses local site filter)
interface_list(filters: {site: {name: {exact: "NYC-DC1"}}})
```

### When to Use GraphQL vs REST vs Diode

| Use Case | Recommendation |
|----------|----------------|
| Related objects (2+ types) | GraphQL |
| Bulk writes with dependencies | Diode |
| Bulk create/update/delete | REST or Diode |
| Simple CRUD | REST |
| Flexible field selection | GraphQL |
| Reading/querying data | REST or GraphQL |

### Offset Pagination Limitations

NetBox GraphQL uses offset-based pagination, which degrades at scale:

| Page | Offset | Performance |
|------|--------|-------------|
| 1 | 0 | Fast |
| 100 | 9,900 | Slow |
| 1000 | 99,900 | Timeout risk |

**Version-specific solutions:**

| Version | Strategy |
|---------|----------|
| 4.4.x | Offset only (avoid deep pagination) |
| 4.5.x | ID range filtering workaround |
| 4.6.0+ | Cursor-based (`start` parameter) - [#21110](https://github.com/netbox-community/netbox/issues/21110) |

**4.5.x workaround** - Use ID range filtering to emulate cursors:

```graphql
query GetDevices($minId: Int!, $limit: Int!) {
  device_list(limit: $limit, filters: { id__gte: $minId }) {
    id
    name
  }
}
```

Track `max(id) + 1` from each page as the next `$minId`.

---

## Data Model

### Dependency Order

Objects must be created in dependency order. Key sequence:

1. **Organization**: Regions → Sites → Locations
2. **DCIM**: Manufacturers → Device Types → Device Roles → Devices → Interfaces
3. **IPAM**: RIRs → Aggregates → Prefixes → IP Addresses

**Tip:** Use Diode to avoid managing dependency order manually.

### Hierarchies

```
Site Hierarchy:               IPAM Hierarchy:
Region / Site Group           RIR
└── Site                      └── Aggregate
    └── Location (recursive)      └── Prefix (recursive)
        └── Rack                      └── IP Address
            └── Device
```

Region and Site Group are **parallel** groupings (both optional on a Site). Locations can be nested recursively.

### Custom Fields and Tags

```python
# Custom fields in create/update
data = {"name": "switch-01", "custom_fields": {"environment": "production"}}

# Filter by custom field
params = {"cf_environment": "production"}

# Tags for cross-cutting classification
data = {"name": "switch-01", "tags": [{"name": "pci-compliant"}]}
```

---

## Diode: Simplified Ingestion

For data ingestion, use Diode instead of direct API. Key benefits:

- **No dependency order** - Diode handles it automatically
- **Reference by name** - No ID lookups needed
- **Auto-creates missing objects** - Manufacturers, sites, roles, etc.

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Entity

with DiodeClient(
    target="https://<your-diode-host>:443/diode",
    app_name="network-discovery",
    app_version="1.0.0",
) as client:
    device = Device(
        name="switch-nyc-01",
        device_type="Cisco Catalyst 9300",  # By name!
        manufacturer="Cisco",                # Auto-created if missing
        site="NYC-DC1",                      # Auto-created if missing
        role="Access Switch",
    )
    response = client.ingest([Entity(device=device)])
```

> **Note on uniqueness:** Name resolution varies by model—devices are unique by name + site, interfaces by name + device, etc.

### Dry Run for Testing

Test ingestion without sending to server:

```python
from netboxlabs.diode.sdk import DiodeDryRunClient

with DiodeDryRunClient(app_name="my-app", output_dir="./test") as client:
    # Same API as DiodeClient, outputs to file instead
    client.ingest([Entity(device=device)])
```

### When to Use What

| Scenario | Tool |
|----------|------|
| Network discovery pushing data | Diode |
| Bulk migrations | Diode |
| Reading/querying data | REST/GraphQL |
| Single object CRUD | REST |

---

## Performance Tips

1. **Exclude config_context** - Single biggest performance win for device queries
2. **Use brief mode** - `?brief=True` for lists
3. **Avoid `q=` search** - Use specific filters instead

---

## Quick Reference

### Common Endpoints

| Resource | Endpoint |
|----------|----------|
| Devices | `/api/dcim/devices/` |
| Interfaces | `/api/dcim/interfaces/` |
| Sites | `/api/dcim/sites/` |
| Prefixes | `/api/ipam/prefixes/` |
| IP Addresses | `/api/ipam/ip-addresses/` |
| VLANs | `/api/ipam/vlans/` |

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200/201 | Success | Process response |
| 400 | Validation error | Check response body |
| 401 | Unauthorized | Check token |
| 403 | Forbidden | Check permissions |
| 429 | Rate limited | Backoff and retry |

### pynetbox Cheat Sheet

```python
nb.dcim.devices.all()                    # All (paginated iterator)
nb.dcim.devices.filter(site="nyc")       # Filter
nb.dcim.devices.get(name="switch-01")    # Single object
nb.dcim.devices.create(name="new", ...)  # Create
device.status = "active"; device.save()  # Update
device.delete()                          # Delete
```

---

## Additional Resources

- [NetBox REST API](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/)
- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
- [pynetbox](https://github.com/netbox-community/pynetbox)
- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)
- [Diode](https://github.com/netboxlabs/diode) and [Diode SDK](https://github.com/netboxlabs/diode-sdk-python)

---

> **Note:** For detailed rationale, exceptions, and comprehensive examples, see the individual rule files in `references/rules/`.
