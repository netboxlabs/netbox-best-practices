---
title: Use v2 Tokens on NetBox 4.5+
impact: CRITICAL
category: auth
tags: [authentication, tokens, security, migration]
netbox_version: "4.5+"
---

# auth-use-v2-tokens: Use v2 Tokens on NetBox 4.5+

## Rationale

NetBox 4.5 introduced v2 tokens with significant security improvements. v1 tokens store plaintext secrets in the database, creating risk if the database is compromised. v2 tokens use HMAC-SHA256 hashing with a pepper, ensuring the plaintext token is never stored.

**Timeline:**
- NetBox 4.5.0: v2 tokens introduced
- NetBox 4.7.0: v1 tokens deprecated (removal planned)

Migrating to v2 tokens before 4.7 is essential for uninterrupted API access.

## Incorrect Pattern

```python
# WRONG: v1 token format (deprecated in 4.7+)
import requests

NETBOX_URL = "https://netbox.example.com"
TOKEN = "0123456789abcdef0123456789abcdef01234567"

headers = {
    "Authorization": f"Token {TOKEN}",  # v1 format
    "Content-Type": "application/json"
}

response = requests.get(f"{NETBOX_URL}/api/dcim/devices/", headers=headers)
```

**Problems with this approach:**
- v1 tokens are stored in plaintext in the database
- Database compromise exposes all tokens
- Will stop working after v1 removal (post-4.7)
- No cryptographic protection of token secrets

## Correct Pattern

```python
# CORRECT: v2 token format (NetBox 4.5+)
import requests

NETBOX_URL = "https://netbox.example.com"
TOKEN = "nbt_abc123def456.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

headers = {
    "Authorization": f"Bearer {TOKEN}",  # v2 format
    "Content-Type": "application/json"
}

response = requests.get(f"{NETBOX_URL}/api/dcim/devices/", headers=headers)
```

**Benefits:**
- Token secret hashed with HMAC-SHA256
- Pepper adds server-side secret to hash
- Database compromise doesn't expose usable tokens
- Future-proof for NetBox 4.7+

## v2 Token Format Details

```
Bearer nbt_<key>.<token>
```

- `Bearer`: OAuth-style prefix (required)
- `nbt_`: NetBox token identifier
- `<key>`: Public key for token lookup
- `<token>`: Secret portion (never stored in plaintext)

## pynetbox Example

```python
import pynetbox

# pynetbox handles the header format automatically
nb = pynetbox.api(
    url="https://netbox.example.com",
    token="nbt_abc123def456.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
)

# Works with both v1 and v2 tokens
devices = nb.dcim.devices.all()
```

## Migration Steps

1. **Check NetBox version:** Ensure you're on 4.5+
2. **Verify API_TOKEN_PEPPERS configured:** Required for v2 tokens
3. **Generate new v2 token** in NetBox UI
4. **Update integration** with new token format
5. **Test thoroughly** before production
6. **Revoke old v1 token** after successful migration

## Server Configuration Requirement

v2 tokens require `API_TOKEN_PEPPERS` in NetBox configuration:

```python
# configuration.py
API_TOKEN_PEPPERS = {
    'default': 'your-secret-pepper-minimum-32-characters-long',
}
```

Without this, v2 tokens cannot be validated.

## Exceptions

- **NetBox < 4.5:** v1 tokens are the only option
- **Legacy integrations:** May require coordinated migration across teams

## Related Rules

- [auth-token-rotation](./auth-token-rotation.md) - Implement regular token rotation
- [sec-token-storage](./sec-token-storage.md) - Never store tokens in code

## References

- [NetBox 4.5 Release Notes](https://netboxlabs.com/docs/netbox/en/stable/release-notes/)
- [NetBox Authentication](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/#authentication)
