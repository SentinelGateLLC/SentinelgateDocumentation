# Mobile Money Payment Method Documentation

**For:** Developers, Merchants & Technical Teams
**Last Updated:** February 24, 2026

---

## Overview

SentinelGate supports mobile money payments across multiple African markets. Mobile money lets customers pay directly from their phone wallets — no bank account or card required. It is the dominant payment method in many African countries.

This document covers how mobile money works on SentinelGate, which providers are supported, and how to integrate.

---

## Supported Mobile Money Providers

### Currently Active

| Provider | Country | Networks | Currency | Status |
|----------|---------|----------|----------|--------|
| **Hubtel** | Ghana | MTN, Vodafone Cash, AirtelTigo | GHS | ✅ Live |
| **BUNI/KCB** | Kenya | M-Pesa (Safaricom) | KES | ✅ Live (UAT) |

### Pending Activation

| Provider | Country | Networks | Currency | Blocker |
|----------|---------|----------|----------|---------|
| **Pesapal** | Kenya, Uganda, Tanzania | M-Pesa, Airtel Money, MTN MoMo | KES, UGX, TZS | Invalid consumer key |

### Available in Code (Not Yet Configured)

| Adapter | Path | Market |
|---------|------|--------|
| MTN MoMo | `src/providers/mobile-money/mtn-momo/mtn-momo.adapter.ts` | West & East Africa |
| Airtel Money | `src/providers/mobile-money/airtel/airtel.adapter.ts` | East & Central Africa |
| Orange Money | `src/providers/mobile-money/orange/orange.adapter.ts` | West & Central Africa |
| Tingg (Cellulant) | `src/providers/mobile-money/tingg/tingg.adapter.ts` | Pan-African |
| Generic M-Pesa | `src/providers/mobile-money/mpesa/mpesa.adapter.ts` | Kenya, Tanzania |

---

## How Mobile Money Works

### For Merchants (Non-Technical)

Mobile money is like a digital wallet on the customer's phone. When they buy from your store:

1. Customer selects "Mobile Money" or "M-Pesa" at checkout
2. They enter their phone number (or receive a push notification)
3. A payment prompt appears on their phone
4. They enter their mobile money PIN to approve
5. Money moves from their wallet to your settlement account
6. Your store is notified that payment is complete

**No card, no bank account needed.** The customer just needs a phone with an active mobile money wallet.

### For Developers (Technical)

Mobile money payments use two patterns:

**Pattern 1: STK Push (Server-Initiated)**
Your server sends the customer's phone number to SentinelGate. SentinelGate triggers an STK (SIM Toolkit) push notification on the customer's phone. The customer sees a payment prompt and enters their PIN to approve.

```
Your Server → SentinelGate → Provider → STK Push to customer's phone
                                              ↓
                                    Customer enters PIN
                                              ↓
Provider confirms → SentinelGate webhook → Your Server updates order
```

Used by: **BUNI M-Pesa**

**Pattern 2: Redirect Checkout (Customer-Initiated)**
The customer is redirected to the provider's payment page, where they select their mobile money network and enter their number. The provider initiates the charge.

```
Your Server → SentinelGate → redirect_url
                                  ↓
                    Customer lands on provider page
                    Selects network, enters phone number
                    Approves on their phone
                                  ↓
Provider confirms → SentinelGate webhook → Your Server updates order
```

Used by: **Hubtel**, **Pesapal**

---

## Provider Details

### Hubtel (Ghana)

Hubtel handles mobile money payments across all three Ghanaian networks via a hosted checkout page. The same Hubtel integration also supports card payments.

**Networks:** MTN Mobile Money, Vodafone Cash, AirtelTigo Money
**Currency:** GHS (Ghana Cedis)
**Settlement:** Next business day (T+1) to Hubtel merchant account
**Integration:** Redirect checkout

**How it works:**

1. SentinelGate creates a checkout session with Hubtel
2. Customer is redirected to `pay.hubtel.com`
3. Customer selects their mobile money provider (MTN, Vodafone, AirtelTigo)
4. Customer enters their phone number
5. A payment prompt appears on the customer's phone
6. Customer enters their mobile money PIN
7. Hubtel sends callback to SentinelGate
8. SentinelGate sends webhook to your callback URL

**API Flow:**

```bash
# Step 1: Create hosted session (same endpoint as card payments)
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "X-API-Key: sg_key_yourstore_abc123" \
  -H "X-API-Secret: sg_secret_yourstore_def456" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50.00",
    "currency": "USD",
    "order_id": "ORD-001",
    "customer_email": "buyer@example.com",
    "callback_url": "https://yoursite.com/webhook",
    "return_url": "https://yoursite.com/confirmed"
  }'

# Response includes redirect_url → redirect customer there
# They choose Card or Mobile Money on the Hubtel page
```

