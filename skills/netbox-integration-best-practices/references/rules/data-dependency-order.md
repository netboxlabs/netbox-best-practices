---
title: Create Objects in Correct Dependency Order
impact: CRITICAL
category: data
tags: [data-model, population, dependencies, ordering]
netbox_version: "4.4+"
---

# data-dependency-order: Create Objects in Correct Dependency Order

## Rationale

NetBox enforces referential integrity. A child object cannot be created until its parent exists. Attempting to create objects out of order results in validation errors referencing non-existent foreign keys.

Understanding the dependency graph is essential for:
- Initial NetBox population scripts
- Data migration from other systems
- Automated infrastructure provisioning
- CI/CD pipeline integrations

## Incorrect Pattern

```python
# WRONG: Creating device before its dependencies exist
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# This will FAIL - device_type, role, and site don't exist yet
device = nb.dcim.devices.create(
    name="switch-01",
    device_type=1,      # Error: device type not found
    role=1,             # Error: role not found
    site=1              # Error: site not found
)
```

**Problems with this approach:**
- Foreign key validation fails
- No device type, role, or site with these IDs
- Error message: "Invalid pk - object does not exist"

## Correct Pattern

```python
# CORRECT: Create dependencies before dependents
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Step 1: Organization layer (no dependencies)
region = nb.dcim.regions.create(
    name="North America",
    slug="na"
)

# Step 2: Site (depends on region, but region is optional)
site = nb.dcim.sites.create(
    name="NYC-DC1",
    slug="nyc-dc1",
    region=region.id,
    status="active"
)

# Step 3: DCIM prerequisites
manufacturer = nb.dcim.manufacturers.create(
    name="Cisco",
    slug="cisco"
)

device_type = nb.dcim.device_types.create(
    manufacturer=manufacturer.id,  # Manufacturer must exist
    model="Catalyst 9300",
    slug="c9300"
)

role = nb.dcim.device_roles.create(
    name="Access Switch",
    slug="access-switch",
    color="00ff00"
)

# Step 4: Now device can be created (all dependencies exist)
device = nb.dcim.devices.create(
    name="switch-01",
    device_type=device_type.id,
    role=role.id,
    site=site.id,
    status="active"
)

# Step 5: Device-dependent objects
interface = nb.dcim.interfaces.create(
    device=device.id,  # Device must exist
    name="GigabitEthernet0/1",
    type="1000base-t"
)
```

**Benefits:**
- All foreign key references are valid
- No validation errors
- Reproducible population scripts
- Clear data lineage

## Complete Dependency Order

```
1. ORGANIZATION (no dependencies)
   ├── Tenant Groups
   ├── Tenants (optional: Tenant Group)
   ├── Regions
   ├── Site Groups
   └── Contact Groups

2. SITES AND LOCATIONS
   ├── Sites (optional: Region, Site Group, Tenant)
   └── Locations (requires: Site, optional: parent Location)

3. DCIM PREREQUISITES
   ├── Manufacturers
   ├── Device Types (requires: Manufacturer)
   ├── Module Types (requires: Manufacturer)
   ├── Platforms
   ├── Device Roles
   └── Rack Roles

4. RACKS
   └── Racks (requires: Site, optional: Location, Rack Role, Tenant)

5. DEVICES
   ├── Devices (requires: Device Type, Role, Site; optional: Rack, Location)
   ├── Modules (requires: Device, Module Type)
   └── Interfaces, Ports (requires: Device)

6. IPAM PREREQUISITES
   ├── RIRs
   ├── VRFs (optional: Tenant)
   ├── Route Targets
   └── VLAN Groups (optional: Site)

7. IPAM OBJECTS
   ├── Aggregates (requires: RIR)
   ├── Prefixes (optional: VRF, Site, VLAN, Tenant)
   ├── IP Ranges (optional: VRF, Tenant)
   ├── IP Addresses (optional: VRF, Tenant, Interface)
   └── VLANs (optional: VLAN Group, Site, Tenant)

8. VIRTUALIZATION
   ├── Cluster Types
   ├── Cluster Groups
   ├── Clusters (requires: Cluster Type, optional: Site)
   ├── Virtual Machines (requires: Cluster)
   └── VM Interfaces (requires: Virtual Machine)

9. CIRCUITS
   ├── Providers
   ├── Provider Accounts (requires: Provider)
   ├── Provider Networks (requires: Provider)
   ├── Circuit Types
   └── Circuits (requires: Provider, Type)

10. CONNECTIONS (last)
    ├── Cables (requires: endpoints)
    ├── Wireless Links (requires: interfaces)
    └── Circuit Terminations (requires: Circuit, Site)
```

## Idempotent Population Script

```python
def get_or_create(endpoint, defaults, **lookup):
    """Get existing object or create new one."""
    existing = endpoint.get(**lookup)
    if existing:
        return existing, False

    data = {**lookup, **defaults}
    return endpoint.create(**data), True


# Idempotent population
site, created = get_or_create(
    nb.dcim.sites,
    defaults={"status": "active"},
    name="NYC-DC1",
    slug="nyc-dc1"
)

manufacturer, created = get_or_create(
    nb.dcim.manufacturers,
    defaults={},
    name="Cisco",
    slug="cisco"
)

device_type, created = get_or_create(
    nb.dcim.device_types,
    defaults={"manufacturer": manufacturer.id},
    model="Catalyst 9300",
    slug="c9300"
)
```

## Alternative: Use Diode for Automatic Dependency Resolution

For high-volume data ingestion, consider using [Diode](https://github.com/netboxlabs/diode) instead. **Diode eliminates the need for manual dependency ordering**—you specify objects by name, and Diode resolves or creates dependencies automatically:

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Entity

with DiodeClient(
    target="grpc://diode.example.com:8080/diode",
    app_name="my-app",
    app_version="1.0.0",
) as client:
    # No need to create manufacturer, device_type, site, or role first!
    # Just specify them by name - Diode handles the rest
    device = Device(
        name="switch-01",
        device_type="Catalyst 9300",  # Created if doesn't exist
        manufacturer="Cisco",          # Created if doesn't exist
        site="NYC-DC1",                # Created if doesn't exist
        role="Access Switch",          # Created if doesn't exist
        status="active",
    )

    response = client.ingest([Entity(device=device)])
```

**When to use Diode vs manual ordering:**
- **Diode**: Bulk imports, network discovery, migrations, high-volume ingestion
- **Manual ordering**: Single object creation, real-time CRUD, when using REST/GraphQL directly

See [integ-diode-ingestion](./integ-diode-ingestion.md) for comprehensive Diode guidance.

## Exceptions

- **Optional relationships:** Some foreign keys are optional (e.g., region on site)
- **Circular references:** Some relationships can be set after creation
- **Bulk imports:** NetBox's built-in import handles ordering automatically
- **Diode ingestion:** Diode handles dependency ordering automatically

## Related Rules

- [integ-diode-ingestion](./integ-diode-ingestion.md) - Use Diode for automatic dependency resolution
- [data-site-hierarchy](./data-site-hierarchy.md) - Site hierarchy structure
- [data-ipam-hierarchy](./data-ipam-hierarchy.md) - IPAM hierarchy
- [rest-list-endpoint-bulk-ops](./rest-list-endpoint-bulk-ops.md) - Bulk operations are atomic

## References

- [NetBox Data Model](https://netboxlabs.com/docs/netbox/en/stable/models/)
- [NetBox REST API](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/)
