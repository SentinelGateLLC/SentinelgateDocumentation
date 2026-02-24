# SentinelGate — API Reference

**For developers building custom integrations**

**Base URL:** `https://sentinelgate.biz`
**Protocol:** HTTPS only (TLS 1.2+)
**Format:** JSON

---

## Authentication

All authenticated endpoints require these headers:

```http
X-API-Key: sg_key_your_merchant_key
X-API-Secret: sg_secret_your_merchant_secret
Content-Type: application/json
```

Contact SentinelGate to obtain your API credentials.

---

## Endpoints

### POST /v1/hosted/create

Create a hosted payment session. Returns a URL to redirect the customer to.

**Authentication:** Required

**Request:**

```json
{
  "amount": "191.00",
  "currency": "USD",
  "order_id": "ORD-7700",
  "description": "Order #7700",
  "customer_email": "buyer@example.com",
  "customer_name": "John Doe",
  "callback_url": "https://yoursite.com/payment-webhook",
  "return_url": "https://yoursite.com/order-confirmed",
  "cancel_url": "https://yoursite.com/checkout"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | Yes | Payment amount (e.g., "191.00") |
| `currency` | string | Yes | ISO 4217 currency code (USD, GHS, KES, etc.) |
| `order_id` | string | Yes | Your internal order identifier |
| `description` | string | No | Payment description shown to customer |
| `customer_email` | string | No | Customer email for receipts |
| `customer_name` | string | No | Customer display name |
| `callback_url` | string | Yes | URL to receive payment status webhook |
| `return_url` | string | Yes | Where to redirect customer after successful payment |
| `cancel_url` | string | No | Where to redirect if customer cancels |

**Response (200 OK):**

```json
{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "session_id": "sg_session_cc7d30bba805dca1c7c0828b",
  "redirect_url": "https://pay.provider.com/checkout-session-id",
  "hosted_url": "https://pay.provider.com/checkout-session-id",
  "status": "pending"
}
```

**Next step:** Redirect the customer's browser to `redirect_url`.

---

### POST /v1/charge

Process a direct card charge. **Requires PCI DSS Level 1 compliance.** Most integrations should use `/v1/hosted/create` instead.

**Authentication:** Required

**Request:**

```json
{
  "amount": "50.00",
  "currency": "USD",
  "order_id": "ORD-001",
  "customer_email": "buyer@example.com",
  "card": {
    "number": "4111111111111111",
    "cvv": "123",
    "expiry_month": "12",
    "expiry_year": "26"
  }
}
```

**Response (200 OK):**

```json
{
  "sentinel_transaction_id": "sg_txn_...",
  "status": "captured",
  "message": "Approved",
  "provider": "paystack"
}
```

Possible statuses: `captured`, `failed`, `3DS_REDIRECT`, `AUTH_REQUIRED`.

If `status` is `3DS_REDIRECT`, redirect the customer to the provided `redirect_url` for 3D Secure verification.

---

### GET /v1/transaction/:txnId

Query the status of a transaction.

**Authentication:** Required

**Request:**

```http
GET /v1/transaction/sg_txn_1771888643979_cfe07b9d7fe6
X-API-Key: sg_key_...
X-API-Secret: sg_secret_...
```

**Response (200 OK):**

```json
{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "status": "captured",
  "amount": 191.00,
  "currency": "USD",
  "provider": "hubtel",
  "created_at": "2026-02-24T06:30:00Z"
}
```

---

### POST /v1/refund

Process a full or partial refund.

**Authentication:** Required

**Request:**

```json
{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "amount": 50.00,
  "reason": "Customer requested partial refund"
}
```

**Response (200 OK):**

```json
{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "refund_id": "sg_ref_abc123def456",
  "amount": 50.00,
  "status": "refunded",
  "reason": "Customer requested partial refund"
}
```

---

## Webhooks

When a payment status changes, SentinelGate sends an HTTP POST to your `callback_url`.

### Webhook Payload

```json
{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "wc_order_id": "7700",
  "status": "captured",
  "amount": 191.00,
  "currency": "USD",
  "provider": "hubtel",
  "provider_reference": "hubtel-ref-abc123",
  "gateway_response": "Approved",
  "channel": "card"
}
```

### Webhook Statuses

| Status | Meaning | Action |
|--------|---------|--------|
| `captured` | Payment successful | Fulfill the order |
| `failed` | Payment declined | Notify customer, allow retry |
| `refunded` | Refund processed | Process return |

### Webhook Signature Verification

Webhooks include an `X-Sentinel-Signature` header for verification:

```
X-Sentinel-Signature: sha256=<hmac_hex_digest>
```

Verify by computing HMAC-SHA256 of the raw request body using your Webhook Secret:

```javascript
const crypto = require('crypto');