**Currency conversion:** If you send USD, Hubtel converts to GHS at their current exchange rate. The customer sees the GHS amount on the checkout page (e.g., $50 → ~GHS 750).

**Callback URL:** `https://sentinelgate.biz/hubtel/callback`

**Configuration (in .env):**
```bash
HUBTEL_API_ID="your_api_id"
HUBTEL_API_KEY="your_api_key"
HUBTEL_AUTH_TOKEN="your_auth_token"
HUBTEL_BASE_URL="https://api.hubtel.com"
HUBTEL_MERCHANT_ACCOUNT="2037739"
HUBTEL_CALLBACK_URL="https://sentinelgate.biz/hubtel/callback"
```

---

### BUNI / KCB M-Pesa (Kenya)

BUNI is KCB Bank's gateway for M-Pesa STK Push payments. The customer receives a push notification on their Safaricom phone and enters their M-Pesa PIN to complete payment.

**Network:** Safaricom M-Pesa only
**Currency:** KES (Kenya Shillings)
**Integration:** STK Push (server-initiated)
**Current Environment:** UAT (sandbox) — production URL pending from KCB

**Configured Stores:**

| Store | Consumer Key Env Var | Callback URL |
|-------|---------------------|-------------|
| **Castellas** | `BUNI_CASTELLAS_CONSUMER_KEY` | `https://sentinelgate.biz/buni/mpesa/callback?store=castellas` |
| **KareenHub** | `BUNI_KAREENHUB_CONSUMER_KEY` | `https://sentinelgate.biz/buni/mpesa/callback?store=kareenhub` |

**How STK Push works:**

1. Your server sends the customer's phone number and amount
2. SentinelGate authenticates with BUNI (OAuth2 token)
3. SentinelGate calls BUNI STK Push API with the phone number
4. Safaricom sends a push notification to the customer's phone:
   ```
   ┌─────────────────────────────┐
   │  M-PESA                     │
   │                             │
   │  Pay KES 1,000 to           │
   │  STORE NAME?                │
   │                             │
   │  Enter M-PESA PIN:          │
   │  [____]                     │
   │                             │
   │  [Cancel]        [OK]       │
   └─────────────────────────────┘
   ```
5. Customer enters their M-PESA PIN
6. BUNI sends callback to SentinelGate
7. SentinelGate sends webhook to your callback URL

**Processing time:** 10-30 seconds from STK push to confirmation

**Phone number format:** Must be `254XXXXXXXXX` (12 digits, starts with 254, no + sign, no spaces)

```javascript
// ❌ Wrong formats
'0712345678'         // Missing country code
'+254712345678'      // Has + sign
'254 712 345 678'    // Has spaces

// ✅ Correct format
'254712345678'

// Sanitizer function
function formatKenyanPhone(phone) {
  phone = phone.replace(/[\s\-\+]/g, '');
  if (phone.startsWith('0')) phone = '254' + phone.substring(1);
  if (!/^254\d{9}$/.test(phone)) throw new Error('Invalid format: use 254XXXXXXXXX');
  return phone;
}
```

**M-Pesa Limits:**

| Limit | Amount |
|-------|--------|
| Minimum per transaction | KES 10 |
| Maximum per transaction | KES 150,000 |
| Daily transaction limit | KES 300,000 |

**Configuration (in .env):**
```bash
BUNI_ENV=production
BUNI_TOKEN_ENDPOINT="https://accounts.buni.kcbgroup.com/oauth2/token"
BUNI_STK_PUSH_URL="https://buni.kcbgroup.com/mm/api/request/1.0.0/stkpush"
BUNI_STK_QUERY_URL="https://buni.kcbgroup.com/mm/api/request/1.0.0/stkpushquery"

# Per-store credentials
BUNI_CASTELLAS_CONSUMER_KEY="your_key"
BUNI_CASTELLAS_CONSUMER_SECRET="your_secret"
BUNI_CALLBACK_CASTELLAS="https://sentinelgate.biz/buni/mpesa/callback?store=castellas"

BUNI_KAREENHUB_CONSUMER_KEY="your_key"
BUNI_KAREENHUB_CONSUMER_SECRET="your_secret"
BUNI_CALLBACK_KAREENHUB="https://sentinelgate.biz/buni/mpesa/callback?store=kareenhub"
```

**Multi-store adapter:** The `buni-mpesa-multi.adapter.ts` handles routing to the correct store credentials based on the `store` parameter in the callback URL.

---

### Pesapal (East Africa — Pending)

Pesapal provides mobile money and card payments across Kenya, Uganda, and Tanzania. Currently blocked by invalid consumer key — awaiting new credentials.

