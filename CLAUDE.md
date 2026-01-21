# CLAUDE.md - NetBox Skills Maintenance Guide

This repository contains best practices for NetBox integrations, available in multiple formats for different AI coding assistants and human readers. This guide explains the structure, conventions, and maintenance procedures.

## Repository Structure

```
netbox-best-practices/
├── CLAUDE.md                              # This file - maintenance guide
├── AGENTS.md                              # Cursor/general AI assistant guide
├── HUMAN.md                               # Human-readable guide (for engineers)
├── README.md                              # Repository overview
├── .cursor/rules/                         # Cursor IDE rules
│   ├── netbox-auth.mdc
│   ├── netbox-rest-api.mdc
│   ├── netbox-graphql.mdc
│   ├── netbox-data-modeling.mdc
│   ├── netbox-integration.mdc
│   └── netbox-performance.mdc
└── skills/
    └── netbox-integration-best-practices/
        ├── SKILL.md                       # Main skill definition (for Claude Code)
        └── references/
            ├── netbox-integration-guidelines.md  # Technical reference
            └── rules/
                ├── _sections.md           # Category definitions
                ├── _template.md           # Rule template
                └── [individual rule files]
```

## Multi-Format Content

This repository maintains content in three parallel formats:

| Format | Location | Audience | Style |
|--------|----------|----------|-------|
| **Claude Code Skills** | `skills/*/SKILL.md`, `references/` | Claude Code users | Progressive disclosure, reference files |
| **Cursor Rules** | `.cursor/rules/*.mdc` | Cursor IDE users | Concise (<500 lines), YAML frontmatter |
| **Human Documentation** | `HUMAN.md` | Engineers | Concise orientation, all skills combined |

The **authoritative source** for all rules is `skills/*/references/rules/*.md`. The other formats are derived from and must stay aligned with these detailed rule files.

**Important:** `HUMAN.md` is a single document covering ALL best practices across all skills. When adding new skills, add their content to this document rather than creating skill-specific human docs.

## Important Maintenance Principles

### Keep All Three Formats in Sync (CRITICAL)

When adding, updating, or removing rules or guidance, **always update all three formats**:

1. **Authoritative Rules** (`skills/*/references/rules/*.md`)
   - This is the source of truth
   - Contains full rationale, examples, exceptions
   - Update this FIRST

2. **Claude Code Skills** (`skills/*/SKILL.md`)
   - Progressive disclosure format
   - References the detailed rule files
   - Update rule tables to match

3. **Cursor Rules** (`.cursor/rules/*.mdc`)
   - Concise format (<500 lines per file)
   - Organized by category (auth, rest, graphql, etc.)
   - Include key patterns and examples
   - Link to detailed rules for more info

4. **Human Documentation** (`HUMAN.md`)
   - Single document for ALL skills
   - Concise orientation for engineers/reviewers
   - Add new skill content as new sections

5. **AGENTS.md**
   - Summary for Cursor/other AI assistants
   - Quick reference of key principles
   - Update if adding new categories

### Cross-Format Update Checklist

When making changes:

- [ ] Update the authoritative rule file in `references/rules/`
- [ ] Update `_sections.md` if adding/removing rules or categories
- [ ] Update `SKILL.md` rule tables
- [ ] Update relevant `.cursor/rules/*.mdc` file
- [ ] Update `HUMAN.md` for significant changes
- [ ] Update `AGENTS.md` if adding new categories or critical rules

### Cursor Rules Guidelines

Cursor `.mdc` files have specific requirements:

```markdown
---
description: "Brief description for intelligent application"
globs: ["**/*.py"]
alwaysApply: false
---

# Rule content in markdown
```

- Keep each file under 500 lines (ideally much shorter)
- Use `globs` to match relevant file types
- Include concrete code examples (correct/incorrect patterns)
- Link to detailed rule files for comprehensive information
- Organize by category, not by individual rule

### NetBox-Specific Content Only

This repository focuses exclusively on **NetBox and Diode-specific** best practices. Do NOT include generic software development guidance such as:
- Token/secret storage patterns (use environment variables, vaults, etc.)
- Retry strategies and exponential backoff
- Connection pooling
- Generic error handling
- Audit logging patterns
- Rate limiting implementation

These are important topics but are covered extensively elsewhere. Keep rules focused on NetBox API behaviors, Diode ingestion patterns, and NetBox-specific performance considerations.

## Skills Overview

### Current Skills

| Skill | Purpose | Target Audience |
|-------|---------|-----------------|
| `netbox-integration-best-practices` | REST/GraphQL API integration patterns | Engineers building integrations |

### Future Skills (Roadmap)

| Skill | Purpose | Status |
|-------|---------|--------|
| `netbox-plugin-development` | Plugin architecture and hooks | Planned |
| `netbox-custom-scripts` | Custom scripts and reports | Planned |
| `netbox-admin-operations` | Administration and operations | Planned |

## Adding/Updating Rules

### Rule File Structure

All rules follow the template in `_template.md`:

