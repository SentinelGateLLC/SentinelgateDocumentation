# SentinelGate API - Quick Reference

## Base URL
```
http://185.229.224.244:3000
```

## Authentication Headers
```
X-API-Key: sk_live_...
X-API-Secret: secret_...
```

## Key Endpoints

### Create Payment
```bash
POST /api/payments/create
{
  "amount_cents": 10000,
  "currency": "KES",
  "provider": "MPESA_SAFARICOM",
  "metadata": {"phone_number": "254712345678"}
}
```

### Get Payment
```bash
GET /api/payments/{payment_id}
```

### List Payments
```bash
GET /api/payments?status=SUCCESS&limit=10
```

### Create Payout
```bash
POST /api/payouts/create
{
  "amount_cents": 5000,
  "currency": "KES",
  "provider": "MPESA_SAFARICOM",
  "destination": "254712345678"
}
```

## Providers

- `MPESA_SAFARICOM` - M-Pesa (Kenya)
- `BUNI` - Cards & MPESA (Kenya, Nigeria)
- `KAREENHUB` - Bank Transfer (Kenya)
- `STRIPE` - Cards (Global)

## Payment Statuses

- `PENDING` - Awaiting completion
- `SUCCESS` - Completed
- `FAILED` - Failed
- `CANCELLED` - Cancelled
- `EXPIRED` - Expired

## Webhook Events

- `payment.pending`
- `payment.completed`
- `payment.failed`
- `payout.completed`
