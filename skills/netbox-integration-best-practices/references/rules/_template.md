# Rule Template

Use this template when creating new rules. Copy this file and fill in all sections.

---

```markdown
---
title: [Descriptive Rule Title]
impact: [CRITICAL | HIGH | MEDIUM | LOW]
category: [auth | rest | graphql | perf | data | integ | sec]
tags: [tag1, tag2, tag3]
netbox_version: "4.4+"
---

# [rule-name]: [Rule Title]

## Rationale

[Explain WHY this rule matters. What problems does it prevent? What benefits does following it provide? Include performance numbers or security implications where relevant.]

## Incorrect Pattern

[Show what NOT to do, with annotations explaining the problems]

```python
# WRONG: [Brief explanation of why this is wrong]
[code example showing the anti-pattern]
```

**Problems with this approach:**
- [Problem 1]
- [Problem 2]
- [Problem 3]

## Correct Pattern

[Show the recommended approach with explanation]

```python
# CORRECT: [Brief explanation of the approach]
[code example showing the correct pattern]
```

**Benefits:**
- [Benefit 1]
- [Benefit 2]
- [Benefit 3]

## Async Example

[Include if applicable for high-volume patterns]

```python
import asyncio
import httpx

async def example():
    [async implementation]
```

## pynetbox Example

[Include if applicable]

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token=TOKEN)
[pynetbox implementation]
```

## Exceptions

[List any cases where this rule doesn't apply or has different considerations]

- **Exception 1:** [When and why this exception applies]
- **Exception 2:** [When and why this exception applies]

## Related Rules

- [related-rule-1](./related-rule-1.md) - [Brief description]
- [related-rule-2](./related-rule-2.md) - [Brief description]

## References

- [NetBox Documentation: Topic](https://netboxlabs.com/docs/netbox/en/stable/...)
- [Additional reference](URL)
```

---

## Template Guidelines

### Impact Levels

| Level | Criteria |
|-------|----------|
| **CRITICAL** | Security vulnerabilities, data loss risk, breaking changes, or severe performance degradation |
| **HIGH** | Significant performance impact, reliability concerns, or common source of bugs |
| **MEDIUM** | Notable improvements, important best practices |
| **LOW** | Minor improvements, style preferences, optional optimizations |

### Category Prefixes

| Category | Prefix | Description |
|----------|--------|-------------|
| Authentication | `auth-` | Token management, authentication methods |
| REST API | `rest-` | REST API patterns and practices |
| GraphQL | `graphql-` | GraphQL query patterns |
| Performance | `perf-` | Optimization and efficiency |
| Data Modeling | `data-` | Data model usage and relationships |
| Integration | `integ-` | Integration patterns and tooling |
| Security | `sec-` | Security practices |

### Code Example Standards

1. **Primary library:** Use `requests` for most examples
2. **pynetbox:** Include pynetbox examples when simpler or more idiomatic
3. **Async:** Use `httpx` for async examples
4. **URLs:** Always use `https://netbox.example.com`
5. **Tokens:** Never use real tokens; use placeholder patterns like `nbt_abc123.xxxxx`
6. **Error handling:** Include in correct patterns
7. **Comments:** Use `# WRONG:` and `# CORRECT:` annotations

### Version Notes

When a feature requires a specific NetBox version, use this format:

> **NetBox 4.5+**: [Feature description]

### File Naming

- Use kebab-case: `auth-use-v2-tokens.md`
- Prefix with category: `rest-`, `graphql-`, etc.
- Keep names concise but descriptive
