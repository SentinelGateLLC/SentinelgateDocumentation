# Card Payment Method Documentation

**For:** Developers & Technical Teams
**Last Updated:** February 24, 2026

---

## Overview

SentinelGate processes card payments (Visa and Mastercard) through multiple provider adapters. The payment routing engine selects the appropriate provider based on merchant configuration, currency, and provider availability.

This document covers how card payments work end-to-end, which providers handle cards, and what to expect during integration.

---

## Integration Modes

### Mode 1: Hosted Checkout (Recommended)

The customer is redirected to a payment page hosted by SentinelGate or the provider. Card details are entered on their page — never on yours.

**Endpoint:** `POST /v1/hosted/create`

**Flow:**
```
Your server → POST /v1/hosted/create → get redirect_url
Customer → redirect to payment page → enter card → pay
Provider → processes card → callback to SentinelGate
SentinelGate → webhook to your callback_url
Customer → redirected to your return_url
```

**PCI Requirement:** None. Card data never touches your server.

**Example:**
```bash
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
    "return_url": "https://yoursite.com/order-confirmed"
  }'
```

### Mode 2: Direct Charge (PCI Required)

Card details are collected on your checkout page and sent directly to SentinelGate for processing. **Requires PCI DSS Level 1 compliance.** Most merchants should not use this.

**Endpoint:** `POST /v1/charge`

**Flow:**
```
Customer → enters card on YOUR page
Your server → POST /v1/charge with card details
SentinelGate → charges via provider
Response → captured / failed / 3DS_REDIRECT
```

**PCI Requirement:** PCI DSS Level 1 (full audit, SAQ D).

**Example:**
```bash
curl -X POST https://sentinelgate.biz/v1/charge \
  -H "X-API-Key: sg_key_yourstore_abc123" \
  -H "X-API-Secret: sg_secret_yourstore_def456" \
  -H "Content-Type: application/json" \
  -d '{
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
  }'
```

### Mode 3: iFrame Token (Partial PCI)

SentinelGate provides an iframe that renders a card form on your page. Card details are tokenized inside the iframe and the token is sent to your server. Requires PCI DSS SAQ A-EP.

This mode is available through the WooCommerce plugin (Integration Mode: "iFrame Token").

---

## Card Payment Providers

### Currently Active

| Provider | Status | Currencies | Features |
|----------|--------|-----------|----------|
| **Hubtel** | ✅ Live | GHS (USD converted) | Hosted checkout, cards + mobile money combined |

### Pending Activation

| Provider | Status | Currencies | Blocker |
|----------|--------|-----------|---------|
| **Paystack** | ⏳ Read-only | USD, GHS, NGN | Business verification incomplete on dashboard.paystack.co |
| **Emergent** | ⏳ Server error | USD, multi-currency | IIS configuration error on api.interpayafrica.com |
| **Pesapal** | ⏳ Invalid key | USD, KES, UGX | Consumer key/secret rejected by API |
| **Korapay** | ⏳ Wrong credentials | USD, NGN, GHS | Provided keys are actually Paystack keys |

### Available in Code (Not Yet Configured)

These adapters exist in the codebase and can be activated when credentials are available:

| Provider | Adapter Path | Capabilities |
|----------|-------------|-------------|
| Stripe | `src/providers/cards/stripe/stripe.adapter.ts` | Global card processing |
| Adyen | `src/providers/cards/adyen/adyen.adapter.ts` | Global card processing |
| Checkout.com | `src/providers/cards/checkout/checkout.adapter.ts` | Global card processing |
| Finix | `src/providers/cards/finix/finix.adapter.ts` | US card processing |
| Airwallex | `src/providers/cards/airwallex/airwallex.adapter.ts` | APAC card processing |

---

## 3D Secure (3DS)

3D Secure is an additional authentication step required by many banks. The customer receives a one-time password (OTP) on their phone and enters it on the payment page.

### Hosted Checkout

3D Secure is handled automatically. The payment page renders the 3DS challenge, the customer enters their OTP, and payment completes. No action needed from you.

### Direct Charge

If the provider requires 3DS, the `/v1/charge` response will return:

```json
{
  "status": "3DS_REDIRECT",
  "redirect_url": "https://provider.com/3ds/challenge?ref=abc123",
  "sentinel_transaction_id": "sg_txn_..."
}
```

You must redirect the customer to `redirect_url`. After 3DS completes, they are returned to your return URL and the webhook fires with the final status.

---

## Currency Handling

### How Currency Conversion Works

If your store prices in USD but the card provider operates in a local currency (e.g., Hubtel in GHS):

1. You send `amount: "50.00"` and `currency: "USD"` to SentinelGate
2. SentinelGate forwards to Hubtel
3. Hubtel converts USD → GHS at their current exchange rate
4. The customer sees GHS amount on the payment page (e.g., GHS 750)
5. Hubtel settles in GHS to the merchant account

**You don't control the exchange rate.** It is set by the provider.

### Supported Currency Combinations

