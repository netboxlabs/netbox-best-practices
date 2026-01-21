---
title: Use Provisioning Endpoint for Automated Token Creation
impact: MEDIUM
category: auth
tags: [authentication, tokens, automation, provisioning]
netbox_version: "4.4+"
---

# auth-provisioning-endpoint: Use Provisioning Endpoint for Automated Token Creation

## Rationale

For systems that need to bootstrap their own tokens, NetBox provides a token provisioning endpoint. This enables automated token creation without pre-existing API access.

## Correct Pattern

```python
import requests

def provision_token(netbox_url, username, password, description=None):
    """Provision a new API token using credentials."""
    payload = {
        "username": username,
        "password": password
    }
    if description:
        payload["description"] = description

    response = requests.post(
        f"{netbox_url}/api/users/tokens/provision/",
        json=payload,
        headers={"Content-Type": "application/json"}
    )

    if response.status_code == 201:
        return response.json()["key"]
    else:
        raise Exception(f"Failed to provision token: {response.text}")

# Usage
token = provision_token(
    "https://netbox.example.com",
    "automation-user",
    "secure-password",
    "Automated pipeline token"
)
# Store token securely for future use
```

## Use Cases

- CI/CD pipeline bootstrapping
- Dynamic environment provisioning
- Token rotation automation
- Multi-tenant token management

## Security Considerations

- Protect the username/password used for provisioning
- Use service accounts with limited permissions
- Consider provisioning tokens with short lifespans

## Related Rules

- [auth-token-rotation](./auth-token-rotation.md) - Rotate tokens regularly
- [sec-token-storage](./sec-token-storage.md) - Store tokens securely
