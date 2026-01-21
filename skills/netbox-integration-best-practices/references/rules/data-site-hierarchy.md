---
title: Understand Site Hierarchy
impact: MEDIUM
category: data
tags: [data-model, hierarchy, sites, locations]
netbox_version: "4.4+"
---

# data-site-hierarchy: Understand Site Hierarchy

## Rationale

NetBox's site hierarchy is fundamental to the data model. Understanding it enables proper data population and querying.

## Hierarchy Structure

Region and Site Group are **parallel** organizational groupings that can optionally be assigned to a Site:

```
Region (geographic grouping)      Site Group (logical grouping)
         \                              /
          \                            /
           └──────── Site ────────────┘
                       │
                   Location (recursive)
                       │
                     Rack
                       │
                    Device
```

- **Region:** Geographic hierarchy (e.g., "North America" → "US-East" → "New York")
- **Site Group:** Logical grouping independent of geography (e.g., "Production DCs", "Edge Sites")
- **Site:** Physical facility (assigned to region and/or site group)
- **Location:** Areas within a site (can be nested recursively, e.g., "Floor 1" → "Row A" → "Cage 1")

## Correct Pattern

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Create organizational groupings (parallel, independent of each other)
region = nb.dcim.regions.create(name="North America", slug="na")
site_group = nb.dcim.site_groups.create(name="Data Centers", slug="dcs")

# Site can belong to region, site group, or both
site = nb.dcim.sites.create(
    name="NYC-DC1",
    slug="nyc-dc1",
    region=region.id,      # Optional
    group=site_group.id,   # Optional
    status="active"
)

# Locations are recursive (can be nested to any depth)
floor = nb.dcim.locations.create(
    name="Floor 1",
    slug="floor-1",
    site=site.id
)
row = nb.dcim.locations.create(
    name="Row A",
    slug="row-a",
    site=site.id,
    parent=floor.id  # Nested within floor
)
cage = nb.dcim.locations.create(
    name="Cage 1",
    slug="cage-1",
    site=site.id,
    parent=row.id    # Nested within row
)
```

## Querying by Hierarchy

```python
# Devices in a region
devices = nb.dcim.devices.filter(region="na")

# Devices in a site group
devices = nb.dcim.devices.filter(site_group="dcs")

# Devices at a specific site
devices = nb.dcim.devices.filter(site="nyc-dc1")
```

## Related Rules

- [data-dependency-order](./data-dependency-order.md) - Creation order
- [data-ipam-hierarchy](./data-ipam-hierarchy.md) - IPAM structure
