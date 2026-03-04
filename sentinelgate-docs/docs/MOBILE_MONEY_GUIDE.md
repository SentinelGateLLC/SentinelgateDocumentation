# Mobile Money Guide

How to accept mobile money payments through SentinelGate.

---

## Supported Networks

| Country | Networks | Currency |
|---------|----------|----------|
| Kenya | M-Pesa | KES |
| Ghana | MTN MoMo, AirtelTigo, Telecel | GHS |
| Uganda | MTN MoMo | UGX |
| Tanzania | MTN MoMo | TZS |

---

## How It Works

### STK Push Flow (Kenya)

1. Customer selects "Mobile Money" and enters phone number
2. SentinelGate sends an STK push to the customer's phone
3. Customer sees a payment prompt on their phone
4. Customer enters their M-Pesa PIN
5. Payment is confirmed via callback
6. Order is updated automatically

### USSD/Redirect Flow (Ghana)

1. Customer selects "Mobile Money" and enters phone number
2. Customer receives a USSD prompt or approval request
3. Customer confirms on their phone
4. Payment is confirmed via callback

---

## Integration

### Via Hosted Checkout

Specify `currency: "KES"` or `currency: "GHS"` — the checkout page will automatically show mobile money as an option.

```bash
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "x-api-key: sk_live_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "500.00",
    "currency": "KES",
    "order_id": "ORD-001",
    "customer_email": "buyer@example.com",
    "callback_url": "https://yoursite.com/webhooks/payment"
  }'
```

### Via Direct Charge

```bash
curl -X POST https://sentinelgate.biz/v1/charge \
  -H "x-api-key: sk_live_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_cents": 50000,
    "currency": "KES",
    "merchant_id": "your-merchant-id",
    "rail": "MOMO",
    "email": "buyer@example.com",
    "metadata": {
      "phone": "254712345678"
    }
  }'
```

---

## Phone Number Format

| Country | Format | Example |
|---------|--------|---------|
| Kenya | 254XXXXXXXXX | 254712345678 |
| Ghana | 233XXXXXXXXX | 233241234567 |
| Uganda | 256XXXXXXXXX | 256771234567 |

Always use the international format without the leading `+`.

---

## Transaction Timing

| Step | Typical Duration |
|------|-----------------|
| STK push delivery | 1–5 seconds |
| Customer enters PIN | 5–60 seconds |
| Confirmation callback | 1–10 seconds |
| Total | Under 2 minutes |

If the customer does not respond within 60 seconds, the transaction times out and status is set to `FAILED`.

---

## Callback Handling

Mobile money callbacks follow the same format as card payments:

```json
{
  "event": "payment.captured",
  "transaction_id": "sg_txn_abc123",
  "status": "CAPTURED",
  "amount_cents": 50000,
  "currency": "KES"
}
```

See [Webhook Guide](WEBHOOK_GUIDE.md) for verification details.

---

## Common Issues

| Issue | Cause | Solution |
|-------|-------|---------|
| STK not received | Wrong phone format | Use international format (254...) |
| Transaction timeout | Customer didn't respond | Customer must retry |
| "Service unavailable" | Temporary network issue | Retry after a few minutes |
| Wrong amount | Amount in major vs minor units | Use cents (50000 = KES 500) |

---

## Best Practices

- Always validate phone number format before sending
- Show the customer a "waiting for confirmation" screen
- Implement polling as a fallback if callbacks are delayed
- Set appropriate timeout messages for the customer
- Display the exact amount the customer will see on their phone

---

© 2026 SentinelGate — Whyte AG Group