```yaml
---
title: Rule Title
impact: CRITICAL | HIGH | MEDIUM | LOW
category: auth | rest | graphql | perf | data | integ
tags: [relevant, tags]
netbox_version: "4.4+"
---
```

### Rule Naming Convention

Rules are prefixed by category:
- `auth-` - Authentication rules
- `rest-` - REST API rules
- `graphql-` - GraphQL rules
- `perf-` - Performance rules
- `data-` - Data modeling rules
- `integ-` - Integration rules

### Impact Levels

| Impact | Description | Action Required |
|--------|-------------|-----------------|
| CRITICAL | Security vulnerabilities, data loss risk, breaking changes | Must fix immediately |
| HIGH | Significant performance/reliability impact | Should fix soon |
| MEDIUM | Notable improvements, best practices | Plan to address |
| LOW | Minor improvements, style preferences | Consider when convenient |

### Adding a New Rule

1. Copy `_template.md` to `rules/{category}-{rule-name}.md`
2. Fill in YAML frontmatter with appropriate metadata
3. Document the rationale (why this matters)
4. Provide incorrect pattern with annotation
5. Provide correct pattern with explanation
6. Add async example if applicable for high-volume patterns
7. List exceptions and edge cases
8. Add related rules and references
9. Update `_sections.md` if adding a new category
10. Update `SKILL.md` rule tables
11. **Update the corresponding `.cursor/rules/*.mdc` file**
12. **Update `HUMAN.md`** if it's a significant rule
13. Update `AGENTS.md` if it's a critical rule or new category

## Code Example Standards

### Primary Libraries (in order of preference)

1. **requests** - Standard synchronous HTTP library (most examples)
2. **pynetbox** - Official NetBox Python client
3. **httpx** - Async HTTP library for high-volume patterns

### Example Conventions

```python
# Base URL format
NETBOX_URL = "https://netbox.example.com"
API_URL = f"{NETBOX_URL}/api"

# Authentication headers (v2 token format)
headers = {
    "Authorization": "Bearer nbt_abc123.xxxxxxxxxxxxxxxx",
    "Content-Type": "application/json"
}

# Never include real tokens
# Use descriptive placeholder values
```

### Code Quality Requirements

- All examples must be syntactically correct Python
- Include error handling in "correct" patterns
- Show both incorrect and correct patterns
- Use type hints where it aids clarity
- Include complexity scores for GraphQL examples
- Test examples where possible

## NetBox Version Compatibility

### Target Versions

| Version | Support Level | Notes |
|---------|--------------|-------|
| 4.4+ | Primary target | General compatibility |
| 4.5+ | v2 tokens | Required for v2 token features |
| 4.7+ | Future | v1 tokens deprecated |

### Version-Specific Notes

Always note when features require specific versions:

```markdown
> **NetBox 4.5+**: v2 tokens use `Bearer nbt_<key>.<token>` format.
```

## v2 Token Migration Timeline

This is critical information for all authentication-related documentation:

| Version | Token Status |
|---------|-------------|
| < 4.5 | v1 tokens only (`Token <token>`) |
| 4.5.0 | v2 tokens introduced (`Bearer nbt_<key>.<token>`) |
| 4.7.0 | v1 tokens deprecated (removal planned) |

### v2 Token Format

```
Bearer nbt_<key>.<token>
```

- `nbt_` prefix identifies as NetBox token
- `<key>` is the public key identifier
- `<token>` is the secret portion
- Uses HMAC-SHA256 with pepper
- Plaintext never stored in database
- Requires `API_TOKEN_PEPPERS` configuration

## Validation Checklist

Before committing changes:

- [ ] All internal markdown links resolve
- [ ] Code examples are syntactically correct
- [ ] YAML frontmatter is valid in all rule files
- [ ] Rule tables in SKILL.md match actual rule files
- [ ] Cursor rules in `.cursor/rules/*.mdc` are updated
- [ ] AGENTS.md reflects any new categories or critical rules
- [ ] HUMAN.md is updated for significant changes
- [ ] Examples use correct NetBox API behavior
- [ ] Version requirements are documented
- [ ] Impact levels are appropriate
- [ ] All three formats (Claude Skills, Cursor Rules, Human Docs) are aligned

## External References

### Official Documentation
- [NetBox Documentation](https://netboxlabs.com/docs/netbox/en/stable/)
- [NetBox REST API](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/)
- [NetBox GraphQL API](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)

### Key Tools
- [pynetbox](https://github.com/netbox-community/pynetbox) - Official Python client
- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer) - Query analysis tool
- [Diode](https://github.com/netboxlabs/diode) - Data ingestion service (for high-volume writes)
- [Diode Python SDK](https://github.com/netboxlabs/diode-sdk-python) - Python client for Diode

### Community Resources
- [NetBox GitHub Discussions](https://github.com/netbox-community/netbox/discussions)
- [NetBox Community Slack](https://netdev.chat/)

## Maintenance Schedule

- Review for NetBox version updates: Each minor release
- Check community discussions for new performance insights: Monthly
- Validate code examples against latest API: Quarterly
- Update GraphQL optimizer recommendations: As new versions release
