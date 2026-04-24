# SentinelGate — Server-to-Server Integration Guide

> For payment providers and PCI-compliant merchants who need direct API integration without redirects.

---

## Overview

Server-to-server (S2S) integration allows you to:

- Collect card details on your own secure form
- Submit payments directly via API
- Handle 3DS authentication flows
- Receive real-time webhook notifications
- Query transaction status programmatically

**Prerequisites:**
- PCI DSS Level 1 SAQ-D compliance (for handling raw card data)
- Valid SentinelGate API credentials
- HTTPS-enabled webhook endpoint
- Server-side implementation (never expose API secrets in client code)

---

## Authentication

Every request must include your merchant credentials:

```bash
curl -X POST https://sentinelgate.biz/v1/charge \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "X-API-Secret: YOUR_API_SECRET" \
  -d '{ ... }'
```

---

## Direct Charge Flow

### Step 1: Submit Payment

**`POST /v1/charge`**

```json
{
  "amount": 99.99,
  "currency": "USD",
  "order_id": "INV-2026-0042",
  "description": "Premium subscription — annual",
  "payment_method": "card",
  "card": {
    "number": "4111111111111111",
    "exp_month": "03",
    "exp_year": "2028",
    "cvv": "456",
    "name": "Jane Smith"
  },
  "customer": {
    "email": "jane@example.com",
    "name": "Jane Smith",
    "phone": "+1-555-0123",
    "ip_address": "203.0.113.42"
  },
  "callback_url": "https://api.yourplatform.com/webhooks/sg",
  "return_url": "https://yourplatform.com/payment/complete",
  "metadata": {
    "user_id": "usr_12345",
    "plan": "annual_premium"
  }
}
```

### Step 2: Handle Response

There are three possible outcomes:

#### A) Direct Success (No 3DS)

```json
{
  "status": "captured",
  "sentinel_transaction_id": "sg_txn_...",
  "provider": "pesapal",
  "provider_reference": "PP-abc123",
  "amount": 99.99,
  "currency": "USD",
  "gateway_response": "Approved",
  "channel": "card"
}
```

→ Payment is complete. Update your order.

#### B) 3DS Authentication Required

```json
{
  "status": "3ds_redirect",
  "sentinel_transaction_id": "sg_txn_...",
  "redirect_url": "https://acs.bank.com/3ds?id=xyz",
  "message": "Bank verification required"
}
```

→ Redirect the customer's browser to `redirect_url`. See [3DS Flow](#3ds-authentication-flow) below.

#### C) Declined

```json
{
  "status": "failed",
  "sentinel_transaction_id": "sg_txn_...",
  "error_reason": "Insufficient funds",
  "provider": "pesapal"
}
```

→ Display error to customer. They can retry with a different card.

---

## 3DS Authentication Flow

When a card requires 3D Secure verification:

```
┌─────────────┐     POST /v1/charge      ┌──────────────┐
│ Your Server  │ ──────────────────────── │ SentinelGate │
│              │ ◄── 3ds_redirect ─────── │              │
└──────┬───────┘                          └──────┬───────┘
       │                                         │
       │ 302 redirect_url                        │
       ▼                                         │
┌─────────────┐                                  │
│  Customer's  │     OTP / Biometric              │
│   Browser    │ ─── authentication ──→ Bank ACS  │
│              │ ◄── redirect back ───            │
└──────┬───────┘                                  │
       │                                         │
       │ Redirected to return_url                 │
       ▼                                         │
┌─────────────┐                          ┌──────────────┐
│ Your Server  │ ◄── webhook (captured) ─ │ SentinelGate │
│ (return_url) │                          │              │
└─────────────┘                          └──────────────┘
```

**Implementation:**