function verifyWebhook(rawBody, signature, webhookSecret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', webhookSecret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );
}
```

```python
import hmac
import hashlib

def verify_webhook(raw_body: bytes, signature: str, webhook_secret: str) -> bool:
    expected = 'sha256=' + hmac.new(
        webhook_secret.encode(),
        raw_body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

```php
function verifyWebhook($rawBody, $signature, $webhookSecret) {
    $expected = 'sha256=' . hash_hmac('sha256', $rawBody, $webhookSecret);
    return hash_equals($expected, $signature);
}
```

### Webhook Best Practices

1. **Respond with 200 quickly** — Process the webhook asynchronously if needed. SentinelGate expects a response within 15 seconds.
2. **Handle duplicates** — Use the `sentinel_transaction_id` to detect duplicate webhooks. Process each transaction only once.
3. **Verify signatures** — Always verify the `X-Sentinel-Signature` header before processing.
4. **Use HTTPS** — Your callback URL must use HTTPS.
5. **Retry behavior** — If your endpoint returns a non-2xx status, SentinelGate will retry up to 3 times with exponential backoff.

---

## Payment Links

SentinelGate can generate shareable payment links for invoices, one-time payments, or recurring collection.

### Link Types

| Type | Description | Use Case |
|------|-------------|----------|
| `STANDARD` | Fixed amount, multiple payment methods | General payments |
| `INVOICE` | Fixed amount, limited methods (Card, Bank, Crypto) | B2B invoices |
| `GAMING` | Variable amount, card/crypto only | Top-ups, credits |

### Payment Link URLs

```
https://sentinelgate.biz/pay/<token>
```

Each link has a unique token. Customers visit the URL, see the amount and merchant info, choose a payment method, and complete payment.

### QR Codes

Each payment link has an auto-generated QR code:

```
https://sentinelgate.biz/pay/<token>/qr.png
```

Use this in printed invoices, POS displays, or email templates.

---

## Error Handling

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request — check the request body |
| 401 | Unauthorized — invalid or missing API key |
| 404 | Not found — invalid transaction ID or session |
| 429 | Rate limited — slow down requests |
| 500 | Server error — retry after a delay |

### Error Response Format

```json
{
  "error": "INVALID_REQUEST",
  "message": "amount is required"
}
```

### Common Errors

| Error Code | Cause | Solution |
|-----------|-------|---------|
| `MISSING_API_KEY` | No `X-API-Key` header | Add the header to your request |
| `INVALID_REQUEST` | Missing required fields | Check required fields in the endpoint docs |
| `SESSION_EXPIRED` | Payment session timed out | Create a new session |
| `PROVIDER_ERROR` | Payment provider returned an error | Check `message` field for details |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| `/v1/hosted/create` | 60 requests/minute |
| `/v1/charge` | 30 requests/minute |
| `/v1/transaction/:id` | 120 requests/minute |
| `/v1/refund` | 10 requests/minute |
| Webhooks (inbound) | 100 requests/minute per IP |

Exceeding rate limits returns HTTP 429. Implement exponential backoff in your integration.

---

## Testing

### Test Card Numbers

Use these card numbers in sandbox/test mode:

| Card Number | Result |
|-------------|--------|
| 4111 1111 1111 1111 | Approved |
| 4000 0000 0000 0002 | Declined |
| 4000 0000 0000 3220 | 3D Secure required |

Use any future expiry date and any 3-digit CVV.

**Note:** Test cards only work with providers that have test/sandbox mode enabled. Contact SentinelGate for test environment access.

### Webhook Testing

Use a tool like [webhook.site](https://webhook.site) or [ngrok](https://ngrok.com) to test webhook delivery during development:

```bash
# Using ngrok to expose a local endpoint
ngrok http 3000

# Use the ngrok URL as your callback_url
# https://abc123.ngrok.io/payment-webhook
```

---

## SDKs and Libraries

Official SDKs are planned. In the meantime, use standard HTTP libraries:

| Language | Recommended Library |
|----------|-------------------|
| JavaScript/Node.js | `axios`, `node-fetch` |
| Python | `requests`, `httpx` |
| PHP | `Guzzle`, `cURL` |
| Ruby | `Faraday`, `HTTParty` |

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.1.0 | 2026-02-24 | Added hosted checkout with card form, Hubtel integration, payment links |
| 1.0.0 | 2026-02-07 | Initial API release |
