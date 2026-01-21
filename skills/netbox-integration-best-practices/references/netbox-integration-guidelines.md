# NetBox Integration Guidelines

Master technical reference for NetBox REST and GraphQL API integration patterns.

**Target:** NetBox 4.4+ (4.5+ for v2 tokens)
**Scope:** API integration only (not plugin/script development)

---

## Table of Contents

1. [Authentication](#authentication)
2. [REST API Reference](#rest-api-reference)
3. [GraphQL API Reference](#graphql-api-reference)
4. [Performance Optimization](#performance-optimization)
5. [Data Model Reference](#data-model-reference)
6. [Integration Patterns](#integration-patterns)
7. [Diode: Data Ingestion Service](#diode-data-ingestion-service)
8. [Troubleshooting](#troubleshooting)

---

## Authentication

### Token Format Overview

NetBox supports two token formats. Understanding the differences is essential for secure integrations.

#### v1 Tokens (Legacy)

**Format:**
```
Token 0123456789abcdef0123456789abcdef01234567
```

**Characteristics:**
- 40-character hexadecimal string
- Stored in plaintext in database
- Simple format, widely supported
- **Status:** Deprecated in NetBox 4.7.0

**HTTP Header:**
```http
Authorization: Token 0123456789abcdef0123456789abcdef01234567
```

#### v2 Tokens (Recommended)

**Format:**
```
Bearer nbt_<key>.<token>
```

**Characteristics:**
- `nbt_` prefix identifies NetBox tokens
- `<key>` is the public key identifier (used for lookup)
- `<token>` is the secret portion
- Only HMAC-SHA256 hash stored in database
- Requires `API_TOKEN_PEPPERS` configuration
- **Introduced:** NetBox 4.5.0

**HTTP Header:**
```http
Authorization: Bearer nbt_abc123def456.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### v2 Token Migration Timeline

| NetBox Version | Token Status |
|----------------|-------------|
| < 4.5.0 | v1 tokens only |
| 4.5.0 | v2 tokens introduced, v1 fully supported |
| 4.7.0 | v1 tokens deprecated (still functional, warnings logged) |
| Future | v1 tokens removed |

**Recommendation:** All new integrations should use v2 tokens. Existing integrations should migrate before NetBox 4.7.

### Server Configuration for v2 Tokens

v2 tokens require `API_TOKEN_PEPPERS` in NetBox configuration:

```python
# configuration.py
API_TOKEN_PEPPERS = {
    'default': 'your-secret-pepper-value-minimum-32-chars',
}
```

Without this configuration, v2 tokens cannot be validated.

### Token Provisioning Endpoint

For automated token creation, use the provisioning endpoint:

```python
import requests

def provision_token(netbox_url, username, password, token_description=None):
    """Provision a new API token programmatically."""
    payload = {
        "username": username,
        "password": password
    }

    if token_description:
        payload["description"] = token_description

    response = requests.post(
        f"{netbox_url}/api/users/tokens/provision/",
        json=payload,
        headers={"Content-Type": "application/json"}
    )

    if response.status_code == 201:
        token_data = response.json()
        return token_data["key"]  # This is the full token (v1 or v2 format)
    else:
        raise Exception(f"Token provisioning failed: {response.text}")

# Usage
token = provision_token(
    "https://netbox.example.com",
    "automation-user",
    "secure-password",
    "CI/CD Pipeline Token"
)
```

### IP Restrictions

Tokens can be restricted to specific IP addresses or CIDR ranges:

```python
# Via NetBox UI or API, set allowed_ips on the token:
# ["10.0.0.0/8", "192.168.1.100/32"]

# Requests from other IPs will receive 403 Forbidden
```

### Read-Only Tokens

Create tokens without write permissions for monitoring and reporting:

1. Create a user with read-only permissions
2. Generate a token for that user
3. Use this token for dashboards, monitoring, and exports

### Token Best Practices Summary

| Practice | Priority |
|----------|----------|
| Use v2 tokens on NetBox 4.5+ | CRITICAL |
| Migrate v1 → v2 before 4.7 | CRITICAL |
| Never store tokens in code | CRITICAL |
| Use environment variables | HIGH |
| Implement token rotation | HIGH |
| Apply IP restrictions in production | MEDIUM |
| Use read-only tokens when possible | MEDIUM |

---

## REST API Reference

### Base URL Structure

```
https://netbox.example.com/api/
```

All endpoints are relative to this base URL.

### HTTP Methods

| Method | Purpose | Request Body | Idempotent |
|--------|---------|--------------|------------|
| `GET` | Retrieve resources | No | Yes |
| `POST` | Create resources | Yes | No |
| `PUT` | Replace entire resource | Yes | Yes |
| `PATCH` | Partial update | Yes | Yes |
| `DELETE` | Remove resources | Optional | Yes |
| `OPTIONS` | Discover endpoint schema | No | Yes |

### PATCH vs PUT

**Always use PATCH for updates unless you intend to replace the entire object.**

```python
import requests

API_URL = "https://netbox.example.com/api"
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxx",
    "Content-Type": "application/json"
}

# WRONG: PUT resets omitted fields
response = requests.put(
    f"{API_URL}/dcim/devices/123/",
    headers=headers,
    json={"status": "active"}  # Other fields may be cleared!
)

# CORRECT: PATCH updates only specified fields
response = requests.patch(
    f"{API_URL}/dcim/devices/123/",
    headers=headers,
    json={"status": "active"}  # Only status changes
)
```

### Bulk Operations with List Endpoints

NetBox does NOT have separate bulk endpoints. Instead, bulk operations use the standard list endpoints with JSON arrays.

**Bulk Create (POST array to list endpoint):**

```python
devices = [
    {
        "name": "switch-01",
        "device_type": 1,
        "role": 1,
        "site": 1,
        "status": "active"
    },
    {
        "name": "switch-02",
        "device_type": 1,
        "role": 1,
        "site": 1,
        "status": "active"
    }
]

response = requests.post(
    f"{API_URL}/dcim/devices/",
    headers=headers,
    json=devices  # Array of objects
)

# Returns array of created objects
created = response.json()
```

**Bulk Update (PATCH array with IDs to list endpoint):**

```python
updates = [
    {"id": 1, "status": "active"},
    {"id": 2, "status": "active"},
    {"id": 3, "status": "staged"}
]

response = requests.patch(
    f"{API_URL}/dcim/devices/",
    headers=headers,
    json=updates  # Each object MUST include "id"
)

# Returns array of updated objects
updated = response.json()
```

**Bulk Delete (DELETE array with IDs to list endpoint):**

```python
deletions = [
    {"id": 1},
    {"id": 2},
    {"id": 3}
]

response = requests.delete(
    f"{API_URL}/dcim/devices/",
    headers=headers,
    json=deletions
)

# Returns 204 No Content on success
```

**Atomicity:** All bulk operations are atomic. If any single item fails validation, the entire operation is rolled back and no changes are made.

### Pagination

**Always paginate list requests.** NetBox defaults to 50 items per page with a maximum of 1000.

**Query Parameters:**
- `limit`: Number of items per page (1-1000)
- `offset`: Number of items to skip

**Response Format:**
```json
{
    "count": 1500,
    "next": "https://netbox.example.com/api/dcim/devices/?limit=100&offset=100",
    "previous": null,
    "results": [...]
}
```

**Pagination Pattern:**

```python
def get_all_objects(api_url, endpoint, headers, limit=100):
    """Fetch all objects from an endpoint with proper pagination."""
    all_results = []
    url = f"{api_url}/{endpoint}/?limit={limit}"

    while url:
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        data = response.json()

        all_results.extend(data["results"])
        url = data.get("next")  # None when no more pages

    return all_results

# Usage
all_devices = get_all_objects(API_URL, "dcim/devices", headers)
```

**Async Pagination:**

```python
import asyncio
import httpx

async def get_all_objects_async(api_url, endpoint, headers, limit=100):
    """Fetch all objects with concurrent page fetching."""
    async with httpx.AsyncClient(headers=headers) as client:
        # First, get the count
        response = await client.get(f"{api_url}/{endpoint}/?limit=1")
        total = response.json()["count"]

        # Calculate number of pages
        pages = (total + limit - 1) // limit

        # Fetch all pages concurrently
        tasks = [
            client.get(f"{api_url}/{endpoint}/?limit={limit}&offset={i * limit}")
            for i in range(pages)
        ]

        responses = await asyncio.gather(*tasks)

        # Combine results
        all_results = []
        for resp in responses:
            all_results.extend(resp.json()["results"])

        return all_results
```

### Brief Mode

Use `?brief=True` for reduced response payloads when you only need basic object information.

**Full Response (~2KB per device):**
```python
response = requests.get(f"{API_URL}/dcim/devices/", headers=headers)
```

**Brief Response (~200 bytes per device):**
```python
response = requests.get(f"{API_URL}/dcim/devices/?brief=True", headers=headers)
```

Brief mode returns: `id`, `url`, `display`, and natural key fields only.

**Use brief mode when:**
- Populating dropdown/select menus
- Building relationship references
- Checking object existence
- Any scenario where full details aren't needed

### Field Selection

For fine-grained control, use `?fields=` to select specific fields:

```python
# Request only specific fields
response = requests.get(
    f"{API_URL}/dcim/devices/?fields=id,name,status,primary_ip4",
    headers=headers
)
```

**Nested field selection:**
```python
# Include nested fields
response = requests.get(
    f"{API_URL}/dcim/devices/?fields=id,name,site.name,device_type.model",
    headers=headers
)
```

### Excluding Heavy Fields

Some fields are computationally expensive. Use `?exclude=` to omit them:

```python
# CRITICAL: Exclude config_context for device lists
response = requests.get(
    f"{API_URL}/dcim/devices/?exclude=config_context",
    headers=headers
)
```

**Config context is the most impactful field to exclude.** Community reports show device list requests can be 10-100x slower with config_context included.

### Filtering

NetBox provides powerful filtering capabilities with lookup expressions.

**Basic Filtering:**
```python
# Exact match
response = requests.get(f"{API_URL}/dcim/devices/?status=active", headers=headers)

# Multiple values (OR)
response = requests.get(f"{API_URL}/dcim/devices/?status=active&status=planned", headers=headers)

# Related object filter
response = requests.get(f"{API_URL}/dcim/devices/?site=nyc-dc1", headers=headers)
response = requests.get(f"{API_URL}/dcim/devices/?site_id=1", headers=headers)
```

**Lookup Expressions:**

| Suffix | Description | Example |
|--------|-------------|---------|
| (none) | Exact match | `name=switch-01` |
| `__n` | Not equal | `status__n=offline` |
| `__ic` | Contains (case-insensitive) | `name__ic=core` |
| `__nic` | Not contains (case-insensitive) | `name__nic=test` |
| `__isw` | Starts with (case-insensitive) | `name__isw=sw-` |
| `__nisw` | Not starts with | `name__nisw=temp-` |
| `__iew` | Ends with (case-insensitive) | `name__iew=-prod` |
| `__niew` | Not ends with | `name__niew=-dev` |
| `__ie` | Exact (case-insensitive) | `name__ie=Switch-01` |
| `__nie` | Not exact (case-insensitive) | `name__nie=Switch-01` |
| `__empty` | Is empty | `description__empty=true` |
| `__gte` | Greater than or equal | `vlan_id__gte=100` |
| `__gt` | Greater than | `vlan_id__gt=100` |
| `__lte` | Less than or equal | `vlan_id__lte=200` |
| `__lt` | Less than | `vlan_id__lt=200` |
| `__isnull` | Is null | `primary_ip4__isnull=false` |

**Examples:**

```python
# Devices with names containing "core"
response = requests.get(f"{API_URL}/dcim/devices/?name__ic=core", headers=headers)

# VLANs in range 100-200
response = requests.get(
    f"{API_URL}/ipam/vlans/?vid__gte=100&vid__lte=200",
    headers=headers
)

# Devices with primary IPs assigned
response = requests.get(
    f"{API_URL}/dcim/devices/?primary_ip4__isnull=false",
    headers=headers
)

# Devices NOT in offline status
response = requests.get(
    f"{API_URL}/dcim/devices/?status__n=offline",
    headers=headers
)
```

### Avoiding the q= Search Filter at Scale

The `q=` parameter provides simple text search across multiple fields but becomes extremely slow with large datasets, especially devices with primary IPs.

```python
# SLOW with thousands of devices
response = requests.get(f"{API_URL}/dcim/devices/?q=switch", headers=headers)

# FASTER: Use specific filters
response = requests.get(f"{API_URL}/dcim/devices/?name__ic=switch", headers=headers)
```

### Custom Field Filters

Filter by custom fields using the `cf_` prefix:

```python
# Filter by custom field value
response = requests.get(
    f"{API_URL}/dcim/devices/?cf_environment=production",
    headers=headers
)

# Multiple custom field filters
response = requests.get(
    f"{API_URL}/dcim/devices/?cf_environment=production&cf_tier=1",
    headers=headers
)

# Custom field with lookup expression
response = requests.get(
    f"{API_URL}/dcim/devices/?cf_deployment_date__gte=2024-01-01",
    headers=headers
)
```

### Ordering Results

Use `?ordering=` to sort results:

```python
# Ascending order
response = requests.get(f"{API_URL}/dcim/devices/?ordering=name", headers=headers)

# Descending order (prefix with -)
response = requests.get(f"{API_URL}/dcim/devices/?ordering=-created", headers=headers)

# Multiple fields
response = requests.get(f"{API_URL}/dcim/devices/?ordering=site,name", headers=headers)
```

### Request Correlation

Use `X-Request-ID` header to correlate requests with NetBox logs:

```python
import uuid

request_id = str(uuid.uuid4())
headers["X-Request-ID"] = request_id

response = requests.get(f"{API_URL}/dcim/devices/", headers=headers)

# Log for debugging
print(f"Request {request_id}: {response.status_code}")
```

### OPTIONS Discovery

Use the OPTIONS method to discover endpoint schema:

```python
response = requests.options(f"{API_URL}/dcim/devices/", headers=headers)
schema = response.json()

# View available fields
for field, meta in schema["actions"]["POST"].items():
    required = meta.get("required", False)
    field_type = meta.get("type", "unknown")
    print(f"{field}: {field_type} (required: {required})")
```

### Error Handling

**HTTP Status Codes:**

| Code | Meaning | Response Body | Action |
|------|---------|---------------|--------|
| 200 | Success (GET/PATCH/PUT) | Object(s) | Process data |
| 201 | Created (POST) | Created object(s) | Process data |
| 204 | No Content (DELETE) | None | Success |
| 400 | Bad Request | Validation errors | Fix input |
| 401 | Unauthorized | Error message | Check token |
| 403 | Forbidden | Error message | Check permissions |
| 404 | Not Found | Error message | Check endpoint/ID |
| 405 | Method Not Allowed | Error message | Check HTTP method |
| 409 | Conflict | Error message | Handle conflict |
| 429 | Too Many Requests | Error message | Backoff and retry |
| 500+ | Server Error | Error message | Retry with backoff |

**Error Response Format:**
```json
{
    "name": ["This field is required."],
    "site": ["Invalid pk \"999\" - object does not exist."]
}
```

**Comprehensive Error Handling:**

```python
def api_request(method, url, headers, json=None, max_retries=3):
    """Make API request with comprehensive error handling."""
    for attempt in range(max_retries):
        try:
            response = requests.request(
                method=method,
                url=url,
                headers=headers,
                json=json,
                timeout=30
            )

            if response.status_code == 429:
                retry_after = int(response.headers.get("Retry-After", 60))
                time.sleep(retry_after)
                continue

            if response.status_code >= 500:
                if attempt < max_retries - 1:
                    time.sleep(2 ** attempt)
                    continue
                response.raise_for_status()

            if response.status_code == 400:
                errors = response.json()
                raise ValidationError(f"Validation failed: {errors}")

            if response.status_code == 401:
                raise AuthenticationError("Invalid or expired token")

            if response.status_code == 403:
                raise PermissionError("Insufficient permissions")

            if response.status_code == 404:
                raise NotFoundError(f"Resource not found: {url}")

            response.raise_for_status()
            return response

        except requests.exceptions.Timeout:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise

        except requests.exceptions.ConnectionError:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise

    raise Exception("Max retries exceeded")
```

---

## GraphQL API Reference

### Endpoint

```
https://netbox.example.com/graphql/
```

All GraphQL requests are POST requests to this single endpoint.

### Basic Query Structure

```graphql
query {
  device_list(limit: 100) {
    name
    status
    site {
      name
    }
  }
}
```

**Python Request:**

```python
import requests

def graphql_query(netbox_url, token, query, variables=None):
    """Execute a GraphQL query against NetBox."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    payload = {"query": query}
    if variables:
        payload["variables"] = variables

    response = requests.post(
        f"{netbox_url}/graphql/",
        headers=headers,
        json=payload
    )
    response.raise_for_status()

    result = response.json()
    if "errors" in result:
        raise Exception(f"GraphQL errors: {result['errors']}")

    return result["data"]
```

### The Query Optimizer

**The [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer) is essential for production GraphQL usage.**

This tool performs static analysis to detect:
- N+1 query patterns
- Unbounded queries (missing pagination)
- Fan-out patterns
- Query depth violations

**Installation:**
```bash
pip install netbox-graphql-query-optimizer
```

**Usage:**
```bash
# Analyze a query file
netbox-query-optimizer analyze query.graphql

# Calibrate against production NetBox
netbox-query-optimizer analyze query.graphql \
  --calibrate \
  --url https://netbox.example.com \
  --token nbt_abc123.xxxxx
```

**Real Impact:** Queries have been optimized from complexity scores of 20,500 down to 17 (~1,200x improvement).

### Pagination Requirements

**Every list query MUST include pagination limits.**

```graphql
# WRONG: Unbounded query
query {
  device_list {
    name
  }
}

# CORRECT: Paginated query
query {
  device_list(limit: 100, offset: 0) {
    name
  }
}
```

### Pagination at Every Nesting Level

Nested lists must also be paginated:

```graphql
# WRONG: Nested lists unbounded
query {
  site_list(limit: 10) {
    name
    devices {  # Could return thousands
      name
      interfaces {  # Could return thousands per device
        name
      }
    }
  }
}

# CORRECT: Paginate at every level
query {
  site_list(limit: 10) {
    name
    devices(limit: 50) {
      name
      interfaces(limit: 100) {
        name
      }
    }
  }
}
```

### Offset Pagination Performance at Scale

NetBox GraphQL uses offset-based pagination, which has significant performance implications for large datasets.

**The Problem:** Offset pagination requires scanning all rows up to the offset:

```
Query: device_list(limit: 100, offset: 50000)
Database: Scan 50,000 rows, discard them, return next 100 → SLOW
```

Performance degrades linearly with offset depth:

| Offset | Rows Scanned | Typical Latency |
|--------|--------------|-----------------|
| 0 | 100 | ~50ms |
| 10,000 | 10,100 | ~500ms |
| 100,000 | 100,100 | ~5s or timeout |

**Version-Specific Solutions:**

| Version | Approach | Syntax |
|---------|----------|--------|
| ≤4.4.x | Offset only | `limit: 100, offset: 0` |
| 4.5.x | ID range filtering | `limit: 100, filters: {id__gte: 5000}` |
| 4.6.0+ | Cursor-based (planned) | `start: 5000, limit: 100` |

**ID Range Pagination (4.5.x Workaround):**

```graphql
query GetDevicesAfter($lastId: Int!, $limit: Int!) {
  device_list(
    limit: $limit
    filters: { id__gte: $lastId }
  ) {
    id  # Required for cursor tracking
    name
    status
  }
}
```

```python
def fetch_with_id_cursor(netbox_url, token, page_size=100):
    """Efficient pagination using ID cursors."""
    all_results = []
    min_id = 0

    while True:
        data = graphql_query(netbox_url, token, query,
                           {"lastId": min_id, "limit": page_size})
        items = data["device_list"]

        if not items:
            break

        all_results.extend(items)

        if len(items) < page_size:
            break

        min_id = max(item["id"] for item in items) + 1

    return all_results
```

**Caveats:**
- Inconsistent page sizes if IDs have gaps
- Must fetch `id` in every query
- Cannot jump to arbitrary pages

**Cursor-Based Pagination (4.6.0+):**

```graphql
# Planned syntax
query {
  device_list(start: 5000, limit: 100) {
    id
    name
  }
}
```

Uses `WHERE pk >= start` for O(1) performance at any depth.

See [GitHub #21110](https://github.com/netbox-community/netbox/issues/21110) for status.

### Field Selection

Only request fields you need:

```graphql
# WRONG: Over-fetching
query {
  device_list(limit: 100) {
    id
    name
    status
    device_type {
      id
      name
      slug
      manufacturer {
        id
        name
        slug
        description
      }
      model
      part_number
    }
    role {
      id
      name
      slug
      color
      description
    }
    # ... many more fields
  }
}

# CORRECT: Request only what you need
query {
  device_list(limit: 100) {
    name
    status
    device_type {
      model
    }
  }
}
```

### Query Depth Guidelines

Keep query depth at 3 or below. Never exceed 5:

```
Level 1: site_list
├── Level 2: devices
│   ├── Level 3: interfaces
│   │   └── Level 4: ip_addresses (AVOID)
│   │       └── Level 5: vrf (NEVER)
```

**Instead of deep nesting, use multiple targeted queries:**

```graphql
# Query 1: Get sites and devices
query GetSitesAndDevices {
  site_list(limit: 10) {
    id
    name
    devices(limit: 50) {
      id
      name
    }
  }
}

# Query 2: Get interfaces for specific devices
query GetInterfaces($deviceIds: [Int!]) {
  interface_list(device_id: $deviceIds, limit: 200) {
    name
    device {
      id
    }
    ip_addresses(limit: 10) {
      address
    }
  }
}
```

### Server-Side Filtering

Filter in the query, not in your application:

```graphql
# WRONG: Fetch all, filter in code
query {
  device_list(limit: 1000) {
    name
    status
    site { name }
  }
}
# Then: devices.filter(d => d.status === "active")

# CORRECT: Filter server-side
query {
  device_list(
    limit: 100
    filters: {
      status: "active"
      site: "nyc-dc1"
    }
  ) {
    name
    status
    site { name }
  }
}
```

### Complexity Budgets

Establish complexity budgets for different query types:

| Query Type | Recommended Max Score |
|------------|----------------------|
| Dashboard widgets | < 50 |
| Detail views | < 200 |
| Reports | < 500 |
| Batch operations | < 1000 |

**Example Scores:**

```graphql
# Score: ~15 (excellent for dashboard)
query DashboardSummary {
  site_list(limit: 20) {
    name
    status
  }
}

# Score: ~150 (acceptable for detail view)
query SiteDetail($id: Int!) {
  site(id: $id) {
    name
    status
    region { name }
    devices(limit: 10) {
      name
      status
    }
    prefixes(limit: 10) {
      prefix
      status
    }
  }
}
```

### Calibration

Default optimizer scores are estimates. Calibrate against your production data for accuracy:

```bash
netbox-query-optimizer analyze query.graphql \
  --calibrate \
  --url https://netbox.example.com \
  --token nbt_abc123.xxxxx
```

Calibration fetches actual object counts from your NetBox instance to compute realistic complexity scores.

### Fan-Out Pattern Avoidance

Fan-out occurs when each parent fetches many children, multiplying total objects:

```
10 sites × 100 devices × 50 interfaces = 50,000 objects
```

**Instead of fan-out queries, use targeted queries:**

```graphql
# BAD: Fan-out pattern
query {
  site_list(limit: 10) {
    devices(limit: 100) {
      interfaces(limit: 50) {
        name
      }
    }
  }
}

# BETTER: Get summary first
query GetSiteSummary {
  site_list(limit: 10) {
    id
    name
    device_count
  }
}

# Then fetch details for specific sites of interest
query GetSiteDevices($siteId: Int!) {
  device_list(site_id: $siteId, limit: 100) {
    name
    interface_count
  }
}
```

### GraphQL vs REST Decision Matrix

| Use Case | Recommendation | Reasoning |
|----------|---------------|-----------|
| Single object by ID | REST | Simpler, more cacheable |
| List with simple filters | REST | Well-optimized, built-in pagination |
| Related data (2+ object types) | GraphQL | Single request vs multiple |
| Deeply nested data | Consider carefully | May need multiple queries either way |
| Real-time dashboards | REST | Easier HTTP caching |
| Flexible reporting | GraphQL | Dynamic field selection |
| Bulk create/update/delete | REST | Native support for bulk operations |
| CI/CD scripts | REST | Simpler scripting |

### GraphQL Error Handling

```python
def graphql_query(netbox_url, token, query, variables=None):
    """Execute GraphQL query with error handling."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    payload = {"query": query}
    if variables:
        payload["variables"] = variables

    try:
        response = requests.post(
            f"{netbox_url}/graphql/",
            headers=headers,
            json=payload,
            timeout=30
        )
        response.raise_for_status()

        result = response.json()

        # GraphQL returns 200 even with query errors
        if "errors" in result:
            for error in result["errors"]:
                print(f"GraphQL Error: {error.get('message')}")
                if "locations" in error:
                    print(f"  Location: {error['locations']}")
            raise GraphQLError(result["errors"])

        return result["data"]

    except requests.exceptions.HTTPError as e:
        if response.status_code == 401:
            raise AuthenticationError("Invalid token")
        elif response.status_code == 403:
            raise PermissionError("Insufficient permissions")
        raise
```

---

## Performance Optimization

### Critical: Exclude Config Context

**This is the single most impactful optimization for device queries.**

```python
# SLOW: May be 10-100x slower
response = requests.get(f"{API_URL}/dcim/devices/", headers=headers)

# FAST: Exclude config_context
response = requests.get(
    f"{API_URL}/dcim/devices/?exclude=config_context",
    headers=headers
)
```

Config context computation involves:
- Traversing device hierarchy
- Evaluating context data rules
- Merging multiple context sources
- JSON serialization of potentially large data

Always exclude unless specifically needed.

### Brief Mode for Lists

Use brief mode when you don't need full object details:

```python
# Full response: ~2KB per device
response = requests.get(f"{API_URL}/dcim/devices/", headers=headers)

# Brief response: ~200 bytes per device
response = requests.get(f"{API_URL}/dcim/devices/?brief=True", headers=headers)
```

**10x reduction in payload size** for typical device objects.

### Avoid Generic Search at Scale

The `q=` parameter becomes slow with large datasets:

```python
# SLOW: Generic search
requests.get(f"{API_URL}/dcim/devices/?q=switch", headers=headers)

# FAST: Specific filter
requests.get(f"{API_URL}/dcim/devices/?name__ic=switch", headers=headers)
```

The slowdown is especially severe for devices with primary IPs assigned.

### Parallel Requests

Parallelize independent requests for faster data fetching:

**Using asyncio and httpx:**

```python
import asyncio
import httpx

async def fetch_inventory():
    """Fetch multiple resource types in parallel."""
    async with httpx.AsyncClient(headers=headers, timeout=30) as client:
        tasks = [
            client.get(f"{API_URL}/dcim/devices/?limit=100&exclude=config_context"),
            client.get(f"{API_URL}/dcim/sites/?limit=100&brief=True"),
            client.get(f"{API_URL}/ipam/prefixes/?limit=100"),
            client.get(f"{API_URL}/ipam/ip-addresses/?limit=100"),
        ]

        responses = await asyncio.gather(*tasks)

        return {
            "devices": responses[0].json()["results"],
            "sites": responses[1].json()["results"],
            "prefixes": responses[2].json()["results"],
            "ip_addresses": responses[3].json()["results"],
        }

inventory = asyncio.run(fetch_inventory())
```

### Pagination Strategy

Choose appropriate page sizes based on your use case:

| Scenario | Recommended Limit |
|----------|------------------|
| Interactive UI | 25-50 |
| Background sync | 100-250 |
| Bulk export | 500-1000 |
| Streaming processing | 100 |

Larger pages reduce HTTP overhead but increase memory usage and response latency.

### Caching Strategies

Implement caching for data that doesn't change frequently:

```python
import time
from functools import lru_cache

class NetBoxCache:
    def __init__(self, default_ttl=300):
        self._cache = {}
        self._default_ttl = default_ttl

    def get(self, key):
        if key in self._cache:
            value, expiry = self._cache[key]
            if time.time() < expiry:
                return value
            del self._cache[key]
        return None

    def set(self, key, value, ttl=None):
        ttl = ttl or self._default_ttl
        self._cache[key] = (value, time.time() + ttl)

    def invalidate(self, key):
        self._cache.pop(key, None)

cache = NetBoxCache(default_ttl=300)

def get_sites(api_url, headers):
    """Get sites with caching."""
    cached = cache.get("sites")
    if cached:
        return cached

    response = requests.get(f"{api_url}/dcim/sites/?brief=True", headers=headers)
    sites = response.json()["results"]
    cache.set("sites", sites)
    return sites
```

**Cache appropriate data:**
- Site/Location hierarchy (changes rarely)
- Device types and roles (reference data)
- Tags and custom fields (configuration)

**Don't cache:**
- Device status (changes frequently)
- IP address assignments (operational data)
- Object counts (dynamic)

### Version Performance Notes

Performance varies by NetBox version:

- **v4.0.0:** Had some performance regressions
- **v4.4.9+:** Includes fixes for several performance issues
- **Always test performance before upgrading** in a staging environment with production-like data

### Infrastructure Considerations

Performance depends on infrastructure:

| Component | Impact |
|-----------|--------|
| Database indexes | Critical - missing indexes cause severe slowdowns |
| Redis/Valkey cache | High - proper configuration dramatically impacts performance |
| Connection pooling | Medium - important for high-volume applications |
| Database maintenance | Medium - regular VACUUM and REINDEX |

---

## Data Model Reference

### Dependency Order

Objects must be created in dependency order. A child cannot reference a parent that doesn't exist.

**Complete Population Order:**

```
1. Organization Layer
   ├── Tenant Groups → Tenants
   ├── Regions → Site Groups → Sites → Locations
   └── Contact Groups → Contacts → Contact Assignments

2. DCIM Prerequisites
   ├── Manufacturers
   ├── Device Types (requires Manufacturer)
   ├── Module Types (requires Manufacturer)
   ├── Platforms
   ├── Device Roles
   └── Rack Roles

3. DCIM Infrastructure
   ├── Racks (requires Site, optional Location)
   ├── Devices (requires Device Type, Role, Site)
   ├── Modules (requires Device, Module Type)
   └── Interfaces, Ports (require Device)

4. IPAM
   ├── RIRs
   ├── VRFs
   ├── Route Targets
   ├── Aggregates (require RIR)
   ├── Prefixes (optional VRF, Site)
   ├── IP Ranges
   ├── IP Addresses (optional VRF)
   ├── VLAN Groups
   └── VLANs (optional VLAN Group, Site)

5. Virtualization
   ├── Cluster Types → Cluster Groups → Clusters
   ├── Virtual Machines (require Cluster)
   └── VM Interfaces (require VM)

6. Circuits
   ├── Providers → Provider Accounts → Provider Networks
   ├── Circuit Types
   └── Circuits → Circuit Terminations

7. Connections
   ├── Cables (require endpoints)
   ├── Wireless Links
   └── Power Paths
```

**Example Population Script:**

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Step 1: Organization
region = nb.dcim.regions.create(name="North America", slug="na")
site = nb.dcim.sites.create(
    name="NYC-DC1",
    slug="nyc-dc1",
    region=region.id,
    status="active"
)

# Step 2: DCIM Prerequisites
manufacturer = nb.dcim.manufacturers.create(name="Cisco", slug="cisco")
device_type = nb.dcim.device_types.create(
    manufacturer=manufacturer.id,
    model="Catalyst 9300",
    slug="c9300"
)
role = nb.dcim.device_roles.create(
    name="Access Switch",
    slug="access-switch",
    color="00ff00"
)

# Step 3: Create device (dependencies exist)
device = nb.dcim.devices.create(
    name="nyc-dc1-sw01",
    device_type=device_type.id,
    role=role.id,
    site=site.id,
    status="active"
)
```

### DCIM Hierarchy

```
Region (optional)
└── Site Group (optional)
    └── Site
        ├── Location (recursive hierarchy)
        │   └── Rack
        │       └── Device (racked)
        │           ├── Module Bay
        │           │   └── Module
        │           │       └── Module Interface
        │           ├── Interface
        │           ├── Console Port
        │           ├── Console Server Port
        │           ├── Power Port
        │           ├── Power Outlet
        │           ├── Front Port
        │           ├── Rear Port
        │           ├── Device Bay
        │           │   └── Child Device
        │           └── Inventory Item (recursive)
        └── Device (non-racked)
```

### IPAM Hierarchy

```
RIR
└── Aggregate
    └── Prefix (hierarchical - can be nested)
        ├── IP Range
        └── IP Address

VRF (scopes Prefixes and IP Addresses)
├── Import Route Targets
├── Export Route Targets
└── Associated Prefixes/IPs

VLAN Group (optional)
└── VLAN
    └── Prefix (many-to-many via VLAN assignment)
```

### Natural Keys

Natural keys provide human-readable identification:

```python
# By name (most common natural key)
device = nb.dcim.devices.get(name="switch-01")
site = nb.dcim.sites.get(name="NYC-DC1")

# By slug
site = nb.dcim.sites.get(slug="nyc-dc1")

# Multiple fields for unique identification
interface = nb.dcim.interfaces.get(device="switch-01", name="Gi0/1")
```

### Custom Fields

Extend the data model for organization-specific needs:

```python
# Create with custom fields
device = nb.dcim.devices.create(
    name="server-01",
    device_type=device_type.id,
    role=role.id,
    site=site.id,
    custom_fields={
        "environment": "production",
        "cost_center": "IT-001",
        "maintenance_window": "sunday-0200"
    }
)

# Update custom fields
device.custom_fields["environment"] = "staging"
device.save()

# Filter by custom fields
production_devices = nb.dcim.devices.filter(cf_environment="production")
```

### Tags

Tags provide cross-cutting classification:

```python
# Create a tag
tag = nb.extras.tags.create(
    name="PCI-Compliant",
    slug="pci-compliant",
    color="ff0000"
)

# Apply to an object
device = nb.dcim.devices.get(name="server-01")
device.tags = [{"name": "PCI-Compliant"}]
device.save()

# Query by tag (works on multiple object types)
pci_devices = nb.dcim.devices.filter(tag="pci-compliant")
pci_vlans = nb.ipam.vlans.filter(tag="pci-compliant")
pci_prefixes = nb.ipam.prefixes.filter(tag="pci-compliant")
```

### Tenant Isolation

Use tenants for logical resource separation:

```python
# Create tenant structure
tenant_group = nb.tenancy.tenant_groups.create(
    name="Customers",
    slug="customers"
)
tenant = nb.tenancy.tenants.create(
    name="ACME Corp",
    slug="acme-corp",
    group=tenant_group.id
)

# Assign resources to tenant
prefix = nb.ipam.prefixes.create(
    prefix="10.100.0.0/24",
    tenant=tenant.id,
    status="active"
)

vlan = nb.ipam.vlans.create(
    vid=100,
    name="ACME-Data",
    tenant=tenant.id
)

# Query by tenant
acme_resources = {
    "prefixes": list(nb.ipam.prefixes.filter(tenant="acme-corp")),
    "vlans": list(nb.ipam.vlans.filter(tenant="acme-corp")),
    "devices": list(nb.dcim.devices.filter(tenant="acme-corp")),
}
```

---

## Integration Patterns

### pynetbox (Recommended Python Client)

pynetbox is the official Python client for NetBox:

```python
import pynetbox

# Initialize
nb = pynetbox.api(
    url="https://netbox.example.com",
    token="nbt_abc123.xxxxx"
)

# Optional: Disable SSL verification (development only)
# nb.http_session.verify = False

# Query patterns
all_devices = nb.dcim.devices.all()  # Iterator with automatic pagination
filtered = nb.dcim.devices.filter(site="nyc-dc1", status="active")
single = nb.dcim.devices.get(name="switch-01")
by_id = nb.dcim.devices.get(123)

# Create
new_device = nb.dcim.devices.create(
    name="new-switch",
    device_type=1,
    role=1,
    site=1
)

# Update
device = nb.dcim.devices.get(name="switch-01")
device.status = "planned"
device.save()

# Delete
device.delete()

# Bulk create
devices_data = [
    {"name": f"switch-{i:02d}", "device_type": 1, "role": 1, "site": 1}
    for i in range(10)
]
created = nb.dcim.devices.create(devices_data)
```

### Webhook Integration

Configure NetBox to send webhooks on object changes:

```python
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret"

def verify_signature(payload, signature):
    """Verify webhook signature."""
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha512
    ).hexdigest()
    return hmac.compare_digest(signature, expected)

@app.route("/netbox-webhook", methods=["POST"])
def handle_webhook():
    # Verify signature
    signature = request.headers.get("X-Hook-Signature", "")
    if not verify_signature(request.data, signature):
        return jsonify({"error": "Invalid signature"}), 401

    event = request.json

    event_type = event["event"]     # "created", "updated", "deleted"
    model = event["model"]          # "dcim.device", "ipam.prefix", etc.
    username = event["username"]    # User who made the change
    data = event["data"]            # Object data
    snapshots = event.get("snapshots", {})  # Before/after for updates

    # Route to handlers
    if model == "dcim.device":
        if event_type == "created":
            handle_device_created(data)
        elif event_type == "updated":
            handle_device_updated(data, snapshots)
        elif event_type == "deleted":
            handle_device_deleted(data)

    return jsonify({"status": "processed"}), 200
```

### Change Tracking

Query object changes for audit trails:

```python
# Get recent changes for a specific object
changes = nb.extras.object_changes.filter(
    changed_object_type="dcim.device",
    changed_object_id=123,
    limit=50
)

for change in changes:
    print(f"Time: {change.time}")
    print(f"Action: {change.action}")
    print(f"User: {change.user_name}")
    print(f"Before: {change.prechange_data}")
    print(f"After: {change.postchange_data}")
    print("---")

# Get all recent changes
recent_changes = nb.extras.object_changes.filter(
    time_after="2024-01-01T00:00:00Z",
    limit=100
)
```

### Event-Driven Architecture

Design integrations to react to NetBox events:

```python
class NetBoxEventHandler:
    """Handle NetBox webhook events."""

    def __init__(self, cmdb_client, dns_client):
        self.cmdb = cmdb_client
        self.dns = dns_client

    def handle_device_created(self, data):
        """React to new device creation."""
        device_name = data["name"]
        primary_ip = data.get("primary_ip4", {}).get("address")

        # Update CMDB
        self.cmdb.create_ci(
            name=device_name,
            type="network_device",
            ip=primary_ip
        )

        # Create DNS record if IP assigned
        if primary_ip:
            ip, prefix_len = primary_ip.split("/")
            self.dns.create_a_record(device_name, ip)

    def handle_device_updated(self, data, snapshots):
        """React to device updates."""
        device_name = data["name"]

        # Check if IP changed
        old_ip = snapshots.get("prechange", {}).get("primary_ip4")
        new_ip = data.get("primary_ip4", {}).get("address")

        if old_ip != new_ip:
            self.dns.update_a_record(device_name, new_ip)

    def handle_device_deleted(self, data):
        """React to device deletion."""
        device_name = data["name"]

        self.cmdb.delete_ci(device_name)
        self.dns.delete_a_record(device_name)
```

---

## Diode: Data Ingestion Service

For high-volume data ingestion scenarios, [Diode](https://github.com/netboxlabs/diode) provides a simplified alternative to direct API usage.

### What is Diode?

Diode is a data ingestion service from NetBox Labs that:

- **Resolves dependencies automatically**: Specify objects by name, not ID
- **Creates missing objects**: Referenced objects are created if they don't exist
- **Eliminates ordering concerns**: No need to create objects in dependency order
- **Uses gRPC protocol**: High-performance, efficient transport
- **Supports 70+ entity types**: All major NetBox object types

### When to Use Diode

| Scenario | Recommendation |
|----------|----------------|
| Network discovery pushing data | **Diode** |
| Bulk data migrations | **Diode** |
| Scripts creating many related objects | **Diode** |
| Reading/querying NetBox data | REST/GraphQL |
| Single object CRUD operations | REST API |
| Complex filtered searches | REST/GraphQL |

### Prerequisites

- NetBox 4.2.3+
- Diode Server deployed
- Diode NetBox Plugin installed
- Diode SDK (Python or Go)

### Installation

```bash
pip install netboxlabs-diode-sdk
```

### Basic Usage

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Entity

with DiodeClient(
    target="grpc://diode.example.com:8080/diode",
    app_name="my-discovery-tool",
    app_version="1.0.0",
) as client:
    # Specify dependencies by NAME - no IDs needed!
    # Diode creates manufacturer, device_type, site, role if they don't exist
    device = Device(
        name="switch-nyc-01",
        device_type="Cisco Catalyst 9300",  # By name
        manufacturer="Cisco",                # By name
        site="NYC-DC1",                      # By name
        role="Access Switch",                # By name
        serial="ABC123456",
        status="active",
        tags=["production", "network"],
    )

    response = client.ingest([Entity(device=device)])

    if response.errors:
        print(f"Errors: {response.errors}")
```

### Dependency Resolution: Before and After

**Without Diode (manual dependency management):**

```python
# Must create in order: manufacturer → device_type → site → role → device
mfr = nb.dcim.manufacturers.create(name="Cisco", slug="cisco")
dt = nb.dcim.device_types.create(manufacturer=mfr.id, model="C9300", slug="c9300")
site = nb.dcim.sites.create(name="NYC-DC1", slug="nyc-dc1")
role = nb.dcim.device_roles.create(name="Access", slug="access")
device = nb.dcim.devices.create(
    name="switch-01",
    device_type=dt.id,
    site=site.id,
    role=role.id
)
```

**With Diode (automatic dependency resolution):**

```python
# Just describe what you want - Diode handles the rest
device = Device(
    name="switch-01",
    device_type="C9300",
    manufacturer="Cisco",
    site="NYC-DC1",
    role="Access",
)
client.ingest([Entity(device=device)])
```

### Nested Objects with Full Control

```python
from netboxlabs.diode.sdk import DiodeClient, Entity
from netboxlabs.diode.sdk.ingester import Device, Site, DeviceType, Manufacturer

device = Device(
    name="router-01",
    device_type=DeviceType(
        model="ISR 4451",
        manufacturer=Manufacturer(name="Cisco"),
    ),
    site=Site(
        name="Chicago-DC",
        status="active",
        metadata={
            "region": "us-central",
            "tier": "tier-1",
        },
    ),
    role="Core Router",
    status="active",
)
```

### Bulk Ingestion

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Interface, Entity

discovered_devices = [
    {"name": "sw-01", "type": "Catalyst 9300", "site": "NYC"},
    {"name": "sw-02", "type": "Catalyst 9300", "site": "NYC"},
    {"name": "rtr-01", "type": "ISR 4451", "site": "NYC"},
]

with DiodeClient(
    target="grpc://diode.example.com:8080/diode",
    app_name="network-scanner",
    app_version="2.0.0",
) as client:
    entities = []

    for dev in discovered_devices:
        device = Device(
            name=dev["name"],
            device_type=dev["type"],
            manufacturer="Cisco",
            site=dev["site"],
            role="Network Device",
            status="active",
        )
        entities.append(Entity(device=device))

    # Ingest all entities in one call
    response = client.ingest(
        entities=entities,
        metadata={
            "scan_id": "discovery-2026-01-15",
            "source": "network_scanner",
        }
    )
```

### Metadata Support

Diode supports both entity-level and request-level metadata:

```python
# Entity-level metadata
device = Device(
    name="switch-01",
    device_type="Catalyst 9300",
    site="NYC-DC1",
    metadata={
        "discovered_by": "network_scanner",
        "scan_timestamp": "2026-01-15T10:30:00Z",
        "confidence_score": 0.95,
    },
)

# Request-level metadata
response = client.ingest(
    entities=[Entity(device=device)],
    metadata={
        "batch_id": "scan-2026-01-15-001",
        "source_system": "network_scanner",
    },
)
```

### Dry Run Mode

Test ingestion logic without contacting the server:

```python
from netboxlabs.diode.sdk import DiodeDryRunClient, Entity
from netboxlabs.diode.sdk.ingester import Device

with DiodeDryRunClient(app_name="my_app", output_dir="/tmp") as client:
    device = Device(name="test-switch", device_type="Test Type", site="Test Site")
    client.ingest([Entity(device=device)])
    # Creates /tmp/my_app_<timestamp>.json for review
```

Replay dry-run files:

```bash
diode-replay-dryrun \
  --file /tmp/my_app_92722156890707.json \
  --target grpc://diode.example.com:8080/diode \
  --app-name my-test-app \
  --app-version 0.0.1
```

### Authentication

Diode uses OAuth2 client credentials. Generate credentials in the NetBox UI under **Diode > Client Credentials**.

```python
import os

# Environment variables (recommended)
os.environ["DIODE_CLIENT_ID"] = "your-client-id"
os.environ["DIODE_CLIENT_SECRET"] = "your-client-secret"

with DiodeClient(
    target="grpcs://diode.example.com/diode",
    app_name="my-app",
    app_version="1.0.0",
) as client:
    # ...
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DIODE_CLIENT_ID` | OAuth2 client ID |
| `DIODE_CLIENT_SECRET` | OAuth2 client secret |
| `DIODE_CERT_FILE` | Path to custom CA certificate |
| `DIODE_SKIP_TLS_VERIFY` | Skip TLS verification (dev only) |
| `DIODE_SDK_LOG_LEVEL` | Log level (default: INFO) |
| `HTTPS_PROXY` / `HTTP_PROXY` | Proxy configuration |
| `NO_PROXY` | Hosts to bypass proxy |

### Supported Entity Types

Diode supports all major NetBox object types including:

- **DCIM**: Device, DeviceType, DeviceBay, Interface, ConsolePort, PowerPort, Rack, Site, Location, Manufacturer, Platform, Module, Cable
- **IPAM**: IPAddress, Prefix, VLAN, VLANGroup, VRF, ASN, Aggregate
- **Virtualization**: VirtualMachine, VMInterface, Cluster, ClusterType
- **Circuits**: Circuit, CircuitType, Provider, ProviderNetwork
- **Tenancy**: Tenant, TenantGroup, Contact, ContactRole
- **And more**: 70+ entity types total

### Diode Architecture

```
┌─────────────────┐     gRPC      ┌─────────────────┐     REST     ┌─────────────────┐
│  Your Script    │──────────────>│  Diode Server   │─────────────>│     NetBox      │
│  (Diode SDK)    │               │                 │              │                 │
└─────────────────┘               └─────────────────┘              └─────────────────┘
                                         │
                                         │ Resolves dependencies
                                         │ Creates missing objects
                                         │ Handles ordering
                                         ▼
```

### References

- [Diode Server](https://github.com/netboxlabs/diode)
- [Diode Python SDK](https://github.com/netboxlabs/diode-sdk-python)
- [Diode Go SDK](https://github.com/netboxlabs/diode-sdk-go)
- [Diode NetBox Plugin](https://github.com/netboxlabs/diode-netbox-plugin)
- [Getting Started Guide](https://github.com/netboxlabs/diode/blob/develop/GET_STARTED.md)

---

## Troubleshooting

### Common Issues

**Authentication Failures (401):**
- Token expired or revoked
- Wrong token format (v1 vs v2)
- Missing `Bearer` prefix for v2 tokens
- `API_TOKEN_PEPPERS` not configured for v2

**Permission Denied (403):**
- Token lacks required permissions
- IP restrictions blocking request
- Object-level permissions not granted

**Validation Errors (400):**
- Missing required fields
- Invalid foreign key references
- Constraint violations (duplicate name, etc.)
- Invalid data types

**Performance Issues:**
- Missing `?exclude=config_context`
- Using `q=` search with large datasets
- Unbounded GraphQL queries
- Missing pagination

### Debug Checklist

1. **Verify token format:**
   ```python
   # v1 format
   headers = {"Authorization": "Token xxx..."}

   # v2 format
   headers = {"Authorization": "Bearer nbt_xxx.yyy..."}
   ```

2. **Check response body for errors:**
   ```python
   response = requests.get(url, headers=headers)
   print(f"Status: {response.status_code}")
   print(f"Body: {response.text}")
   ```

3. **Use X-Request-ID for log correlation:**
   ```python
   import uuid
   request_id = str(uuid.uuid4())
   headers["X-Request-ID"] = request_id
   print(f"Check NetBox logs for request: {request_id}")
   ```

4. **Verify endpoint with OPTIONS:**
   ```python
   response = requests.options(url, headers=headers)
   print(response.json())
   ```

5. **Test with curl:**
   ```bash
   curl -H "Authorization: Bearer nbt_xxx.yyy" \
        -H "Content-Type: application/json" \
        "https://netbox.example.com/api/dcim/devices/?limit=1"
   ```

### Performance Debugging

```python
import time

def timed_request(session, url):
    """Measure request timing."""
    start = time.time()
    response = session.get(url)
    elapsed = time.time() - start

    print(f"URL: {url}")
    print(f"Status: {response.status_code}")
    print(f"Time: {elapsed:.2f}s")
    print(f"Size: {len(response.content)} bytes")

    return response

# Compare with and without config_context
timed_request(session, f"{API_URL}/dcim/devices/?limit=100")
timed_request(session, f"{API_URL}/dcim/devices/?limit=100&exclude=config_context")
```

### GraphQL Debugging

```python
def debug_graphql_query(netbox_url, token, query):
    """Execute GraphQL with detailed debugging."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    start = time.time()
    response = requests.post(
        f"{netbox_url}/graphql/",
        headers=headers,
        json={"query": query}
    )
    elapsed = time.time() - start

    print(f"Status: {response.status_code}")
    print(f"Time: {elapsed:.2f}s")

    result = response.json()

    if "errors" in result:
        print("ERRORS:")
        for error in result["errors"]:
            print(f"  - {error.get('message')}")
            if "locations" in error:
                print(f"    Location: {error['locations']}")

    if "data" in result:
        import json
        print("DATA:")
        print(json.dumps(result["data"], indent=2)[:1000])

    return result
```