```javascript
// 1. Call /v1/charge
const response = await fetch('https://sentinelgate.biz/v1/charge', { ... });
const result = await response.json();

if (result.status === '3ds_redirect') {
  // 2. Store sentinel_transaction_id for later verification
  await saveTransaction(result.sentinel_transaction_id, orderId);
  
  // 3. Redirect customer to bank's 3DS page
  res.redirect(302, result.redirect_url);
  // Customer completes OTP on bank page, then is redirected to your return_url
}

// 4. On your return_url handler — verify payment status
app.get('/payment/complete', async (req, res) => {
  const txnId = await getStoredTransaction(orderId);
  const status = await fetch(`https://sentinelgate.biz/v1/transaction/${txnId}`, {
    headers: { 'X-API-Key': API_KEY, 'X-API-Secret': API_SECRET }
  });
  const result = await status.json();
  
  if (result.status === 'captured') {
    // Payment successful
    await updateOrder(orderId, 'paid');
    res.render('success');
  } else {
    res.render('failed', { reason: result.error_reason });
  }
});
```

---

## Webhook Integration

### Setup

When creating a payment, include your `callback_url`. SentinelGate will POST to this URL when the transaction reaches a terminal state.

### Receiving Webhooks

```javascript
// Express.js example
app.post('/webhooks/sg', express.json(), (req, res) => {
  // 1. Verify signature
  const signature = req.headers['x-sg-signature'];
  const expectedSig = 'sha256=' + crypto
    .createHmac('sha256', API_SECRET)
    .update(JSON.stringify(req.body))
    .digest('hex');
  
  if (signature !== expectedSig) {
    return res.status(401).send('Invalid signature');
  }
  
  // 2. Respond immediately (before processing)
  res.status(200).json({ received: true });
  
  // 3. Process asynchronously
  const { sentinel_transaction_id, wc_order_id, status, amount, currency } = req.body;
  
  processPaymentAsync({
    txnId: sentinel_transaction_id,
    orderId: wc_order_id,
    status,  // "captured" or "failed"
    amount,
    currency
  });
});
```

### Webhook Payload Reference

```json
{
  "sentinel_transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "wc_order_id": "INV-2026-0042",
  "status": "captured",
  "amount": 99.99,
  "currency": "USD",
  "provider": "pesapal",
  "provider_reference": "f6732b57-a425-4d28-9e41-dab1d97c43dd",
  "gateway_response": "Approved",
  "channel": "card",
  "metadata": {
    "user_id": "usr_12345",
    "plan": "annual_premium"
  }
}
```

---

## Querying Transaction Status

Use this as a fallback if webhooks are delayed or for order reconciliation.

**`GET /v1/transaction/:sentinel_transaction_id`**

```bash
curl -s https://sentinelgate.biz/v1/transaction/sg_txn_1771933613686_b9accd7562da \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "X-API-Secret: YOUR_API_SECRET"
```

**Response:**

```json
{
  "sentinel_transaction_id": "sg_txn_1771933613686_b9accd7562da",
  "status": "captured",
  "amount": 99.99,
  "currency": "USD",
  "provider": "pesapal",
  "provider_reference": "PP-abc123",
  "order_id": "INV-2026-0042",
  "channel": "card",
  "gateway_response": "Approved",
  "created_at": "2026-02-24T14:30:00Z",
  "updated_at": "2026-02-24T14:32:15Z"
}
```

---

## Code Examples

### Python

```python
import requests, hmac, hashlib, json

API_KEY = "sg_key_your_key"
API_SECRET = "sg_secret_your_secret"
BASE_URL = "https://sentinelgate.biz"

def charge_card(amount, currency, card, customer, order_id, callback_url):
    resp = requests.post(f"{BASE_URL}/v1/charge", json={
        "amount": amount,
        "currency": currency,
        "order_id": order_id,
        "payment_method": "card",
        "card": card,
        "customer": customer,
        "callback_url": callback_url
    }, headers={
        "X-API-Key": API_KEY,
        "X-API-Secret": API_SECRET,
        "Content-Type": "application/json"
    })
    return resp.json()

