# SentinelGate — Shopify Integration

**For developers and store administrators integrating SentinelGate with Shopify**

---

## Overview

SentinelGate integrates with Shopify via a server-side middleware that captures order webhooks, processes payments, and updates order status through the Shopify Admin API. Customers are redirected to a secure hosted checkout page or receive a payment link by email.

---

## Requirements

| Requirement | Details |
|------------|---------|
| Shopify store | Any plan |
| Shopify Custom App | With `read_orders` and `write_orders` scopes |
| SentinelGate merchant account | Tenant ID and credentials |
| Domain with HTTPS | For receiving webhooks |

---

## Architecture

```
┌──────────────┐    Webhook     ┌──────────────────────────────┐
│              │───────────────▶│  SentinelGate Middleware      │
│   Shopify    │                │                              │
│   Store      │◀───────────────│  • HMAC signature verify     │
│              │  Admin API     │  • Redis idempotency check   │
└──────┬───────┘  (mark paid)   │  • BullMQ async queue        │
       │                        │  • Payment session creation  │
       │                        │  • Email with payment link   │
       │                        └──────────────┬───────────────┘
       │                                       │
       │         ┌─────────────────────────────┘
       │         │ Payment redirect
       │         ▼
       │    ┌──────────────┐    Charge    ┌─────────────┐
       │    │   Checkout   │─────────────▶│  Provider   │
       │    │   Page       │◀─────────────│  (Hubtel,   │
       │    └──────┬───────┘   Result     │   etc.)     │
       │           │                      └─────────────┘
       │           │ Return URL
       │           ▼
       │    ┌──────────────┐
       └───▶│  Thank You   │
            │  Page        │
            └──────────────┘
```

---

## Setup Instructions

### Step 1 — Create a Shopify Custom App

1. In Shopify Admin, go to **Settings → Apps and sales channels**
2. Click **Develop apps** → **Create an app**
3. Name it `SentinelGate Payments`
4. Click **Configure Admin API scopes** and enable:
   - `read_orders` — Read order details
   - `write_orders` — Mark orders as paid
   - `read_products` — Access product information (optional)
5. Click **Save** then **Install app**
6. Copy the **Admin API access token** (shown only once — save it securely)
7. Go to **Settings → Notifications → Webhooks** (scroll to bottom)
8. Copy the **HMAC secret** displayed there

### Step 2 — Register Your Store with SentinelGate

Provide the following to your SentinelGate integration team:

| Information | Where To Find It |
|------------|------------------|
| **Shopify domain** | `your-store.myshopify.com` (Settings → Domains) |
| **Store public URL** | `https://yourstore.com` |
| **Admin API access token** | From Step 1 (starts with `shpat_`) |
| **Webhook HMAC secret** | From Step 1 (Settings → Notifications → Webhooks) |
| **Contact email** | Your business email |
| **Preferred currency** | e.g., USD |

Your SentinelGate team will register your store as a tenant and provide a **Tenant ID**.

### Step 3 — Configure Webhooks in Shopify

In Shopify Admin → **Settings → Notifications → Webhooks**, click **Create webhook** for each:

| Event | Webhook URL | Format |
|-------|-------------|--------|
| Order creation | `https://sentinelgate.biz/webhooks/shopify/order-created` | JSON |
| Order payment | `https://sentinelgate.biz/shopify/webhook/orders/paid` | JSON |
| Order cancellation | `https://sentinelgate.biz/shopify/webhook/orders/cancelled` | JSON |
| Refund creation | `https://sentinelgate.biz/shopify/webhook/refunds/create` | JSON |

After creating each webhook, Shopify will send a test payload. Check with your SentinelGate team that the test was received.

### Step 4 — Set Up Payment Method

1. Go to **Settings → Payments**
2. Under **Manual payment methods**, click **Create custom payment method**
3. Configure:

| Setting | Value |
|---------|-------|
| Custom payment method name | `Pay with Card / Mobile Money` |
| Additional details | `You will be redirected to complete your payment securely` |
| Payment instructions | `Complete your payment using the secure link. Your order will be confirmed once payment is received.` |

4. Click **Activate**

### Step 5 — Add Checkout Redirect Script (Optional)

If you want customers to be redirected to pay immediately after checkout (instead of waiting for an email), add this script:

1. Go to **Settings → Checkout → Order status page → Additional scripts**
2. Paste:

```html
<script>
  if (Shopify.checkout && Shopify.checkout.order_id) {
    var amount = Math.round(Shopify.checkout.total_price * 100);
    var email = Shopify.checkout.email || '';
    var name = '';
    if (Shopify.checkout.billing_address) {
      name = Shopify.checkout.billing_address.first_name + ' ' 
           + Shopify.checkout.billing_address.last_name;
    }
    var orderId = Shopify.checkout.order_id;
    
    window.location.href = 'https://sentinelgate.biz/pay/shopify-redirect'
      + '?amount=' + amount
      + '&email=' + encodeURIComponent(email)
      + '&order=' + orderId
      + '&name=' + encodeURIComponent(name);
  }
</script>
```

3. Click **Save**

This redirects customers to the SentinelGate payment page immediately after they complete Shopify checkout.

---

## Payment Flow

### Flow A: Redirect at Checkout (with script from Step 5)

