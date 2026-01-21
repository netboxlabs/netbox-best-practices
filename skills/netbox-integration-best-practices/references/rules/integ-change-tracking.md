---
title: Query Object Changes for Audit Trails
impact: LOW
category: integ
tags: [integration, audit, changes, tracking]
netbox_version: "4.4+"
---

# integ-change-tracking: Query Object Changes for Audit Trails

## Rationale

NetBox tracks all object changes. Query the change log for audit trails, compliance reporting, or catching up on missed events.

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

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

# Get all changes since a timestamp
recent = nb.extras.object_changes.filter(
    time_after="2024-01-01T00:00:00Z"
)
```

## Use Cases

- Audit compliance reporting
- Catching up after webhook outage
- Debugging integration issues
- Change notification systems

## Related Rules

- [integ-event-driven](./integ-event-driven.md) - Real-time events
- [sec-audit-logging](./sec-audit-logging.md) - Client-side logging