def verify_webhook(payload_bytes, signature):
    expected = "sha256=" + hmac.new(
        API_SECRET.encode(), payload_bytes, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

# Usage
result = charge_card(
    amount=50.00,
    currency="USD",
    card={"number": "4111111111111111", "exp_month": "12", "exp_year": "2028", "cvv": "123"},
    customer={"email": "test@example.com", "name": "Test User"},
    order_id="ORD-001",
    callback_url="https://api.yoursite.com/webhooks/sg"
)

if result["status"] == "captured":
    print(f"Payment successful: {result['sentinel_transaction_id']}")
elif result["status"] == "3ds_redirect":
    print(f"Redirect customer to: {result['redirect_url']}")
else:
    print(f"Payment failed: {result.get('error_reason')}")
```

### PHP

```php
<?php
class SentinelGateClient {
    private $apiKey;
    private $apiSecret;
    private $baseUrl = 'https://sentinelgate.biz';

    public function __construct($apiKey, $apiSecret) {
        $this->apiKey = $apiKey;
        $this->apiSecret = $apiSecret;
    }

    public function charge($amount, $currency, $card, $customer, $orderId, $callbackUrl) {
        $payload = json_encode([
            'amount' => $amount,
            'currency' => $currency,
            'order_id' => $orderId,
            'payment_method' => 'card',
            'card' => $card,
            'customer' => $customer,
            'callback_url' => $callbackUrl,
        ]);

        $ch = curl_init("{$this->baseUrl}/v1/charge");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                "X-API-Key: {$this->apiKey}",
                "X-API-Secret: {$this->apiSecret}",
            ],
        ]);
        $response = curl_exec($ch);
        curl_close($ch);
        return json_decode($response, true);
    }

    public function verifyWebhook($payloadBody, $signatureHeader) {
        $expected = 'sha256=' . hash_hmac('sha256', $payloadBody, $this->apiSecret);
        return hash_equals($expected, $signatureHeader);
    }

    public function getTransaction($txnId) {
        $ch = curl_init("{$this->baseUrl}/v1/transaction/{$txnId}");
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                "X-API-Key: {$this->apiKey}",
                "X-API-Secret: {$this->apiSecret}",
            ],
        ]);
        $response = curl_exec($ch);
        curl_close($ch);
        return json_decode($response, true);
    }
}

// Usage
$sg = new SentinelGateClient('sg_key_...', 'sg_secret_...');

$result = $sg->charge(50.00, 'USD', [
    'number' => '4111111111111111',
    'exp_month' => '12',
    'exp_year' => '2028',
    'cvv' => '123',
    'name' => 'Test User'
], [
    'email' => 'test@example.com',
    'name' => 'Test User'
], 'ORD-001', 'https://yoursite.com/webhook');

if ($result['status'] === 'captured') {
    echo "Success: " . $result['sentinel_transaction_id'];
} elseif ($result['status'] === '3ds_redirect') {
    header("Location: " . $result['redirect_url']);
} else {
    echo "Failed: " . ($result['error_reason'] ?? 'Unknown');
}
```

### Node.js

```javascript
const crypto = require('crypto');
const axios = require('axios');

class SentinelGate {
  constructor(apiKey, apiSecret) {
    this.apiKey = apiKey;
    this.apiSecret = apiSecret;
    this.baseUrl = 'https://sentinelgate.biz';
  }

  async charge({ amount, currency, card, customer, orderId, callbackUrl }) {
    const { data } = await axios.post(`${this.baseUrl}/v1/charge`, {
      amount, currency, order_id: orderId,
      payment_method: 'card', card, customer,
      callback_url: callbackUrl,
    }, {
      headers: {
        'X-API-Key': this.apiKey,
        'X-API-Secret': this.apiSecret,
      }
    });
    return data;
  }

  async getTransaction(txnId) {
    const { data } = await axios.get(`${this.baseUrl}/v1/transaction/${txnId}`, {
      headers: { 'X-API-Key': this.apiKey, 'X-API-Secret': this.apiSecret }
    });
    return data;
  }

  verifyWebhook(rawBody, signature) {
    const expected = 'sha256=' + crypto
      .createHmac('sha256', this.apiSecret)
      .update(rawBody)
      .digest('hex');
    return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
  }
}

module.exports = SentinelGate;
```

---

## Security Checklist

- [ ] API secrets stored in environment variables, never in client code
- [ ] All API calls made server-side over HTTPS
- [ ] Webhook signatures verified on every callback
- [ ] Idempotent webhook processing (handle duplicates)
- [ ] Transaction status verified via GET after 3DS redirect
- [ ] PCI DSS compliance maintained for card data handling
- [ ] Rate limits implemented on your end
- [ ] Error responses don't expose internal details to end users