| Your Store Currency | Provider | Customer Pays In |
|-------------------|----------|-----------------|
| USD | Hubtel | GHS (converted) |
| GHS | Hubtel | GHS (no conversion) |
| USD | Paystack (when active) | USD or NGN |
| USD | Emergent (when active) | USD (no conversion) |
| KES | Pesapal (when active) | KES |

---

## Card Payment Lifecycle

### Statuses

```
pending → processing → captured (success)
                     → failed (declined)
                     → 3DS_REDIRECT (needs customer action)
```

| Status | Meaning | Is Final? |
|--------|---------|-----------|
| `pending` | Session created, waiting for customer | No |
| `processing` | Card submitted, waiting for provider | No |
| `captured` | Payment successful | Yes |
| `failed` | Card declined or error | Yes |
| `3DS_REDIRECT` | Awaiting 3D Secure verification | No |
| `refunded` | Payment reversed | Yes |
| `expired` | Session timed out (customer never paid) | Yes |

### Decline Reasons

| Decline Code | Meaning | Customer Action |
|-------------|---------|----------------|
| `insufficient_funds` | Not enough money on card | Use a different card |
| `card_declined` | Generic decline from issuing bank | Contact their bank |
| `expired_card` | Card past expiry date | Use a valid card |
| `incorrect_cvv` | Wrong CVV entered | Re-enter carefully |
| `3ds_failed` | 3D Secure OTP was wrong or timed out | Try again, enter correct OTP |
| `do_not_honor` | Bank refuses without specific reason | Contact their bank |
| `card_not_supported` | Card type not accepted | Use Visa or Mastercard |
| `online_payments_disabled` | Card not enabled for e-commerce | Contact bank to enable |

---

## Card Form (Hosted Checkout Page)

The hosted checkout page at `/v1/hosted/pay/:sessionId` renders a card form with:

- Card number input (with Luhn validation)
- Expiry date (MM/YY)
- CVV (3 digits)
- Pay button with amount and currency
- SentinelGate branding

The form submits to `/v1/hosted/pay/:sessionId/process`, which routes to the configured provider. If the provider requires a redirect (e.g., Hubtel checkout), the customer is redirected. If processing is inline, the result is shown immediately.

**Template:** `src/routes/checkout-template.html`

---

## Refunds

### Processing a Card Refund

```bash
curl -X POST https://sentinelgate.biz/v1/refund \
  -H "X-API-Key: sg_key_yourstore_abc123" \
  -H "X-API-Secret: sg_secret_yourstore_def456" \
  -H "Content-Type: application/json" \
  -d '{
    "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
    "amount": 50.00,
    "reason": "Customer requested refund"
  }'
```

**Partial refunds:** Send an `amount` less than the original transaction.

**Full refunds:** Send the full original amount or omit the `amount` field.

**Timeline:** Refunds typically take 5-10 business days to appear on the customer's card statement. This is controlled by the issuing bank, not SentinelGate.

---

## Security

### Data Handling

- **Hosted Checkout:** Card data never touches your server. Zero PCI scope.
- **Direct Charge:** Card data passes through your server. Full PCI DSS required.
- **At Rest:** SentinelGate does not store full card numbers. Only last 4 digits and card brand are retained for reference.
- **In Transit:** All communication over HTTPS/TLS 1.2+.

### Fraud Prevention

Card payments include built-in protections from providers:

- 3D Secure verification (OTP)
- AVS (Address Verification System) where supported
- CVV verification
- Velocity checks (rapid transaction detection)
- BIN checks (card issuer validation)

### PCI Compliance Summary

| Integration Mode | PCI Requirement | Who Handles Card Data |
|-----------------|----------------|-----------------------|
| Hosted Checkout (Redirect) | None | Provider |
| iFrame Token | SAQ A-EP | Provider (via iframe) |
| Direct Charge | Level 1 (SAQ D) | You + SentinelGate |

---

## Troubleshooting

### Card payment returns "PROVIDER_ERROR"

The underlying provider had an issue. Check the `message` field in the error response. Common causes:
- Provider API is down (check https://sentinelgate.biz/health)
- Merchant account not fully activated on provider
- Currency mismatch

### Customer sees wrong currency amount

This is expected when your store currency differs from the provider's settlement currency. See [Currency Handling](#currency-handling) above.

### 3D Secure keeps failing

- Customer may be entering wrong OTP
- Customer's bank may have SMS delivery issues
- Some corporate/prepaid cards don't support 3DS
- Solution: Ask customer to contact their bank or try a different card

### Payment stuck in "pending"

The customer started but didn't complete payment. The session will eventually expire. You can:
1. Query the transaction: `GET /v1/transaction/:txnId`
2. If still pending after 30 minutes, treat as abandoned
3. Send a reminder email with a new payment link

---

## Related Documentation

- [API Reference](./API_REFERENCE.md) — Full endpoint documentation
- [Developer Integration Guide](./DEVELOPER_INTEGRATION_GUIDE.md) — Step-by-step integration
- [Common Issues](./COMMON_ISSUES.md) — Troubleshooting guide
- [Merchant Guide](./MERCHANT_GUIDE.md) — Non-technical overview

---

*© 2026 SentinelGate. All rights reserved.*
