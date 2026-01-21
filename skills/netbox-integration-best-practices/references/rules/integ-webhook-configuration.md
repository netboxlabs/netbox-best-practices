---
title: Configure Webhooks for Event-Driven Integration
impact: MEDIUM
category: integ
tags: [integration, webhooks, events, automation]
netbox_version: "4.4+"
---

# integ-webhook-configuration: Configure Webhooks for Event-Driven Integration

## Rationale

Webhooks enable NetBox to push changes to your systems in real-time, eliminating the need for polling and enabling immediate reactions to changes.

## Correct Pattern

```python
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret"

def verify_signature(payload, signature):
    """Verify webhook signature."""
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha512
    ).hexdigest()
    return hmac.compare_digest(signature, expected)

@app.route("/netbox-webhook", methods=["POST"])
def handle_webhook():
    signature = request.headers.get("X-Hook-Signature", "")
    if not verify_signature(request.data, signature):
        return jsonify({"error": "Invalid signature"}), 401

    event = request.json
    event_type = event["event"]    # created, updated, deleted
    model = event["model"]          # dcim.device, etc.
    data = event["data"]

    # Route to handlers
    if model == "dcim.device" and event_type == "created":
        handle_new_device(data)

    return jsonify({"status": "processed"}), 200
```

## Webhook Configuration in NetBox

1. Admin → Webhooks → Add
2. Set content types to monitor
3. Configure HTTP method, URL, headers
4. Set secret for signature verification

## Related Rules

- [integ-event-driven](./integ-event-driven.md) - Event-driven patterns
- [integ-retry-strategies](./integ-retry-strategies.md) - Handle failures
