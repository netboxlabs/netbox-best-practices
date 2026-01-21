# NetBox Best Practices

Best practices for NetBox API integrations, available for multiple AI coding assistants and human readers.

## Quick Start

| Audience | Start Here |
|----------|------------|
| **Engineers** | [HUMAN.md](./HUMAN.md) - Best practices guide |
| **Claude Code users** | Skills auto-discovered from `skills/` directory |
| **Cursor IDE users** | Rules in `.cursor/rules/` directory |

## Available Content

### Skills / Rules

| Topic | Description |
|-------|-------------|
| [netbox-integration-best-practices](./skills/netbox-integration-best-practices/) | REST/GraphQL API integration patterns |

### Content Formats

| Format | Location | Description |
|--------|----------|-------------|
| Human Documentation | `HUMAN.md` | Concise guide for engineers |
| Claude Code Skills | `skills/*/SKILL.md` | Progressive disclosure format |
| Cursor Rules | `.cursor/rules/*.mdc` | Concise IDE rules |
| Detailed Rules | `skills/*/references/rules/` | Authoritative source |

## For Engineers

[HUMAN.md](./HUMAN.md) provides a concise best practices guide covering all skills. For detailed rationale and exceptions, see the individual rule files in `skills/*/references/rules/`.

## For Claude Code Users

Skills are automatically discovered from the `skills/` directory. See [CLAUDE.md](./CLAUDE.md) for maintenance information.

## For Cursor IDE Users

Cursor rules are in the `.cursor/rules/` directory and will be automatically applied based on file patterns. See [AGENTS.md](./AGENTS.md) for a quick reference.