**Networks:** M-Pesa (Kenya), Airtel Money (Kenya/Uganda), MTN MoMo (Uganda)
**Currencies:** KES, UGX, TZS
**Integration:** Redirect checkout

**When activated, flow will be:**

1. SentinelGate registers IPN (Instant Payment Notification) URL with Pesapal
2. SentinelGate creates order on Pesapal
3. Customer redirected to Pesapal checkout page
4. Customer selects mobile money provider and enters phone number
5. Payment prompt on customer's phone
6. Pesapal sends IPN to SentinelGate
7. SentinelGate sends webhook to your callback URL

**Callback URL:** `https://sentinelgate.biz/pesapal/callback`

---

## Payment Link Routing

Payment links route mobile money payments based on currency:

| Payment Link Type | Currency | Routes To |
|------------------|----------|-----------|
| MOMO + KES | KES | BUNI M-Pesa STK Push |
| MOMO + USD | USD | Pesapal (when active) / Hubtel |
| MOMO + GHS | GHS | Hubtel |

Payment link URLs:
```
Checkout:  https://sentinelgate.biz/pay/<token>
QR Code:   https://sentinelgate.biz/pay/<token>/qr.png
```

---

## Mobile Money Payment Lifecycle

### Statuses

```
pending → stk_sent → captured (success)
                   → failed (declined / timeout)
                   → cancelled (user rejected)
```

| Status | Meaning | Is Final? |
|--------|---------|-----------|
| `pending` | Payment session created | No |
| `stk_sent` | STK push sent to customer's phone (BUNI only) | No |
| `processing` | Customer approved, waiting for provider confirmation | No |
| `captured` | Payment successful | Yes |
| `failed` | Payment declined, timed out, or insufficient funds | Yes |
| `cancelled` | Customer rejected the STK push or cancelled on checkout | Yes |
| `expired` | Customer never responded to STK push (timeout) | Yes |

### Timing

| Provider | STK Push to Prompt | Customer Approval | Total |
|----------|-------------------|-------------------|-------|
| BUNI M-Pesa | 2-5 seconds | Up to 60 seconds | 10-65 seconds |
| Hubtel MoMo | N/A (redirect) | Up to 120 seconds | 30-120 seconds |
| Pesapal | N/A (redirect) | Up to 120 seconds | 30-120 seconds |

If the customer doesn't respond within the timeout, the payment expires automatically.

---

## Failure Reasons

### Common Mobile Money Failures

| Failure | Cause | Customer Action |
|---------|-------|----------------|
| `insufficient_balance` | Not enough money in wallet | Top up mobile money balance |
| `wrong_pin` | Customer entered incorrect PIN | Try again with correct PIN |
| `request_cancelled` | Customer rejected the prompt | Try again and approve |
| `timeout` | Customer didn't respond in time | Initiate a new payment |
| `daily_limit_exceeded` | Hit mobile money daily limit | Try again tomorrow |
| `account_inactive` | Mobile money wallet not active | Activate at agent or via USSD |
| `network_error` | Telecom network issue | Wait a few minutes and retry |
| `invalid_phone` | Phone number not registered for mobile money | Verify number is correct |

### M-Pesa Specific Failures

| M-Pesa Code | Meaning |
|-------------|---------|
| `1` | Insufficient funds |
| `1032` | Request cancelled by user |
| `1037` | Request timed out |
| `2001` | Wrong PIN entered |
| `1001` | Unable to lock subscriber — concurrent transaction |

---

## Integration Guide

### For Hosted Checkout (Hubtel — Simplest)

No special mobile money handling needed. The customer chooses Card or Mobile Money on the Hubtel checkout page.

```javascript
// Same code as card payments — the provider handles the UI
const session = await axios.post(
  'https://sentinelgate.biz/v1/hosted/create',
  {
    amount: '50.00',
    currency: 'USD',
    order_id: 'ORD-001',
    customer_email: 'buyer@example.com',
    callback_url: 'https://yoursite.com/webhook',
    return_url: 'https://yoursite.com/confirmed'
  },
  {
    headers: {
      'X-API-Key': process.env.SENTINELGATE_API_KEY,
      'X-API-Secret': process.env.SENTINELGATE_API_SECRET,
      'Content-Type': 'application/json'
    }
  }
);

// Redirect customer — they choose payment method on the Hubtel page
res.redirect(session.data.redirect_url);
```

### For STK Push (BUNI M-Pesa — Direct)

If you want to trigger an M-Pesa STK push directly (e.g., for in-app payments where you already have the phone number), you can use the direct charge endpoint:

