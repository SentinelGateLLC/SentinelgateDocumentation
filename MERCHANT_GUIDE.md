# SentinelGate — Merchant Integration Guide

## Table of Contents

1. [Getting Started](#getting-started)
2. [Authentication](#authentication)
3. [Integration Modes](#integration-modes)
4. [Hosted Checkout (Redirect)](#hosted-checkout-redirect)
5. [Server-to-Server (Direct Charge)](#server-to-server-direct-charge)
6. [Mobile Money (STK Push)](#mobile-money-stk-push)
7. [Payment Links](#payment-links)
8. [Webhooks & Callbacks](#webhooks--callbacks)
9. [Transaction Status](#transaction-status)
10. [Refunds](#refunds)
11. [Supported Currencies & Providers](#supported-currencies--providers)
12. [WooCommerce Plugin](#woocommerce-plugin)
13. [Shopify Integration](#shopify-integration)
14. [Error Handling](#error-handling)
15. [Testing](#testing)
16. [Postman Collection](#postman-collection)
17. [Security Best Practices](#security-best-practices)
18. [FAQ](#faq)

---

## Getting Started

### Prerequisites

- A SentinelGate merchant account (contact sales@sentinelgate.biz)
- API credentials (API Key + API Secret)
- HTTPS-enabled server for receiving webhooks
- For Server-to-Server mode: PCI DSS Level 1 compliance

### Base URL

```
https://sentinelgate.biz
```

### Quick Test

Verify your credentials with a simple hosted checkout request:

```bash
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "X-API-Secret: YOUR_API_SECRET" \
  -d '{
    "amount": "1.00",
    "currency": "USD",
    "order_id": "TEST-001",
    "description": "Test Payment",
    "customer_email": "test@example.com",
    "callback_url": "https://yoursite.com/webhook",
    "return_url": "https://yoursite.com/thank-you"
  }'
```

A successful response returns a `redirect_url` pointing to the payment page.

---

## Authentication

All API requests require your merchant credentials in the headers:

```
X-API-Key: your_api_key
X-API-Secret: your_api_secret
Content-Type: application/json
```

> ⚠️ **Never expose your API Secret in client-side code.** All API calls must be made from your server.

---

## Integration Modes

| Mode | How It Works | PCI Requirement | Best For |
|------|-------------|-----------------|----------|
| **Hosted Checkout** | Redirect customer to SentinelGate/provider payment page | None | Most merchants |
| **iFrame** | Embed payment form in your page | None | Seamless UX |
| **Server-to-Server** | Send card data directly via API | PCI DSS Level 1 | Providers, platforms |
| **Payment Links** | Generate shareable payment URLs | None | Invoices, social |

---

## Hosted Checkout (Redirect)

The simplest integration — no PCI requirements.

### Create Payment Session

**`POST /v1/hosted/create`**

```json
{
  "amount": "125.00",
  "currency": "USD",
  "order_id": "ORD-12345",
  "description": "Order #12345 — 2x Widget Pro",
  "customer_name": "John Doe",
  "customer_email": "john@example.com",
  "customer_phone": "+254712345678",
  "callback_url": "https://yourstore.com/webhooks/sentinelgate",
  "return_url": "https://yourstore.com/order-received/12345/",
  "cancel_url": "https://yourstore.com/cart/",
  "metadata": {
    "internal_ref": "abc123"
  }
}
```

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | ✅ | Payment amount (e.g., `"125.00"`) |
| `currency` | string | ✅ | ISO 4217 code: `USD`, `KES`, `GHS`, `NGN` |
| `order_id` | string | ✅ | Your unique order/invoice reference |
| `description` | string | ❌ | Description shown on payment page |
| `customer_name` | string | ❌ | Customer full name |
| `customer_email` | string | ❌ | Customer email (required for card payments) |
| `customer_phone` | string | ❌ | Customer phone (required for M-Pesa/MoMo) |
| `callback_url` | string | ✅ | Server URL to receive payment notifications |
| `return_url` | string | ✅ | Where to redirect customer after payment |
| `cancel_url` | string | ❌ | Where to redirect if customer cancels |
| `metadata` | object | ❌ | Arbitrary key-value data returned in webhooks |

### Response

```json
{
  "sentinel_transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "session_id": "sg_session_0d34c3fa3b3d41732d0d82fc",
  "redirect_url": "https://pay.pesapal.com/iframe/PesapalIframe3/Index?OrderTrackingId=...",
  "hosted_url": "https://pay.pesapal.com/iframe/PesapalIframe3/Index?OrderTrackingId=...",
  "status": "pending"
}
```

### Payment Flow

```
1. Your Server     POST /v1/hosted/create     →  SentinelGate API
2. SentinelGate    Routes to provider          →  Returns redirect_url
3. Your Server     HTTP 302 redirect           →  Customer's browser
4. Customer        Enters card / confirms MoMo →  Provider payment page
5. Provider        Payment result              →  SentinelGate
6. SentinelGate    POST webhook payload        →  Your callback_url
7. Customer        HTTP 302 redirect           →  Your return_url
```

---

## Server-to-Server (Direct Charge)

For PCI-compliant merchants who collect card data on their own forms.

> ⚠️ **Requires PCI DSS Level 1 SAQ-D compliance.**

### Direct Charge

**`POST /v1/charge`**

```json
{
  "amount": 50.00,
  "currency": "USD",
  "order_id": "ORD-67890",
  "description": "Premium subscription",
  "payment_method": "card",
  "card": {
    "number": "4111111111111111",
    "exp_month": "12",
    "exp_year": "2027",
    "cvv": "123",
    "name": "John Doe"
  },
  "customer": {
    "email": "john@example.com",
    "name": "John Doe",
    "phone": "+254712345678",
    "ip_address": "203.0.113.42"
  },
  "callback_url": "https://api.yourplatform.com/webhooks/sg",
  "return_url": "https://yourplatform.com/payment/complete",
  "metadata": {
    "subscription_id": "sub_abc123"
  }
}
```

### Response Scenarios

**A) Direct Success**
```json
{
  "status": "captured",
  "sentinel_transaction_id": "sg_txn_...",
  "provider": "pesapal",
  "provider_reference": "PP-abc123",
  "amount": 50.00,
  "currency": "USD",
  "gateway_response": "Approved",
  "channel": "card"
}
```

**B) 3DS Authentication Required**
```json
{
  "status": "3ds_redirect",
  "sentinel_transaction_id": "sg_txn_...",
  "redirect_url": "https://acs.bank.com/3ds/authenticate?id=xyz",
  "message": "Bank verification required"
}
```

Customer must be redirected to `redirect_url` to complete 3DS OTP verification. After authentication, they return to your `return_url` and SentinelGate sends the final webhook.

**C) Declined**
```json
{
  "status": "failed",
  "sentinel_transaction_id": "sg_txn_...",
  "error_reason": "Insufficient funds",
  "provider": "pesapal"
}
```

### 3DS Authentication Flow

```
Your Server  ──POST /v1/charge──▶  SentinelGate
             ◀──3ds_redirect────   (redirect_url)
                    │
Customer Browser  ──redirect──▶  Bank 3DS Page (OTP)
                  ◀──redirect──   (back to your return_url)
                    │
SentinelGate  ──POST webhook──▶  Your callback_url
              (status: captured)
```

---

## Mobile Money (STK Push)

For M-Pesa and other mobile money payments. A payment prompt is sent directly to the customer's phone.

### M-Pesa STK Push (KES)

Handled automatically when `currency` is `KES` and a phone number is provided. The customer receives an STK push prompt on their phone and enters their M-Pesa PIN to complete payment.

```json
{
  "amount": "1000.00",
  "currency": "KES",
  "order_id": "ORD-M001",
  "customer_phone": "254712345678",
  "callback_url": "https://yoursite.com/webhook",
  "return_url": "https://yoursite.com/thank-you"
}
```

---

## Payment Links

Generate shareable payment URLs for invoices, social media, or email.

### Create Payment Link

**`POST /pay`** (via backoffice)

Payment links support three types:

| Type | Use Case |
|------|----------|
| `STANDARD` | One-time payment |
| `INVOICE` | Invoice with reference number |
| `GAMING` | Gaming/entertainment top-up |

### Payment Link URL Format

```
https://sentinelgate.biz/pay/:token
```

QR code available at:
```
https://sentinelgate.biz/pay/:token/qr.png
```

### Payment Methods on Links

| Customer Currency | Payment Method | Provider |
|-------------------|---------------|----------|
| USD (Card) | Visa/Mastercard | Paystack redirect |
| KES (Mobile) | M-Pesa STK Push | BUNI |
| USD (Mobile) | Mobile Money | PesaPal |

---

## Webhooks & Callbacks

SentinelGate sends real-time HTTP POST notifications to your `callback_url` when a payment reaches a terminal state.

### Webhook Payload

```json
{
  "sentinel_transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "wc_order_id": "ORD-12345",
  "status": "captured",
  "amount": 125.00,
  "currency": "USD",
  "provider": "pesapal",
  "provider_reference": "f6732b57-a425-4d28-9e41-dab1d97c43dd",
  "gateway_response": "Approved",
  "channel": "card",
  "metadata": {}
}
```

### Webhook Fields

| Field | Description |
|-------|-------------|
| `sentinel_transaction_id` | SentinelGate unique reference |
| `wc_order_id` | Your `order_id` from the original request |
| `status` | `captured` (success) or `failed` |
| `amount` | Amount charged |
| `currency` | Currency code |
| `provider` | Provider used (e.g., `pesapal`, `paystack`, `buni`) |
| `provider_reference` | Provider's own transaction ID |
| `gateway_response` | Human-readable result |
| `channel` | `card`, `mobile_money`, or `bank_transfer` |

### Signature Verification

Webhooks include an HMAC-SHA256 signature:

```
X-SG-Signature: sha256=<hmac_hex>
```

Verify with:
```javascript
const expected = 'sha256=' + crypto
  .createHmac('sha256', YOUR_API_SECRET)
  .update(rawBody)
  .digest('hex');

if (signature !== expected) reject();
```

### Retry Policy

| Attempt | Delay |
|---------|-------|
| 1 | Immediate |
| 2 | 30 seconds |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

### Best Practices

- **Respond HTTP 200 within 15 seconds** — process asynchronously after acknowledging
- **Verify signatures** on every webhook
- **Deduplicate** using `sentinel_transaction_id` (you may receive duplicates)
- **Don't rely solely on webhooks** — also poll `/v1/transaction/:id` as fallback

---

## Transaction Status

### Get Transaction

**`GET /v1/transaction/:sentinel_transaction_id`**

```bash
curl https://sentinelgate.biz/v1/transaction/sg_txn_1771933613686_b9accd7562da \
  -H "X-API-Key: YOUR_KEY" \
  -H "X-API-Secret: YOUR_SECRET"
```

### Response

```json
{
  "sentinel_transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "status": "captured",
  "amount": 125.00,
  "currency": "USD",
  "provider": "pesapal",
  "provider_reference": "PP-TXN-abc123",
  "order_id": "ORD-12345",
  "channel": "card",
  "gateway_response": "Approved",
  "created_at": "2026-02-24T14:30:00Z",
  "updated_at": "2026-02-24T14:32:15Z"
}
```

### Status Reference

| Status | Description | Terminal? |
|--------|-------------|-----------|
| `pending` | Session created, awaiting customer | No |
| `processing` | Payment being processed | No |
| `captured` | Payment successful | ✅ Yes |
| `failed` | Payment declined or errored | ✅ Yes |
| `refunded` | Payment refunded | ✅ Yes |
| `cancelled` | Payment cancelled | ✅ Yes |

---

## Refunds

### Create Refund

**`POST /v1/refund`**

```json
{
  "sentinel_transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "amount": 50.00,
  "reason": "Customer requested partial refund"
}
```

Omit `amount` for a full refund.

### Response

```json
{
  "status": "refunded",
  "refund_id": "sg_ref_abc123",
  "sentinel_transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "amount": 50.00,
  "currency": "USD"
}
```

---

## Supported Currencies & Providers

SentinelGate automatically routes payments to the optimal provider based on currency and payment method.

| Currency | Card | Mobile Money | Providers |
|----------|------|-------------|-----------|
| **USD** | ✅ Visa, MC | — | PesaPal, Paystack, Brooks Routing |
| **KES** | ✅ Visa, MC | M-Pesa (STK Push) | PesaPal, BUNI |
| **GHS** | ✅ Visa, MC | MTN MoMo, Vodafone Cash | Hubtel, Paystack |
| **NGN** | ✅ Visa, MC, Verve | — | Paystack, Korapay |
| **TZS** | ✅ Visa, MC | Airtel Money | PesaPal |
| **UGX** | ✅ Visa, MC | MTN MoMo | PesaPal |

### Provider Details

| Provider | Currencies | Settlement | 3DS |
|----------|-----------|------------|-----|
| **PesaPal** | USD, KES, TZS, UGX | T+2 | Provider-hosted |
| **Paystack** | NGN, GHS, USD, ZAR | T+1 (NGN) | Redirect-based |
| **BUNI (Safaricom)** | KES | Real-time | N/A (M-Pesa) |
| **Hubtel** | GHS | T+1 | Provider-hosted |
| **Korapay** | NGN, GHS, KES | T+1 | Redirect-based |
| **Brooks Routing** | USD | Configurable | Multi-merchant |

---

## WooCommerce Plugin

### Installation

1. Download `sentinelgate-psp.zip`
2. WordPress Admin → Plugins → Add New → Upload Plugin
3. Activate the plugin
4. Go to WooCommerce → Settings → Payments → SentinelGate PSP

### Configuration

| Setting | Value |
|---------|-------|
| Enable | ✅ |
| Title | `Credit/Debit Card` |
| API URL | `https://sentinelgate.biz` |
| API Key | Your merchant key |
| API Secret | Your merchant secret |
| Mode | `Redirect` (recommended) |

### Plugin Modes

- **Redirect** — Customer redirected to provider page (no PCI needed)
- **iFrame** — Payment form embedded on your checkout (no PCI needed)
- **Direct** — Card form on your page (PCI DSS Level 1 required)

### M-Pesa Plugin

For KES M-Pesa STK Push payments, install `sentinelgate-mpesa-gateway.zip` alongside the PSP plugin.

---

## Shopify Integration

SentinelGate integrates with Shopify via a middleware webhook:

1. Register your store via `POST /v1/tenants`
2. Configure Shopify webhook: Order creation → `https://sentinelgate.biz/webhooks/shopify/order-created`
3. Middleware processes orders via BullMQ job queue with HMAC verification, idempotency, and automatic retries

---

## Error Handling

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| `200` | Success |
| `400` | Bad request — check parameters |
| `401` | Unauthorized — invalid credentials |
| `404` | Transaction/session not found |
| `429` | Rate limited |
| `500` | Server error — contact support |

### Error Response

```json
{
  "error": "Invalid currency. Supported: USD, KES, GHS, NGN",
  "code": "INVALID_CURRENCY"
}
```

### Common Error Codes

| Code | Description | Fix |
|------|-------------|-----|
| `INVALID_CREDENTIALS` | Wrong API key/secret | Verify credentials |
| `INVALID_CURRENCY` | Unsupported currency | Use: USD, KES, GHS, NGN |
| `INVALID_AMOUNT` | Zero/negative amount | Send positive number |
| `MISSING_CALLBACK` | No callback_url | Include webhook URL |
| `PROVIDER_ERROR` | Upstream provider issue | Retry or contact support |
| `DUPLICATE_ORDER` | order_id reused | Use unique references |
| `RATE_LIMITED` | Too many requests | Implement backoff |

### Rate Limits

| Endpoint | Limit |
|----------|-------|
| `/v1/hosted/create` | 60/min per merchant |
| `/v1/charge` | 30/min per merchant |
| `/v1/transaction/:id` | 120/min per merchant |
| `/v1/refund` | 10/min per merchant |

---

## Testing

### Test Your Integration

1. Create a hosted checkout with a small amount ($1.00 / KES 100)
2. Complete payment on the hosted page
3. Verify webhook received at your callback_url
4. Query transaction status to confirm `captured`

### Test Cards (Provider-Specific)

| Provider | Card Number | Result |
|----------|-------------|--------|
| Paystack | `4084 0840 8408 4081` | Success |
| Paystack | `4084 0840 8408 4099` | Declined |
| PesaPal | Use real cards (live only) | — |

### Health Check

```bash
curl https://sentinelgate.biz/v1/health
```

```json
{
  "status": "ok",
  "providers": {
    "pesapal": "up",
    "paystack": "up",
    "buni": "up",
    "hubtel": "up"
  }
}
```

---

## Postman Collection

### Using the Postman Collection

1. Open Postman
2. Import: `SentinelGate_API.postman_collection.json`
3. Set environment variables:

| Variable | Value |
|----------|-------|
| `base_url` | `https://sentinelgate.biz` |
| `api_key` | Your API key |
| `api_secret` | Your API secret |

4. Test endpoints starting with **Create Hosted Session**

### Example Requests in Collection

- Create Hosted Checkout
- Direct Charge (S2S)
- Get Transaction Status
- Create Refund
- Health Check

---

## Security Best Practices

- ✅ Store API secrets in environment variables, never in code
- ✅ All API calls over HTTPS from your server only
- ✅ Verify webhook signatures on every callback
- ✅ Implement idempotent webhook processing
- ✅ Verify transaction status after 3DS redirect
- ✅ Rate limit your own endpoints
- ✅ Never log full card numbers
- ✅ Use unique `order_id` for every transaction
- ❌ Never expose API secrets in client-side JavaScript
- ❌ Never store raw card data (unless PCI certified)

---

## FAQ

**Q: Which integration mode should I use?**
A: Start with **Hosted Checkout (Redirect)**. It's the simplest, requires no PCI compliance, and works with all payment methods.

**Q: How do I know which provider will process my payment?**
A: SentinelGate automatically routes based on currency and your merchant configuration. You don't need to specify a provider.

**Q: Can I accept multiple currencies?**
A: Yes — configure your merchant account for each currency. The API routes to the appropriate provider automatically.

**Q: What happens if the webhook fails?**
A: SentinelGate retries up to 5 times over 2+ hours. As a fallback, poll `GET /v1/transaction/:id` to check status.

**Q: Is there a sandbox/test environment?**
A: Contact SentinelGate for sandbox credentials. Some providers (Paystack) have test cards; others (PesaPal) only work with live cards.

**Q: How do I handle 3DS?**
A: For Hosted Checkout, 3DS is handled automatically. For Server-to-Server, check for `status: "3ds_redirect"` and redirect the customer to the `redirect_url`.

---

## Support

- **Email:** support@sentinelgate.biz
- **Documentation:** [GitHub — SentinelGateDocumentation](https://github.com/SentinelGateLLC/SentinelgateDocumentation)
- **Status:** `GET https://sentinelgate.biz/v1/health`
