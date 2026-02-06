---
title: NetBox Branching API Workflow
impact: HIGH
category: integ
tags: [branching, change-management, workflow, api]
netbox_version: "4.0+"
---

# integ-branch-api-workflow: NetBox Branching API Workflow

> **Plugin Required:** This rule applies only when the [netbox-branching](https://github.com/netboxlabs/netbox-branching) plugin is installed.

## Rationale

The NetBox Branching plugin enables change management workflows where modifications are made in isolated branches before being merged to main. Understanding the complete branch lifecycle is essential for building reliable automation:

1. **Create branch** - Initialize an isolated workspace
2. **Wait for provisioning** - Branch schema must be READY before use
3. **Make changes** - Work in branch context using the `X-NetBox-Branch` header
4. **Sync (optional)** - Pull latest main changes into your branch
5. **Merge** - Apply branch changes to main

This workflow prevents conflicts, enables review processes, and provides rollback capability via revert operations.

## Branch Lifecycle States

| State | Description | API Operations Allowed |
|-------|-------------|----------------------|
| `NEW` | Just created, not yet provisioned | None |
| `PROVISIONING` | Schema being created | None |
| `READY` | Ready for use | Read, write, sync, merge, revert |
| `SYNCING` | Pulling changes from main | Read only |
| `MERGING` | Applying changes to main | Read only |
| `REVERTING` | Rolling back changes | Read only |
| `MERGED` | Successfully merged | Read only (historical) |
| `ARCHIVED` | No longer active | None |

## Incorrect Pattern

```python
# WRONG: Not waiting for branch to be ready before use
import requests

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

def quick_branch_workflow():
    # Create branch
    branch_resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/",
        headers=HEADERS,
        json={"name": "feature-update", "description": "Network updates"}
    )
    branch = branch_resp.json()

    # WRONG: Using branch immediately without checking state
    headers_with_branch = {**HEADERS, "X-NetBox-Branch": branch["schema_id"]}

    # This may fail - branch not yet provisioned!
    requests.post(
        f"{NETBOX_URL}/api/dcim/devices/",
        headers=headers_with_branch,
        json={"name": "new-switch", "site": 1, "device_type": 1, "role": 1}
    )
```

**Problems with this approach:**
- Branch may still be in `PROVISIONING` state
- API calls will fail with errors about invalid branch context
- No handling of async operations (sync/merge return Jobs, not immediate results)

## Correct Pattern

```python
# CORRECT: Complete branch workflow with proper state handling
import requests
import time

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

def wait_for_branch_ready(branch_id, timeout=120):
    """Poll until branch is READY."""
    start = time.time()
    while time.time() - start < timeout:
        resp = requests.get(
            f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/",
            headers=HEADERS
        )
        branch = resp.json()
        if branch["status"]["value"] == "ready":
            return branch
        if branch["status"]["value"] in ("archived", "merged"):
            raise RuntimeError(f"Branch in terminal state: {branch['status']['value']}")
        time.sleep(2)
    raise TimeoutError("Branch did not become ready in time")


def wait_for_job(job_url, timeout=300):
    """Poll until job completes."""
    start = time.time()
    while time.time() - start < timeout:
        resp = requests.get(job_url, headers=HEADERS)
        job = resp.json()
        if job["status"]["value"] == "completed":
            return job
        if job["status"]["value"] in ("errored", "failed"):
            raise RuntimeError(f"Job failed: {job.get('error', 'Unknown error')}")
        time.sleep(2)
    raise TimeoutError("Job did not complete in time")


def branch_workflow():
    # Step 1: Create branch
    branch_resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/",
        headers=HEADERS,
        json={
            "name": "feature-network-updates",
            "description": "Q1 network infrastructure updates"
        }
    )
    branch_resp.raise_for_status()
    branch = branch_resp.json()
    branch_id = branch["id"]
    schema_id = branch["schema_id"]  # 8-char ID for X-NetBox-Branch header
    print(f"Created branch: {branch['name']} (schema_id: {schema_id})")

    # Step 2: Wait for branch to be READY
    branch = wait_for_branch_ready(branch_id)
    print(f"Branch ready: {branch['status']['label']}")

    # Step 3: Make changes in branch context
    branch_headers = {**HEADERS, "X-NetBox-Branch": schema_id}

    # Create a device in the branch
    device_resp = requests.post(
        f"{NETBOX_URL}/api/dcim/devices/",
        headers=branch_headers,
        json={
            "name": "new-switch-01",
            "site": 1,
            "device_type": 1,
            "role": 1,
            "status": "planned"
        }
    )
    device_resp.raise_for_status()
    print(f"Created device in branch: {device_resp.json()['name']}")

    # Step 4: Optional - sync latest main changes into branch
    sync_resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/sync/",
        headers=HEADERS,
        json={"commit": True}  # Use False for dry-run
    )
    if sync_resp.status_code == 200:
        job = sync_resp.json()
        wait_for_job(job["url"])
        print("Synced latest main changes into branch")

    # Step 5: Review changes before merge (optional but recommended)
    diff_resp = requests.get(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/diff/",
        headers=HEADERS
    )
    changes = diff_resp.json()
    print(f"Changes to merge: {len(changes)} modifications")

    # Step 6: Merge branch to main
    merge_resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
        headers=HEADERS,
        json={"commit": True}  # Use False for dry-run validation
    )
    merge_resp.raise_for_status()
    job = merge_resp.json()
    wait_for_job(job["url"])
    print("Branch merged successfully!")

    return branch


if __name__ == "__main__":
    branch_workflow()
```

**Benefits:**
- Proper state transitions prevent race conditions
- Timeout handling prevents infinite waits
- Dry-run capability (`commit: False`) for validation before committing
- Clear error handling for failed jobs
- ChangeDiff review before merge

## Dry-Run Validation

Always validate complex changes with dry-run before committing:

```python
def merge_with_validation(branch_id):
    """Dry-run first, then merge if clean."""

    # Dry-run to check for conflicts
    dry_run_resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
        headers=HEADERS,
        json={"commit": False}  # Dry-run mode
    )
    dry_run_resp.raise_for_status()
    dry_run_job = dry_run_resp.json()

    # Wait for dry-run to complete
    job = wait_for_job(dry_run_job["url"])

    # Check for conflicts in job output
    if job.get("data", {}).get("conflicts"):
        print(f"Conflicts detected: {job['data']['conflicts']}")
        return False

    # No conflicts - proceed with actual merge
    merge_resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
        headers=HEADERS,
        json={"commit": True}
    )
    merge_resp.raise_for_status()
    job = wait_for_job(merge_resp.json()["url"])
    return True
```

## pynetbox Example

pynetbox supports branching via its plugin API:

```python
import pynetbox
import time

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Create branch
branch = nb.plugins.branching.branches.create(
    name="feature-updates",
    description="Infrastructure updates"
)

# Wait for ready
while branch.status.value != "ready":
    time.sleep(2)
    branch = nb.plugins.branching.branches.get(branch.id)

# Work in branch context - pynetbox doesn't directly support
# X-NetBox-Branch header, so use requests for branch operations
# or create a custom session wrapper (see integ-branch-context-header)
```

## Exceptions

- **Simple changes**: For single object updates that don't need review, direct API may be simpler
- **Emergency changes**: In urgent situations, you may need to bypass branching workflow
- **Read-only operations**: Branching is for writes; reads don't require branch context

## Related Rules

- [integ-branch-context-header](./integ-branch-context-header.md) - Working in branch context
- [integ-branch-async-operations](./integ-branch-async-operations.md) - Handling async jobs

## References

- [NetBox Branching Plugin](https://github.com/netboxlabs/netbox-branching)
- [NetBox Branching Documentation](https://netboxlabs.com/docs/netbox-branching/)
