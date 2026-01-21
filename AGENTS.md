# AGENTS.md - NetBox Best Practices

This repository contains best practices for NetBox API integrations. The content is available in multiple formats for different AI coding assistants and human readers.

## Quick Start

When working on NetBox integrations, follow these key principles:

1. **Use v2 tokens** on NetBox 4.5+ (`Bearer nbt_<key>.<token>`)
2. **Always paginate** list requests - NetBox defaults to 50 items
3. **Use brief mode** (`?brief=True`) and exclude config_context for performance
4. **Use Diode** for high-volume data ingestion (handles dependencies automatically)
5. **Use pynetbox** for Python integrations

## Content Locations

| Format | Location | Audience |
|--------|----------|----------|
| Human Documentation | `HUMAN.md` | Engineers/developers |
| Claude Code Skills | `skills/*/SKILL.md` | Claude Code users |
| Cursor Rules | `.cursor/rules/*.mdc` | Cursor IDE users |
| Detailed Rules | `skills/*/references/rules/*.md` | All (authoritative source) |

## Available Rules by Category

### Authentication (`auth-`)
- Use v2 tokens on NetBox 4.5+ (CRITICAL)
- Use provisioning endpoint for automated token creation

### REST API (`rest-`)
- Always paginate list requests (HIGH)
- Use bulk operations via list endpoints (CRITICAL)
- Use PATCH for partial updates, not PUT
- Use brief mode and field selection for performance
- Avoid `?q=` search filter at scale

### GraphQL (`graphql-`)
- Use netbox-graphql-query-optimizer (CRITICAL)
- Always paginate, including nested lists (CRITICAL)
- Keep query depth ≤3
- Select only needed fields

### Performance (`perf-`)
- Exclude config_context from device queries
- Use brief mode for large list operations

### Data Modeling (`data-`)
- Create objects in dependency order (CRITICAL)
- Understand site/location hierarchy
- Use natural keys for queries
- Properly structure custom fields and tags

### Integration (`integ-`)
- Use Diode for high-volume ingestion (HIGH)
- Use pynetbox for Python integrations (HIGH)
- Configure webhooks for change notifications
- Query object changes for audit trails

## Maintenance

**Important:** This repository maintains content in three parallel formats that must stay synchronized:

1. **Human Documentation** - `HUMAN.md` (single doc for all skills)
2. **Claude Code Skills** - `skills/*/SKILL.md`
3. **Cursor Rules** - `.cursor/rules/*.mdc`

When making changes:
- The authoritative source is `skills/*/references/rules/*.md`
- Update all three formats to maintain consistency
- See `CLAUDE.md` for detailed maintenance procedures

## Key References

- [NetBox REST API Documentation](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/)
- [NetBox GraphQL API Documentation](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
- [pynetbox](https://github.com/netbox-community/pynetbox) - Official Python client
- [Diode](https://github.com/netboxlabs/diode) - High-volume data ingestion
- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)
