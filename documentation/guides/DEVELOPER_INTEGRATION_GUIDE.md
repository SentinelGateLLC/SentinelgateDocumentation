# Developer Integration Guide

**For:** Software Developers & Technical Teams
**Level:** Intermediate to Advanced
**Last Updated:** February 24, 2026

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Authentication](#authentication)
3. [API Architecture](#api-architecture)
4. [Payment Integration](#payment-integration)
5. [Webhook Implementation](#webhook-implementation)
6. [Error Handling](#error-handling)
7. [Testing Strategy](#testing-strategy)
8. [Production Deployment](#production-deployment)
9. [Best Practices](#best-practices)
10. [Advanced Topics](#advanced-topics)

---

## Quick Start

### Prerequisites

- SentinelGate merchant credentials (API Key, API Secret, Webhook Secret)
- Development environment with HTTP client (Node.js, Python, PHP, etc.)
- SSL/TLS certificate on your server (HTTPS required for webhooks)
- Server capable of receiving HTTP POST requests

### Your First Payment (30 seconds)

Create a hosted checkout session and redirect the customer:

```bash
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "X-API-Key: sg_key_yourstore_abc123" \
  -H "X-API-Secret: sg_secret_yourstore_def456" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50.00",
    "currency": "USD",
    "order_id": "ORD-001",
    "description": "Test Payment",
    "customer_email": "buyer@example.com",
    "callback_url": "https://yoursite.com/webhook/sentinelgate",
    "return_url": "https://yoursite.com/order-confirmed",
    "cancel_url": "https://yoursite.com/checkout"
  }'
```

**Response:**

```json
{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "session_id": "sg_session_cc7d30bba805dca1c7c0828b",
  "redirect_url": "https://pay.hubtel.com/dcbe69362cec4c3ba9b5bc717518ae71",
  "hosted_url": "https://pay.hubtel.com/dcbe69362cec4c3ba9b5bc717518ae71",
  "status": "pending"
}
```

Redirect the customer to `redirect_url`. They pay on the hosted page. You receive a webhook when payment completes.

---

## Authentication

### Header-Based Authentication

All API requests require two headers:

```http
X-API-Key: sg_key_yourstore_abc123
X-API-Secret: sg_secret_yourstore_def456
Content-Type: application/json
```

### Environment Variables

```bash
# .env
SENTINELGATE_API_KEY=sg_key_yourstore_abc123
SENTINELGATE_API_SECRET=sg_secret_yourstore_def456
SENTINELGATE_WEBHOOK_SECRET=sg_whsec_yourstore_ghi789
SENTINELGATE_BASE_URL=https://sentinelgate.biz
```

### Security Rules

1. **Never** expose credentials in client-side / frontend code
2. **Always** use HTTPS — SentinelGate rejects plain HTTP in production
3. **Store** credentials in environment variables, not source code
4. **Rotate** credentials if a team member leaves or a breach is suspected
5. **Never** commit `.env` files to Git

---

## API Architecture

### Base URL

```
Production:  https://sentinelgate.biz
```

All endpoints are served over HTTPS via Apache reverse proxy on port 443. The underlying service runs on port 3003 internally.

### Core Endpoints

| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| POST | `/v1/hosted/create` | Required | Create hosted payment session (recommended) |
| GET | `/v1/hosted/pay/:sessionId` | None | Hosted payment page (customer-facing) |
| POST | `/v1/charge` | Required | Direct card charge (PCI DSS required) |
| GET | `/v1/transaction/:txnId` | Required | Query transaction status |
| POST | `/v1/refund` | Required | Process refund |

### Webhook Endpoints (Provider Callbacks)

| Endpoint | Provider |
|----------|----------|
| `/hubtel/callback` | Hubtel (Ghana cards + mobile money) |
| `/buni/mpesa/callback` | BUNI/KCB M-Pesa (Kenya) |
| `/pesapal/callback` | Pesapal (East Africa) |
| `/paystack/callback` | Paystack (Africa) |
| `/emergent/callback` | Emergent/InterPay (Cards) |
| `/korapay/callback` | Korapay (Cards + Mobile) |

### Shopify Middleware Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /webhooks/shopify/order-created` | Order creation webhook |
| `POST /shopify/webhook/orders/paid` | Order paid notification |
| `POST /shopify/webhook/orders/cancelled` | Order cancellation |
| `POST /shopify/webhook/refunds/create` | Refund processing |
| `GET /pay/shopify-redirect` | Customer payment redirect |

### Payment Links

| Endpoint | Purpose |
|----------|---------|
| `GET /pay/:token` | Payment link checkout page |
| `GET /pay/:token/qr.png` | QR code for payment link |

### Request/Response Format

- **Content-Type:** `application/json`
- **Character Encoding:** UTF-8
- **Date Format:** ISO 8601 (`2026-02-24T06:30:00Z`)
- **Amount Format:** String in decimal dollars (`"191.00"`, not cents)

---

## Payment Integration

### Recommended Flow: Hosted Checkout

This is the safest and simplest integration. No PCI compliance required on your side.

```
┌─────────────┐
│  Customer    │  1. Places order on your site
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Your Server  │  2. POST /v1/hosted/create
└──────┬──────┘     (amount, order_id, callback_url, return_url)
       │
       ▼
┌─────────────┐
│ SentinelGate │  3. Returns redirect_url
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Payment     │  4. Customer enters card/mobile money details
│  Page        │     3D Secure / OTP handled automatically
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Provider    │  5. Hubtel / Pesapal / Paystack processes payment
│  (Hubtel,    │
│   BUNI, etc) │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ SentinelGate │  6. Webhook POST to your callback_url
└──────┬──────┘     { status: "captured", sentinel_transaction_id, amount }
       │
       ├──────────────────────────┐
       ▼                          ▼
┌─────────────┐          ┌─────────────┐
│ Your Server  │          │  Customer    │
│ Updates order│          │  Redirected  │
│ status       │          │  to return_url│
└─────────────┘          └─────────────┘
```

### Implementation: Node.js

```javascript
const axios = require('axios');

const SENTINEL_BASE = process.env.SENTINELGATE_BASE_URL; // https://sentinelgate.biz
const API_KEY = process.env.SENTINELGATE_API_KEY;        // sg_key_...
const API_SECRET = process.env.SENTINELGATE_API_SECRET;  // sg_secret_...

// Step 1: Create a payment session
async function createPaymentSession(order) {
  const response = await axios.post(
    `${SENTINEL_BASE}/v1/hosted/create`,
    {
      amount: order.total.toFixed(2),       // "191.00"
      currency: order.currency,              // "USD"
      order_id: order.id,                    // "ORD-7700"
      description: `Order #${order.id}`,
      customer_email: order.customerEmail,
      customer_name: order.customerName,
      callback_url: `${process.env.YOUR_DOMAIN}/webhook/sentinelgate`,
      return_url: `${process.env.YOUR_DOMAIN}/order/${order.id}/confirmed`,
      cancel_url: `${process.env.YOUR_DOMAIN}/checkout`
    },
    {
      headers: {
        'X-API-Key': API_KEY,
        'X-API-Secret': API_SECRET,
        'Content-Type': 'application/json'
      },
      timeout: 30000
    }
  );

  return response.data;
}

// Step 2: Redirect customer
app.post('/checkout', async (req, res) => {
  const order = await createOrder(req.body);

  const payment = await createPaymentSession(order);

  // Save transaction ID to order
  await db.orders.update({
    where: { id: order.id },
    data: {
      sentinelTxnId: payment.sentinel_transaction_id,
      paymentStatus: 'pending'
    }
  });

  // Redirect customer to payment page
  res.redirect(302, payment.redirect_url);
});
```

### Implementation: Python

```python
import requests
import os

SENTINEL_BASE = os.environ['SENTINELGATE_BASE_URL']
API_KEY = os.environ['SENTINELGATE_API_KEY']
API_SECRET = os.environ['SENTINELGATE_API_SECRET']

def create_payment_session(order):
    response = requests.post(
        f"{SENTINEL_BASE}/v1/hosted/create",
        json={
            "amount": f"{order['total']:.2f}",
            "currency": order['currency'],
            "order_id": order['id'],
            "description": f"Order #{order['id']}",
            "customer_email": order['email'],
            "callback_url": f"{os.environ['YOUR_DOMAIN']}/webhook/sentinelgate",
            "return_url": f"{os.environ['YOUR_DOMAIN']}/order/{order['id']}/confirmed",
            "cancel_url": f"{os.environ['YOUR_DOMAIN']}/checkout"
        },
        headers={
            "X-API-Key": API_KEY,
            "X-API-Secret": API_SECRET,
            "Content-Type": "application/json"
        },
        timeout=30
    )
    response.raise_for_status()
    return response.json()
```

### Implementation: PHP

```php
function createPaymentSession($order) {
    $ch = curl_init();

    curl_setopt_array($ch, [
        CURLOPT_URL => getenv('SENTINELGATE_BASE_URL') . '/v1/hosted/create',
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 30,
        CURLOPT_HTTPHEADER => [
            'X-API-Key: ' . getenv('SENTINELGATE_API_KEY'),
            'X-API-Secret: ' . getenv('SENTINELGATE_API_SECRET'),
            'Content-Type: application/json'
        ],
        CURLOPT_POSTFIELDS => json_encode([
            'amount' => number_format($order['total'], 2, '.', ''),
            'currency' => $order['currency'],
            'order_id' => $order['id'],
            'description' => 'Order #' . $order['id'],
            'customer_email' => $order['email'],
            'callback_url' => getenv('YOUR_DOMAIN') . '/webhook/sentinelgate',
            'return_url' => getenv('YOUR_DOMAIN') . '/order/' . $order['id'] . '/confirmed',
            'cancel_url' => getenv('YOUR_DOMAIN') . '/checkout'
        ])
    ]);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($httpCode !== 200) {
        throw new Exception("Payment creation failed: $response");
    }

    return json_decode($response, true);
}
```

---

## Webhook Implementation

### Setting Up Your Webhook Endpoint

When a payment completes (or fails), SentinelGate sends a POST request to your `callback_url`.

**Webhook Payload:**

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

**Webhook Statuses:**

| Status | Meaning | Action |
|--------|---------|--------|
| `captured` | Payment successful | Mark order as paid, fulfill |
| `failed` | Payment declined | Notify customer, allow retry |
| `refunded` | Refund processed | Process return |

### Signature Verification

Every webhook includes an `X-Sentinel-Signature` header. **Always verify before processing.**

**Node.js:**

```javascript
const express = require('express');
const crypto = require('crypto');

const WEBHOOK_SECRET = process.env.SENTINELGATE_WEBHOOK_SECRET;

// CRITICAL: Use express.raw() to get the raw body for signature verification
app.post('/webhook/sentinelgate',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    // 1. Verify signature
    const signature = req.headers['x-sentinel-signature'];
    if (!verifySignature(req.body, signature, WEBHOOK_SECRET)) {
      return res.status(401).send('Invalid signature');
    }

    // 2. Respond immediately (SentinelGate expects response within 15s)
    res.status(200).send('OK');

    // 3. Parse and process asynchronously
    const event = JSON.parse(req.body);
    processWebhook(event).catch(console.error);
  }
);

function verifySignature(rawBody, signature, secret) {
  if (!signature) return false;
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  try {
    return crypto.timingSafeEqual(
      Buffer.from(expected),
      Buffer.from(signature)
    );
  } catch {
    return false;
  }
}

async function processWebhook(event) {
  const { sentinel_transaction_id, status, amount, wc_order_id } = event;

  switch (status) {
    case 'captured':
      await db.orders.update({
        where: { id: wc_order_id },
        data: {
          paymentStatus: 'paid',
          sentinelTxnId: sentinel_transaction_id,
          paidAt: new Date()
        }
      });
      await sendOrderConfirmationEmail(wc_order_id);
      break;

    case 'failed':
      await db.orders.update({
        where: { id: wc_order_id },
        data: { paymentStatus: 'failed' }
      });
      break;

    case 'refunded':
      await db.orders.update({
        where: { id: wc_order_id },
        data: { paymentStatus: 'refunded' }
      });
      break;
  }
}
```

**Python:**

```python
import hmac
import hashlib
from flask import Flask, request

app = Flask(__name__)

@app.route('/webhook/sentinelgate', methods=['POST'])
def handle_webhook():
    # 1. Verify signature
    signature = request.headers.get('X-Sentinel-Signature', '')
    raw_body = request.get_data()

    if not verify_signature(raw_body, signature, os.environ['SENTINELGATE_WEBHOOK_SECRET']):
        return 'Invalid signature', 401

    # 2. Parse event
    event = request.get_json()

    # 3. Process
    if event['status'] == 'captured':
        mark_order_as_paid(event['wc_order_id'], event['sentinel_transaction_id'])
    elif event['status'] == 'failed':
        mark_order_as_failed(event['wc_order_id'])

    return 'OK', 200

def verify_signature(raw_body, signature, secret):
    expected = 'sha256=' + hmac.new(
        secret.encode(),
        raw_body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**PHP:**

```php
<?php
$rawBody = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_SENTINEL_SIGNATURE'] ?? '';
$secret = getenv('SENTINELGATE_WEBHOOK_SECRET');

// Verify signature
$expected = 'sha256=' . hash_hmac('sha256', $rawBody, $secret);
if (!hash_equals($expected, $signature)) {
    http_response_code(401);
    echo 'Invalid signature';
    exit;
}

// Process
$event = json_decode($rawBody, true);

if ($event['status'] === 'captured') {
    // Update order to paid
    markOrderAsPaid($event['wc_order_id'], $event['sentinel_transaction_id']);
}

http_response_code(200);
echo 'OK';
```

### Webhook Best Practices

1. **Respond with 200 immediately** — process asynchronously. SentinelGate expects a response within 15 seconds.
2. **Verify signatures always** — never process unverified webhooks.
3. **Handle duplicates** — use `sentinel_transaction_id` as an idempotency key. Process each transaction only once.
4. **Use HTTPS** — plain HTTP callback URLs are rejected.
5. **Retry behavior** — failed webhooks are retried up to 3 times with exponential backoff (1 min, 5 min, 15 min).

---

## Error Handling

### HTTP Status Codes

| Status | Meaning | Action |
|--------|---------|--------|
| 200 | Success | Process response |
| 400 | Bad Request | Check required fields in request body |
| 401 | Unauthorized | Verify API Key and Secret headers |
| 404 | Not Found | Invalid transaction ID or session |
| 429 | Rate Limited | Back off and retry after delay |
| 500 | Server Error | Retry with exponential backoff |

### Error Response Format

```json
{
  "error": "INVALID_REQUEST",
  "message": "amount is required"
}
```

### Common Error Codes

| Code | Cause | Fix |
|------|-------|-----|
| `MISSING_API_KEY` | No `X-API-Key` header | Add the header |
| `INVALID_API_KEY` | Key doesn't match any merchant | Check credentials |
| `INVALID_REQUEST` | Missing required field | Check required fields |
| `SESSION_EXPIRED` | Payment session timed out | Create a new session |
| `PROVIDER_ERROR` | Payment provider returned error | Check `message` for details |
| `RATE_LIMIT_EXCEEDED` | Too many requests | Implement backoff |

### Retry Strategy

```javascript
async function createPaymentWithRetry(payload, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await createPaymentSession(payload);
    } catch (error) {
      const status = error.response?.status;

      // Don't retry client errors (except rate limiting)
      if (status >= 400 && status < 500 && status !== 429) {
        throw error;
      }

      if (attempt === maxRetries) throw error;

      // Exponential backoff: 2s, 4s, 8s
      const delay = Math.pow(2, attempt) * 1000;
      console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

---

## Testing Strategy

### Test Workflow

1. Create a hosted checkout session using your real credentials
2. Redirect to the payment page
3. Use a real card or mobile money with a small amount ($1 / GHS 5)
4. Verify webhook is received
5. Verify order status updated

### Testing Webhooks Locally

Use [ngrok](https://ngrok.com) to expose your local server:

```bash
# Terminal 1: Start your server
npm run dev
# Server running on http://localhost:3000

# Terminal 2: Expose with ngrok
ngrok http 3000

# Use the ngrok URL as your callback_url:
# https://abc123.ngrok-free.app/webhook/sentinelgate
```

### Webhook Test Script

Send a mock webhook to test your handler:

```bash
# Generate a valid signature
SECRET="sg_whsec_yourstore_ghi789"
BODY='{"sentinel_transaction_id":"sg_txn_test_123","wc_order_id":"TEST-001","status":"captured","amount":1.00,"currency":"USD","provider":"hubtel","gateway_response":"Approved"}'
SIGNATURE="sha256=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')"

# Send to your endpoint
curl -X POST http://localhost:3000/webhook/sentinelgate \
  -H "Content-Type: application/json" \
  -H "X-Sentinel-Signature: $SIGNATURE" \
  -d "$BODY"
```

### Rate Limits (Know Before You Test)

| Endpoint | Limit |
|----------|-------|
| `POST /v1/hosted/create` | 60/min |
| `POST /v1/charge` | 30/min |
| `GET /v1/transaction/:id` | 120/min |
| `POST /v1/refund` | 10/min |

---

## Production Deployment

### Pre-Launch Checklist

- [ ] Using production credentials (`sg_key_`, `sg_secret_`, `sg_whsec_`)
- [ ] Callback URL uses HTTPS with valid SSL certificate
- [ ] Webhook endpoint responds within 15 seconds
- [ ] Webhook signature verification is implemented
- [ ] Idempotency handling for duplicate webhooks
- [ ] Error logging with transaction IDs
- [ ] Return URL and cancel URL point to production pages
- [ ] Test transaction completed successfully
- [ ] Small real payment ($1) verified end-to-end

### Production Environment Variables

```bash
# .env (production)
NODE_ENV=production
SENTINELGATE_API_KEY=sg_key_yourstore_abc123
SENTINELGATE_API_SECRET=sg_secret_yourstore_def456
SENTINELGATE_WEBHOOK_SECRET=sg_whsec_yourstore_ghi789
SENTINELGATE_BASE_URL=https://sentinelgate.biz
YOUR_DOMAIN=https://yourproductionsite.com
```

### Health Check

Verify the SentinelGate service is reachable:

```bash
curl -s https://sentinelgate.biz/health
# Expected: {"status":"ok","providers":7}
```

---

## Best Practices

### 1. Always Use Hosted Checkout

Unless you have PCI DSS Level 1 certification, use `/v1/hosted/create` instead of `/v1/charge`. Hosted checkout handles card form rendering, 3D Secure, and OTP verification automatically.

### 2. Idempotency

Use `sentinel_transaction_id` to prevent processing the same payment twice:

```javascript
async function processWebhook(event) {
  // Check if already processed
  const existing = await db.payments.findUnique({
    where: { sentinelTxnId: event.sentinel_transaction_id }
  });
  if (existing) {
    console.log('Duplicate webhook, skipping:', event.sentinel_transaction_id);
    return;
  }

  // Process payment...
}
```

### 3. Timeout Handling

Set a 30-second timeout on API calls. Payment sessions don't expire immediately if the API call times out — query the transaction status before creating a new session:

```javascript
try {
  const session = await createPaymentSession(order);
} catch (error) {
  if (error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT') {
    // Don't create another session — check if the first one was created
    const txn = await queryTransaction(order.sentinelTxnId);
    if (txn && txn.status === 'pending') {
      // Session exists, redirect customer
      return res.redirect(txn.redirect_url);
    }
  }
  throw error;
}
```

### 4. Logging

Log all payment operations with transaction IDs for debugging:

```javascript
console.log('[Payment] Session created', {
  orderId: order.id,
  txnId: payment.sentinel_transaction_id,
  amount: order.total,
  currency: order.currency
});

console.log('[Webhook] Received', {
  txnId: event.sentinel_transaction_id,
  status: event.status,
  provider: event.provider
});
```

### 5. Database Transactions

Wrap order updates in database transactions to prevent inconsistent state:

```javascript
await prisma.$transaction(async (tx) => {
  await tx.order.update({
    where: { id: orderId },
    data: { paymentStatus: 'paid', paidAt: new Date() }
  });
  await tx.inventory.update({
    where: { productId: order.productId },
    data: { stock: { decrement: order.quantity } }
  });
});
```

---

## Advanced Topics

### Querying Transaction Status

If you need to verify a payment outside of webhooks:

```javascript
async function checkPaymentStatus(txnId) {
  const response = await axios.get(
    `${SENTINEL_BASE}/v1/transaction/${txnId}`,
    {
      headers: {
        'X-API-Key': API_KEY,
        'X-API-Secret': API_SECRET
      },
      timeout: 15000
    }
  );
  return response.data;
}

// Usage
const status = await checkPaymentStatus('sg_txn_1771888643979_cfe07b9d7fe6');
// { sentinel_transaction_id, status, amount, currency, provider, created_at }
```

### Processing Refunds

```javascript
async function refundPayment(txnId, amount, reason) {
  const response = await axios.post(
    `${SENTINEL_BASE}/v1/refund`,
    {
      sentinel_transaction_id: txnId,
      amount: amount,    // Partial refund: less than original. Full refund: omit or use full amount.
      reason: reason
    },
    {
      headers: {
        'X-API-Key': API_KEY,
        'X-API-Secret': API_SECRET,
        'Content-Type': 'application/json'
      }
    }
  );
  return response.data;
  // { sentinel_transaction_id, refund_id, amount, status: "refunded" }
}
```

### Payment Links (No Frontend Required)

SentinelGate supports shareable payment links with QR codes — useful for invoices, POS, or email billing:

```
Checkout page:  https://sentinelgate.biz/pay/<token>
QR code image:  https://sentinelgate.biz/pay/<token>/qr.png
```

Link types: `STANDARD` (general payments), `INVOICE` (B2B), `GAMING` (variable amount).

Contact SentinelGate to set up payment links for your merchant account.

### Provider-Specific Behavior

| Provider | Methods | Currency | Notes |
|----------|---------|----------|-------|
| **Hubtel** | Card + Mobile Money | GHS (USD converted) | Redirects to pay.hubtel.com |
| **BUNI/KCB** | M-Pesa STK Push | KES | Sends push notification to phone |
| **Pesapal** | Card + Mobile | KES, UGX, USD | Redirects to pesapal checkout |
| **Paystack** | Card + Bank | NGN, GHS, USD | Redirects to paystack checkout |
| **Emergent** | Card | USD, multi-currency | Pending server fix |
| **Korapay** | Card + Mobile | NGN, GHS, USD | Pending credentials |
| **Brooks** | Routed (multiple processors) | USD | Daily limits + rollover |

SentinelGate routes payments to the appropriate provider based on your merchant configuration. You don't need to specify a provider — just send amount, currency, and let the routing engine handle it.

---

## Support

| Resource | Location |
|----------|----------|
| **API Reference** | [API_REFERENCE.md](./API_REFERENCE.md) |
| **WooCommerce Plugin** | [WOOCOMMERCE_INTEGRATION.md](./WOOCOMMERCE_INTEGRATION.md) |
| **Shopify Integration** | [SHOPIFY_INTEGRATION.md](./SHOPIFY_INTEGRATION.md) |
| **Troubleshooting** | [COMMON_ISSUES.md](./COMMON_ISSUES.md) |
| **Email Support** | support@sentinelgate.biz |
| **Health Check** | https://sentinelgate.biz/health |

---

*© 2026 SentinelGate. All rights reserved.*
