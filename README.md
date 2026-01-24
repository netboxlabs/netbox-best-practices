# NetBox Best Practices

Best practices for NetBox API integrations, available for multiple AI coding assistants and human readers.

## Quick Start

| Audience | Start Here |
|----------|------------|
| **Engineers** | [HUMAN.md](./HUMAN.md) - Best practices guide |
| **Claude Code users** | Install skills (see below) |
| **Cursor IDE users** | Install rules (see below) |

### Claude Code Installation

**Option 1: npx skills CLI**
```bash
npx skills add netboxlabs/netbox-best-practices
```

**Option 2: Manual install**
```bash
# Global (all projects)
cp -r skills/* ~/.claude/skills/

# Or project-local
cp -r skills/* .claude/skills/
```

### Cursor IDE Installation

Copy `.cursor/rules/` to your project root:
```bash
cp -r .cursor/rules/ /path/to/your/project/.cursor/rules/
```

Rules are auto-applied based on file patterns. See [Cursor Rules docs](https://cursor.com/docs/context/rules) for details.

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
