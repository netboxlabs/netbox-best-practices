---
title: NetBox Branch Context Header
impact: HIGH
category: integ
tags: [branching, context, header, api]
netbox_version: "4.0+"
---

# integ-branch-context-header: NetBox Branch Context Header

> **Plugin Required:** This rule applies only when the [netbox-branching](https://github.com/netboxlabs/netbox-branching) plugin is installed.

## Rationale

When working with NetBox Branching, you switch between main and branch contexts using the `X-NetBox-Branch` header. This header tells NetBox which schema to use for the request:

- **Without header**: Operations target main database
- **With header**: Operations target branch schema

The header value is the `schema_id` (8-character identifier) from the branch object, NOT the branch name or numeric ID. This is a common source of confusion.

## Header Format

```
X-NetBox-Branch: {schema_id}
```

Where `schema_id` is the 8-character alphanumeric identifier returned when creating a branch:

```python
branch = {
    "id": 42,              # Numeric ID - NOT for header
    "name": "feature-x",   # Human name - NOT for header
    "schema_id": "a1b2c3d4" # USE THIS for X-NetBox-Branch header
}
```

## Incorrect Pattern

```python
# WRONG: Using wrong identifier for branch header
import requests

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

# Get branch
branch_resp = requests.get(
    f"{NETBOX_URL}/api/plugins/branching/branches/42/",
    headers=HEADERS
)
branch = branch_resp.json()

# WRONG: Using numeric ID
bad_headers_1 = {**HEADERS, "X-NetBox-Branch": str(branch["id"])}  # 42 - WRONG

# WRONG: Using branch name
bad_headers_2 = {**HEADERS, "X-NetBox-Branch": branch["name"]}  # "feature-x" - WRONG

# WRONG: Forgetting header entirely for branch operations
requests.post(
    f"{NETBOX_URL}/api/dcim/devices/",
    headers=HEADERS,  # No branch header - goes to main!
    json={...}
)
```

**Problems with this approach:**
- Using wrong identifier causes "Invalid branch" errors
- Missing header causes changes to go to main instead of branch
- No way to know which context you're operating in without explicit tracking

## Correct Pattern

```python
# CORRECT: Using schema_id for branch context
import requests

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

def get_branch_headers(schema_id):
    """Create headers with branch context."""
    return {**HEADERS, "X-NetBox-Branch": schema_id}


# Create branch and extract schema_id
branch_resp = requests.post(
    f"{NETBOX_URL}/api/plugins/branching/branches/",
    headers=HEADERS,
    json={"name": "feature-updates", "description": "Q1 updates"}
)
branch = branch_resp.json()
schema_id = branch["schema_id"]  # e.g., "a1b2c3d4"

# Wait for branch ready (see integ-branch-api-workflow)
# ...

# Create headers for branch context
branch_headers = get_branch_headers(schema_id)

# Operations in branch context
device_resp = requests.post(
    f"{NETBOX_URL}/api/dcim/devices/",
    headers=branch_headers,
    json={"name": "new-switch", "site": 1, "device_type": 1, "role": 1}
)

# Read from branch - device exists only in branch
branch_device = requests.get(
    f"{NETBOX_URL}/api/dcim/devices/",
    headers=branch_headers,
    params={"name": "new-switch"}
).json()  # Found in branch

# Read from main - device doesn't exist yet
main_device = requests.get(
    f"{NETBOX_URL}/api/dcim/devices/",
    headers=HEADERS,  # No branch header = main
    params={"name": "new-switch"}
).json()  # Not found in main (until merged)
```

**Benefits:**
- Clear separation between main and branch contexts
- Explicit control over which schema receives changes
- Easy to verify context before critical operations

## Session Wrapper Pattern

For complex workflows, wrap the branch context in a session helper:

```python
import requests

class BranchSession:
    """Context manager for working in a NetBox branch."""

    def __init__(self, netbox_url, token, schema_id):
        self.netbox_url = netbox_url.rstrip("/")
        self.schema_id = schema_id
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
            "X-NetBox-Branch": schema_id,
        })

    def get(self, endpoint, **kwargs):
        """GET request in branch context."""
        url = f"{self.netbox_url}/api/{endpoint.lstrip('/')}"
        return self.session.get(url, **kwargs)

    def post(self, endpoint, **kwargs):
        """POST request in branch context."""
        url = f"{self.netbox_url}/api/{endpoint.lstrip('/')}"
        return self.session.post(url, **kwargs)

    def patch(self, endpoint, **kwargs):
        """PATCH request in branch context."""
        url = f"{self.netbox_url}/api/{endpoint.lstrip('/')}"
        return self.session.patch(url, **kwargs)

    def delete(self, endpoint, **kwargs):
        """DELETE request in branch context."""
        url = f"{self.netbox_url}/api/{endpoint.lstrip('/')}"
        return self.session.delete(url, **kwargs)


# Usage
branch_session = BranchSession(
    netbox_url="https://netbox.example.com",
    token=TOKEN,
    schema_id="a1b2c3d4"
)

# All operations use branch context
devices = branch_session.get("dcim/devices/", params={"site": "nyc"}).json()
new_device = branch_session.post("dcim/devices/", json={...}).json()
```

## Comparing Branch and Main

A common pattern is comparing what exists in branch vs main:

```python
def compare_branch_main(endpoint, params, schema_id):
    """Compare query results between branch and main."""

    # Query main
    main_resp = requests.get(
        f"{NETBOX_URL}/api/{endpoint}/",
        headers=HEADERS,
        params=params
    )
    main_results = main_resp.json()["results"]

    # Query branch
    branch_headers = {**HEADERS, "X-NetBox-Branch": schema_id}
    branch_resp = requests.get(
        f"{NETBOX_URL}/api/{endpoint}/",
        headers=branch_headers,
        params=params
    )
    branch_results = branch_resp.json()["results"]

    # Find differences
    main_ids = {r["id"] for r in main_results}
    branch_ids = {r["id"] for r in branch_results}

    return {
        "added_in_branch": branch_ids - main_ids,
        "deleted_in_branch": main_ids - branch_ids,
        "in_both": main_ids & branch_ids,
    }


# Example: Find new devices in branch
diff = compare_branch_main("dcim/devices", {"site": "nyc"}, schema_id)
print(f"New devices in branch: {diff['added_in_branch']}")
```

## GraphQL with Branch Context

The branch header works with GraphQL queries too:

```python
def graphql_in_branch(query, variables, schema_id):
    """Execute GraphQL query in branch context."""
    headers = {
        "Authorization": f"Bearer {TOKEN}",
        "Content-Type": "application/json",
        "X-NetBox-Branch": schema_id,
    }

    resp = requests.post(
        f"{NETBOX_URL}/graphql/",
        headers=headers,
        json={"query": query, "variables": variables}
    )
    return resp.json()


# Query devices in branch
query = """
query BranchDevices($site_id: Int!) {
    device_list(filters: {site_id: $site_id}, limit: 100) {
        name
        status
        primary_ip4 { address }
    }
}
"""

result = graphql_in_branch(query, {"site_id": 1}, schema_id)
```

## Exceptions

- **Read-only queries**: For queries that don't need isolation, you can query main directly
- **Cross-branch comparison**: Some operations require querying both contexts
- **Merge/sync operations**: These use the branch numeric ID, not schema_id, in the URL path

## Related Rules

- [integ-branch-api-workflow](./integ-branch-api-workflow.md) - Complete branch lifecycle
- [integ-branch-async-operations](./integ-branch-async-operations.md) - Async job handling

## References

- [NetBox Branching Plugin](https://github.com/netboxlabs/netbox-branching)
- [NetBox Branching Documentation](https://netboxlabs.com/docs/netbox-branching/)
