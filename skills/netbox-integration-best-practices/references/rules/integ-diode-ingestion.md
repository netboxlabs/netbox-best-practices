---
title: Use Diode for Simplified Data Ingestion
impact: HIGH
category: integ
tags: [diode, ingestion, bulk, dependency-resolution]
netbox_version: "4.2.3+"
---

# integ-diode-ingestion: Use Diode for Simplified Data Ingestion

## Rationale

For data ingestion scenarios (network discovery, bulk imports, automated population), use [Diode](https://github.com/netboxlabs/diode) instead of the direct REST API. Diode dramatically simplifies data ingestion by:

1. **Automatic dependency resolution**: Specify objects by name, not ID—Diode looks them up or creates them
2. **No dependency order required**: You don't need to create manufacturers before device types, sites before devices, etc.
3. **Efficient gRPC protocol**: Streamlined data transfer
4. **Built-in error handling**: Clear error responses for debugging

This eliminates the most complex and error-prone aspects of NetBox API integrations—managing object creation order and resolving IDs for related objects.

## Incorrect Pattern

```python
# WRONG: Complex direct API usage with manual dependency management
import requests

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

def create_device_with_dependencies(device_data):
    """Create device with all dependencies - tedious and error-prone!"""

    # Step 1: Create or get manufacturer
    mfr_resp = requests.get(
        f"{NETBOX_URL}/api/dcim/manufacturers/",
        headers=HEADERS,
        params={"name": device_data["manufacturer"]}
    )
    if not mfr_resp.json()["results"]:
        mfr_resp = requests.post(
            f"{NETBOX_URL}/api/dcim/manufacturers/",
            headers=HEADERS,
            json={"name": device_data["manufacturer"], "slug": slugify(device_data["manufacturer"])}
        )
        manufacturer_id = mfr_resp.json()["id"]
    else:
        manufacturer_id = mfr_resp.json()["results"][0]["id"]

    # Step 2: Create or get device type (needs manufacturer ID)
    dt_resp = requests.get(
        f"{NETBOX_URL}/api/dcim/device-types/",
        headers=HEADERS,
        params={"model": device_data["device_type"], "manufacturer_id": manufacturer_id}
    )
    if not dt_resp.json()["results"]:
        dt_resp = requests.post(
            f"{NETBOX_URL}/api/dcim/device-types/",
            headers=HEADERS,
            json={
                "model": device_data["device_type"],
                "slug": slugify(device_data["device_type"]),
                "manufacturer": manufacturer_id
            }
        )
        device_type_id = dt_resp.json()["id"]
    else:
        device_type_id = dt_resp.json()["results"][0]["id"]

    # Step 3: Create or get site
    site_resp = requests.get(
        f"{NETBOX_URL}/api/dcim/sites/",
        headers=HEADERS,
        params={"name": device_data["site"]}
    )
    # ... and on and on for site, role, platform, rack, etc.

    # Step 4: Create or get device role
    # ... more lookups and creates

    # Step 5: Finally create the device with all those IDs
    device_resp = requests.post(
        f"{NETBOX_URL}/api/dcim/devices/",
        headers=HEADERS,
        json={
            "name": device_data["name"],
            "device_type": device_type_id,
            "site": site_id,
            "role": role_id,
            # ...
        }
    )
    return device_resp.json()
```

**Problems with this approach:**
- Complex code with many API calls per device
- Must handle lookup-or-create logic for every dependency
- Must understand and maintain correct dependency order
- Error handling is complex (rollback on partial failure?)
- Performance degrades with many devices (N+1 API calls)
- Race conditions possible in parallel scenarios

## Correct Pattern

```python
# CORRECT: Use Diode - specify dependencies by name, Diode handles the rest
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Entity

with DiodeClient(
    target="https://<your-diode-host>:443/diode",
    app_name="network-discovery",
    app_version="1.0.0",
) as client:
    # Just describe what you want - Diode resolves or creates dependencies
    device = Device(
        name="switch-nyc-01",
        device_type="Cisco Catalyst 9300",  # By name, not ID!
        manufacturer="Cisco",                # Auto-created if missing
        site="NYC-DC1",                      # Auto-created if missing
        role="Access Switch",                # Auto-created if missing
        serial="ABC123456",
        status="active",
        tags=["production", "network"],
    )

    response = client.ingest([Entity(device=device)])

    if response.errors:
        print(f"Errors: {response.errors}")
```

**Benefits:**
- Single API call per batch of entities
- No ID lookups required—use names directly
- No dependency order management—Diode handles it
- Clean, simple code that's easy to maintain
- Built-in error handling with clear responses

> **Note on uniqueness:** Name-based references resolve differently per model. Devices are unique by name + site. Interfaces are unique by name + device. Check NetBox model documentation for uniqueness constraints when troubleshooting resolution issues.

## Bulk Ingestion Example

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Interface, IPAddress, Entity

# Discovery data from network scan
discovered_devices = [
    {"name": "sw-01", "type": "Catalyst 9300", "site": "NYC", "interfaces": ["Gi0/1", "Gi0/2"]},
    {"name": "sw-02", "type": "Catalyst 9300", "site": "NYC", "interfaces": ["Gi0/1", "Gi0/2"]},
    {"name": "rtr-01", "type": "ISR 4451", "site": "NYC", "interfaces": ["Gi0/0/0", "Gi0/0/1"]},
]

with DiodeClient(
    target="https://<your-diode-host>:443/diode",
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

        # Add interfaces for this device
        for iface_name in dev["interfaces"]:
            iface = Interface(
                device=dev["name"],  # Reference by name
                name=iface_name,
                type="1000base-t",
            )
            entities.append(Entity(interface=iface))

    # Ingest all entities in one call
    response = client.ingest(
        entities=entities,
        metadata={
            "scan_id": "discovery-2026-01-15",
            "source": "network_scanner",
        }
    )

    if response.errors:
        for error in response.errors:
            print(f"Error: {error}")
    else:
        print(f"Successfully ingested {len(entities)} entities")
```

## Nested Object Example

For full control over nested object attributes:

```python
from netboxlabs.diode.sdk import DiodeClient, Entity
from netboxlabs.diode.sdk.ingester import Device, Site, DeviceType, Manufacturer

with DiodeClient(
    target="https://<your-diode-host>:443/diode",
    app_name="my-app",
    app_version="1.0.0",
) as client:
    device = Device(
        name="router-01",
        # Full nested object with custom attributes
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

    response = client.ingest([Entity(device=device)])
```

## Dry Run for Testing

Test ingestion logic without contacting the server:

```python
from netboxlabs.diode.sdk import DiodeDryRunClient, Entity
from netboxlabs.diode.sdk.ingester import Device

with DiodeDryRunClient(app_name="my_app", output_dir="/tmp") as client:
    device = Device(name="test-switch", device_type="Test Type", site="Test Site")
    client.ingest([Entity(device=device)])
    # Creates /tmp/my_app_<timestamp>.json for review
```

Replay later:

```bash
diode-replay-dryrun \
  --file /tmp/my_app_92722156890707.json \
  --target https://<your-diode-host>:443/diode \
  --app-name my-test-app \
  --app-version 0.0.1
```

## Exceptions

- **Reading data**: Use REST/GraphQL API for queries—Diode is for writing only
- **Single object CRUD**: For simple create/update/delete of single objects, direct API may be simpler
- **Complex filtered searches**: Use REST/GraphQL API for complex queries
- **Real-time lookups**: Use REST API for lookups during processing
- **Version constraints**: Diode requires NetBox 4.2.3+ and Diode server deployment

## When to Use Diode vs Direct API

| Scenario | Recommendation |
|----------|----------------|
| Network discovery pushing data | **Diode** |
| Bulk data migrations | **Diode** |
| Scripts creating many related objects | **Diode** |
| Reading/querying NetBox data | REST/GraphQL |
| Single object operations | REST API |
| Complex filtered searches | REST/GraphQL |

## Related Rules

- [data-dependency-order](./data-dependency-order.md) - Manual approach Diode replaces
- [integ-pynetbox-client](./integ-pynetbox-client.md) - pynetbox for direct API access
- [rest-list-endpoint-bulk-ops](./rest-list-endpoint-bulk-ops.md) - Direct API bulk operations

## References

- [Diode Server](https://github.com/netboxlabs/diode)
- [Diode Python SDK](https://github.com/netboxlabs/diode-sdk-python)
- [Diode NetBox Plugin](https://github.com/netboxlabs/diode-netbox-plugin)
- [Getting Started Guide](https://github.com/netboxlabs/diode/blob/develop/GET_STARTED.md)