```
Customer completes Shopify checkout
        ↓
Redirect script sends customer to /pay/shopify-redirect
        ↓
SentinelGate creates checkout session with provider
        ↓
Customer redirected to payment page
        ↓
Customer pays → redirected to thank-you page
        ↓
Provider callback → SentinelGate marks order as paid via Shopify Admin API
```

### Flow B: Payment Link by Email (without script)

```
Customer completes Shopify checkout
        ↓
Shopify sends order webhook to SentinelGate
        ↓
SentinelGate BullMQ worker processes order
        ↓
Payment link emailed to customer
        ↓
Customer clicks link → pays on hosted checkout
        ↓
Provider callback → SentinelGate marks order as paid via Shopify Admin API
```

---

## Webhook Endpoints

### Order Webhooks (Shopify → SentinelGate)

| Endpoint | Event | Purpose |
|----------|-------|---------|
| `POST /webhooks/shopify/order-created` | Order creation | Primary — creates payment session |
| `POST /shopify/webhook/orders/create` | Order creation | Alternate URL (same handler) |
| `POST /shopify/webhook/orders/paid` | Order marked paid | Logs payment confirmation |
| `POST /shopify/webhook/orders/cancelled` | Order cancelled | Logs cancellation |
| `POST /shopify/webhook/refunds/create` | Refund created | Processes refund |

### Customer-Facing Pages

| Endpoint | Purpose |
|----------|---------|
| `GET /pay/shopify-redirect` | Redirects customer to payment provider |
| `GET /pay/shopify-thankyou` | Post-payment confirmation page |

### Admin Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/tenants` | POST | Register a new store tenant |
| `/v1/tenants/:id/logs` | GET | View transaction logs for a tenant |

---

## Security

### Webhook Verification

Every webhook from Shopify is verified using HMAC-SHA256:

```
Expected: HMAC-SHA256(raw_body, webhook_hmac_secret)
Received: X-Shopify-Hmac-Sha256 header (base64)
```

Requests with invalid or missing signatures are rejected with HTTP 401.

### Idempotency

Redis-based idempotency keys prevent duplicate processing. If Shopify sends the same webhook twice (common during retries), the second request is safely ignored.

### Encryption

Sensitive tenant data (API tokens, secrets) is encrypted at rest using AES-256-GCM before storage in the database.

### Rate Limiting

Webhook endpoints are rate-limited at 100 requests per minute per store IP to prevent abuse.

---

## Queue Processing (Technical Detail)

Webhooks are processed asynchronously via BullMQ to handle traffic spikes:

| Setting | Value |
|---------|-------|
| **Queue backend** | Redis (localhost:6379) |
| **Concurrency** | 10 workers |
| **Max attempts** | 5 per job |
| **Retry strategy** | Exponential backoff |
| **Job timeout** | 30 seconds |

Failed jobs are retried automatically. After 5 failures, the job is moved to a dead-letter queue for manual review.

---

## Troubleshooting

### Webhooks not being received

1. Verify webhook URLs are correct in Shopify Admin → Notifications → Webhooks
2. Check if `https://sentinelgate.biz` is reachable from the internet
3. In Shopify, check the webhook delivery log — failed deliveries show error codes
4. Verify the HMAC secret matches what SentinelGate has on file

### Customer not redirected after checkout

1. Verify the additional checkout script is installed (Step 5)
2. Check browser developer console for JavaScript errors
3. Ensure the script runs on the order status page, not the checkout page
4. Test with a fresh browser / incognito window

### Order not marked as paid after payment

1. Verify the Shopify Admin API token has `write_orders` scope
2. Check if the token has expired or been revoked
3. Check SentinelGate server logs for Shopify API errors
4. Verify the tenant configuration has the correct token

### Payment email not received

1. Check spam/junk folders
2. Verify the customer email address is correct in the Shopify order
3. Check SentinelGate email delivery logs
4. Ensure the email sending domain is verified (Resend.com configuration)

### Duplicate charges

This should not happen due to idempotency. If it does:

1. Check Redis connectivity — idempotency requires Redis
2. Verify the order webhook isn't being sent from multiple Shopify webhook configs
3. Contact SentinelGate support with both transaction IDs

---

## Testing

### Test Checklist

- [ ] Webhooks configured in Shopify (all 4 events)
- [ ] Custom payment method created and activated
- [ ] Redirect script installed (if using Flow A)
- [ ] Place a test order using the custom payment method
- [ ] Verify redirect to payment page (Flow A) or email received (Flow B)
- [ ] Complete payment on the hosted checkout page
- [ ] Verify order is marked as paid in Shopify
- [ ] Verify customer sees thank-you page
- [ ] Check SentinelGate logs for any errors

### Sending a Test Webhook

You can manually test the webhook from Shopify:

1. Go to **Settings → Notifications → Webhooks**
2. Click **Send test notification** next to the Order creation webhook
3. Check SentinelGate logs to confirm it was received and processed

---

## Updating Your Integration

If your SentinelGate team deploys updates to the middleware:

- **No action required from you** — webhook endpoints remain the same
- **If your Shopify token expires** — generate a new one (Settings → Apps → Your App → API Credentials) and share it with SentinelGate
- **If you change your Shopify domain** — notify SentinelGate to update the tenant config
