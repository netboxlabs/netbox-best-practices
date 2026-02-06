---
title: NetBox Branching Async Operations
impact: MEDIUM
category: integ
tags: [branching, async, jobs, polling]
netbox_version: "4.0+"
---

# integ-branch-async-operations: NetBox Branching Async Operations

> **Plugin Required:** This rule applies only when the [netbox-branching](https://github.com/netboxlabs/netbox-branching) plugin is installed.

## Rationale

Branch operations that modify significant data (sync, merge, revert) run asynchronously and return Job objects. Understanding how to poll for job completion is essential for reliable automation:

1. **Sync** - Pulls latest main changes into branch (async)
2. **Merge** - Applies branch changes to main (async)
3. **Revert** - Rolls back branch changes (async)

Each operation returns a Job object with a URL to poll for status. Jobs progress through states and may succeed or fail.

## Job Status Values

| Status | Description | Terminal? |
|--------|-------------|-----------|
| `pending` | Job queued, not yet started | No |
| `scheduled` | Job scheduled for execution | No |
| `running` | Job currently executing | No |
| `completed` | Job finished successfully | Yes |
| `errored` | Job failed with error | Yes |
| `failed` | Job failed | Yes |

## Incorrect Pattern

```python
# WRONG: Not handling async nature of branch operations
import requests

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

def merge_branch_wrong(branch_id):
    # WRONG: Assuming merge completes immediately
    merge_resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
        headers=HEADERS,
        json={"commit": True}
    )

    # This response is a Job, not the merge result!
    result = merge_resp.json()

    # WRONG: Assuming merge is complete
    print("Merge complete!")  # Not true - merge is still running!

    # WRONG: Immediately trying to read merged data
    # Data may not be visible yet
    devices = requests.get(f"{NETBOX_URL}/api/dcim/devices/").json()
```

**Problems with this approach:**
- Merge/sync/revert operations return Jobs, not immediate results
- Assuming completion leads to race conditions
- No error handling for failed jobs
- Data may not be available until job completes

## Correct Pattern

```python
# CORRECT: Proper async job handling
import requests
import time
from typing import Optional

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

def poll_job(job_url: str, timeout: int = 300, interval: int = 2) -> dict:
    """
    Poll a job URL until completion or timeout.

    Args:
        job_url: Full URL to the job resource
        timeout: Maximum seconds to wait
        interval: Seconds between polls

    Returns:
        Completed job object

    Raises:
        RuntimeError: If job fails or errors
        TimeoutError: If job doesn't complete in time
    """
    start = time.time()

    while time.time() - start < timeout:
        resp = requests.get(job_url, headers=HEADERS)
        resp.raise_for_status()
        job = resp.json()

        status = job["status"]["value"]

        if status == "completed":
            return job

        if status in ("errored", "failed"):
            error_msg = job.get("data", {}).get("error", "Unknown error")
            raise RuntimeError(f"Job failed: {error_msg}")

        # Job still in progress
        time.sleep(interval)

    raise TimeoutError(f"Job did not complete within {timeout} seconds")


def merge_branch(branch_id: int, dry_run: bool = False) -> dict:
    """
    Merge a branch to main with proper async handling.

    Args:
        branch_id: Branch numeric ID
        dry_run: If True, validate without committing

    Returns:
        Completed job object
    """
    resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
        headers=HEADERS,
        json={"commit": not dry_run}
    )
    resp.raise_for_status()

    job = resp.json()
    print(f"Merge job started: {job['url']}")

    # Poll until complete
    return poll_job(job["url"])


def sync_branch(branch_id: int, dry_run: bool = False) -> dict:
    """
    Sync main changes into branch with proper async handling.

    Args:
        branch_id: Branch numeric ID
        dry_run: If True, show changes without applying

    Returns:
        Completed job object
    """
    resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/sync/",
        headers=HEADERS,
        json={"commit": not dry_run}
    )
    resp.raise_for_status()

    job = resp.json()
    print(f"Sync job started: {job['url']}")

    return poll_job(job["url"])


def revert_branch(branch_id: int, dry_run: bool = False) -> dict:
    """
    Revert branch changes with proper async handling.

    Args:
        branch_id: Branch numeric ID
        dry_run: If True, show what would be reverted

    Returns:
        Completed job object
    """
    resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/revert/",
        headers=HEADERS,
        json={"commit": not dry_run}
    )
    resp.raise_for_status()

    job = resp.json()
    print(f"Revert job started: {job['url']}")

    return poll_job(job["url"])


# Usage example
try:
    # Dry-run first to validate
    dry_run_job = merge_branch(branch_id=42, dry_run=True)
    print(f"Dry-run passed, changes: {dry_run_job.get('data', {})}")

    # Actual merge
    merge_job = merge_branch(branch_id=42, dry_run=False)
    print("Merge completed successfully!")

except RuntimeError as e:
    print(f"Operation failed: {e}")
except TimeoutError as e:
    print(f"Operation timed out: {e}")
```

**Benefits:**
- Proper handling of async job lifecycle
- Clear timeout handling prevents hanging scripts
- Dry-run validation before committing
- Error details extracted from job data

## Async Pattern with Callbacks

For long-running operations or event-driven architectures:

```python
import requests
import threading
import time
from typing import Callable, Optional

def poll_job_async(
    job_url: str,
    on_complete: Callable[[dict], None],
    on_error: Callable[[Exception], None],
    timeout: int = 300,
    interval: int = 2
) -> threading.Thread:
    """
    Poll job in background thread, call callbacks on completion.

    Args:
        job_url: Full URL to the job resource
        on_complete: Called with job object on success
        on_error: Called with exception on failure
        timeout: Maximum seconds to wait
        interval: Seconds between polls

    Returns:
        Thread object (already started)
    """
    def poll():
        try:
            result = poll_job(job_url, timeout, interval)
            on_complete(result)
        except Exception as e:
            on_error(e)

    thread = threading.Thread(target=poll, daemon=True)
    thread.start()
    return thread


# Usage with callbacks
def handle_merge_complete(job):
    print(f"Merge completed: {job['id']}")
    # Trigger downstream workflows...

def handle_merge_error(error):
    print(f"Merge failed: {error}")
    # Alert, retry, or rollback...

# Start merge and poll in background
merge_resp = requests.post(
    f"{NETBOX_URL}/api/plugins/branching/branches/42/merge/",
    headers=HEADERS,
    json={"commit": True}
)
job = merge_resp.json()

thread = poll_job_async(
    job["url"],
    on_complete=handle_merge_complete,
    on_error=handle_merge_error
)

# Continue with other work...
print("Merge started, continuing with other tasks...")

# Wait for completion if needed
thread.join()
```

## Merge Strategies

When merging, you can specify conflict resolution strategies:

```python
def merge_with_strategy(branch_id: int, strategy: str = "ours") -> dict:
    """
    Merge with specified conflict strategy.

    Args:
        branch_id: Branch numeric ID
        strategy: Conflict resolution - "ours" (branch wins) or "theirs" (main wins)

    Returns:
        Completed job object
    """
    resp = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
        headers=HEADERS,
        json={
            "commit": True,
            "strategy": strategy  # How to resolve conflicts
        }
    )
    resp.raise_for_status()
    return poll_job(resp.json()["url"])


# Example: Branch changes take priority
merge_job = merge_with_strategy(42, strategy="ours")
```

## Checking Job Progress

For long-running operations, provide progress feedback:

```python
def poll_job_with_progress(job_url: str, timeout: int = 300) -> dict:
    """Poll job with progress output."""
    start = time.time()
    last_status = None

    while time.time() - start < timeout:
        resp = requests.get(job_url, headers=HEADERS)
        job = resp.json()

        status = job["status"]["value"]

        # Print status changes
        if status != last_status:
            elapsed = int(time.time() - start)
            print(f"[{elapsed}s] Job status: {status}")
            last_status = status

        if status == "completed":
            return job

        if status in ("errored", "failed"):
            raise RuntimeError(f"Job failed: {job.get('data', {}).get('error')}")

        time.sleep(2)

    raise TimeoutError(f"Job timed out after {timeout}s")


# Usage
job = poll_job_with_progress(job_url, timeout=600)
# Output:
# [0s] Job status: pending
# [2s] Job status: running
# [45s] Job status: completed
```

## Exceptions

- **Small branches**: Very small merges may complete almost instantly, but always poll anyway
- **Dry-run only**: Dry-runs still return jobs but typically complete faster
- **Network issues**: Consider retry logic for transient poll failures

## Related Rules

- [integ-branch-api-workflow](./integ-branch-api-workflow.md) - Complete branch lifecycle
- [integ-branch-context-header](./integ-branch-context-header.md) - Branch context switching

## References

- [NetBox Branching Plugin](https://github.com/netboxlabs/netbox-branching)
- [NetBox Branching Documentation](https://netboxlabs.com/docs/netbox-branching/)
- [NetBox Jobs Framework](https://netboxlabs.com/docs/netbox/en/stable/features/background-jobs/)
