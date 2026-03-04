# Card Processing Guide

How SentinelGate handles credit and debit card payments.

---

## How Card Payments Work

```
Customer → Selects "Pay with Card" → Redirected to Secure Form
→ Enters Card Details → 3D Secure Check → Payment Processed
→ Callback to SentinelGate → Order Updated → Customer Redirected
```

SentinelGate uses a redirect-based card payment flow. Card details are entered on a PCI-compliant hosted form — your server never sees or stores card data.

---

## Supported Cards

| Network | Supported |
|---------|-----------|
| Visa | Yes |
| Mastercard | Yes |
| Verve | Yes (select regions) |

---

## 3D Secure

All card transactions go through 3D Secure (3DS) authentication when supported by the issuing bank. This adds a verification step where the cardholder confirms the transaction via their bank's authentication system.

3DS provides:
- **Fraud protection** — Verifies cardholder identity
- **Liability shift** — Fraud liability moves to the card issuer
- **Higher approval rates** — Banks approve verified transactions more readily

---

## Integration

### Hosted Checkout (Recommended)

No code changes needed. SentinelGate's hosted checkout automatically handles the card form, 3DS, and redirect flow.

```bash
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "x-api-key: sk_live_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "25.00",
    "currency": "USD",
    "order_id": "ORD-001",
    "customer_email": "buyer@example.com",
    "callback_url": "https://yoursite.com/webhooks/payment",
    "success_url": "https://yoursite.com/thank-you"
  }'
```

Redirect the customer to the `redirect_url` in the response.

### Payment Links

Create a payment link and share it. Customers click, enter card details, and pay.

### Direct Charge

```bash
curl -X POST https://sentinelgate.biz/v1/charge \
  -H "x-api-key: sk_live_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_cents": 2500,
    "currency": "USD",
    "merchant_id": "your-merchant-id",
    "rail": "CARD",
    "email": "buyer@example.com"
  }'
```

---

## PCI Compliance

Because SentinelGate uses redirect-based card entry, your server never handles card data. This means:

- **No PCI DSS certification required** for your business
- Card numbers are entered on a secure, PCI-compliant hosted form
- SentinelGate and the underlying payment processors handle all card data security
- Your server only receives transaction IDs and status updates

---

## Test Cards

Use these cards in test/sandbox mode:

| Card Number | Result |
|------------|--------|
| 4111 1111 1111 1111 | Successful payment |
| 4000 0000 0000 0002 | Declined |
| 4000 0000 0000 3220 | 3D Secure required |

- Expiry: Any future date
- CVV: Any 3 digits

---

## Common Decline Reasons

| Reason | Description |
|--------|-------------|
| Insufficient Funds | Card does not have enough balance |
| Card Declined | Issuing bank declined the transaction |
| Invalid Card | Card number is incorrect |
| Expired Card | Card has passed its expiry date |
| 3DS Failed | Customer failed 3D Secure verification |
| Fraud Suspected | Transaction flagged by fraud detection |

---

## Best Practices

- Always use HTTPS for callback and redirect URLs
- Display clear payment amounts and descriptions to customers
- Handle both success and failure callbacks
- Implement webhook verification for all payment notifications
- Use idempotency to prevent duplicate charges

---

© 2026 SentinelGate — Whyte AG Group