```javascript
const result = await axios.post(
  'https://sentinelgate.biz/v1/charge',
  {
    amount: '1000.00',
    currency: 'KES',
    order_id: 'ORD-002',
    customer_email: 'buyer@example.com',
    phone: '254712345678',    // Required for STK push
    provider: 'BUNI',
    callback_url: 'https://yoursite.com/webhook'
  },
  {
    headers: {
      'X-API-Key': process.env.SENTINELGATE_API_KEY,
      'X-API-Secret': process.env.SENTINELGATE_API_SECRET,
      'Content-Type': 'application/json'
    }
  }
);

// result.data.status will be 'stk_sent' initially
// Wait for webhook callback with final status
```

### Webhook Handling (Same for All Providers)

```javascript
app.post('/webhook',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    // Verify signature
    const signature = req.headers['x-sentinel-signature'];
    if (!verifySignature(req.body, signature, process.env.WEBHOOK_SECRET)) {
      return res.status(401).send('Invalid signature');
    }

    res.status(200).send('OK');

    const event = JSON.parse(req.body);

    // event.channel will be 'mobile_money' or 'momo'
    // event.provider will be 'hubtel' or 'buni'
    if (event.status === 'captured') {
      markOrderAsPaid(event.wc_order_id, event.sentinel_transaction_id);
    }
  }
);
```

---

## Refunds

### M-Pesa Refunds

M-Pesa refunds are processed through the provider's reversal API. Refund timelines:

| Provider | Refund Method | Timeline |
|----------|-------------|----------|
| BUNI M-Pesa | Automated reversal | 1-2 business days |
| Hubtel Mobile Money | Dashboard or API | 1-3 business days |
| Pesapal | Dashboard or API | 1-3 business days |

**Note:** Mobile money refunds go back to the same phone number/wallet that made the original payment. Unlike card refunds, they are usually faster.

---

## Troubleshooting

### Customer didn't receive STK push (BUNI)

1. **Verify phone format:** Must be `254XXXXXXXXX`
2. **Confirm Safaricom network:** STK push only works on Safaricom. Airtel/Telkom numbers won't receive it
3. **Check if phone has M-Pesa:** Customer must have an active M-Pesa account
4. **Phone must be on:** Turned on, with mobile signal (not airplane mode)
5. **Check BUNI environment:** Currently on UAT — production URL pending from KCB
6. **Concurrent transactions:** M-Pesa only allows one STK push at a time per phone number. If one is pending, the next will fail with error `1001`

### Customer approved but order not updated

1. Check PM2 logs for callback errors:
   ```bash
   pm2 logs sentinel-svc-rails | grep "buni\|hubtel\|callback"
   ```
2. Verify the callback URL is reachable:
   ```bash
   curl -s https://sentinelgate.biz/buni/mpesa/callback?store=castellas
   ```
3. Query the transaction status:
   ```bash
   curl https://sentinelgate.biz/v1/transaction/sg_txn_xxx \
     -H "X-API-Key: your_key" \
     -H "X-API-Secret: your_secret"
   ```

### Hubtel mobile money timeout

Hubtel mobile money prompts expire after 2 minutes. If the customer doesn't approve in time:
- The transaction is automatically marked as failed/expired
- The customer needs to start a new payment from your store
- No charge is made

### Wrong amount shown on Hubtel (currency conversion)

If your store uses USD and the customer sees GHS on the Hubtel page, this is expected. Hubtel operates in GHS and converts at their rate. You cannot override this — it is a provider-level setting.

---

## Security Considerations

### Phone Number Privacy

- Never log full phone numbers in production — mask as `2547****5678`
- Store phone numbers encrypted if you persist them
- Don't expose phone numbers in URLs or query strings

### Transaction Verification

- Always verify webhook signatures before processing
- Use `sentinel_transaction_id` for idempotency — don't process the same payment twice
- For high-value transactions, query the transaction status as a secondary check:
  ```
  GET /v1/transaction/:txnId
  ```

### STK Push Security

- STK push requires the customer to enter their mobile money PIN on their own phone
- The PIN is never transmitted to or processed by SentinelGate
- The push can only be sent to the phone number provided — it cannot be redirected

---

## Related Documentation

- [API Reference](./API_REFERENCE.md) — Full endpoint documentation
- [Cards Payment Documentation](./CARDS_PAYMENT_METHOD_DOCUMENTATION.md) — Card processing guide
- [Developer Integration Guide](./DEVELOPER_INTEGRATION_GUIDE.md) — Step-by-step integration
- [Common Issues](./COMMON_ISSUES.md) — Troubleshooting guide
- [Merchant Guide](./MERCHANT_GUIDE.md) — Non-technical overview

---

*© 2026 SentinelGate. All rights reserved.*
