# Shopify Integration

Integrate SentinelGate payments into your Shopify store using webhooks and redirect checkout.

---

## Prerequisites

- Shopify store with a paid plan
- Shopify Custom App with order permissions
- SentinelGate merchant credentials
- SSL certificate on your domain

---

## Architecture

```
Customer → Shopify Checkout → Order Created
                                    ↓
                            SentinelGate Webhook
                                    ↓
                            Payment Processing
                                    ↓
                         Order Marked as Paid
```

SentinelGate listens for Shopify `orders/create` webhooks, processes payments, and updates order status via the Shopify Admin API.

---

## Step 1: Create a Custom App

1. Go to **Shopify Admin → Settings → Apps and sales channels → Develop apps**
2. Click **Create an app**
3. Name it "SentinelGate Payments"
4. Under **API Scopes**, grant:
   - `read_orders`, `write_orders`
   - `read_products`
   - `read_customers`
5. Install the app and note the **Admin API access token**

---

## Step 2: Register Your Store

Contact SentinelGate support or use the tenant registration API:

```bash
curl -X POST https://sentinelgate.biz/v1/tenants \
  -H "x-api-key: your_admin_key" \
  -H "Content-Type: application/json" \
  -d '{
    "shop_domain": "your-store.myshopify.com",
    "shopify_access_token": "shpat_xxxxx",
    "merchant_id": "your-merchant-id"
  }'
```

---

## Step 3: Configure Webhooks

In Shopify Admin → Settings → Notifications → Webhooks:

| Event | URL |
|-------|-----|
| Order creation | `https://sentinelgate.biz/webhooks/shopify/order-created` |

Set the format to **JSON**.

SentinelGate verifies all incoming webhooks using HMAC-SHA256 signature validation.

---

## Step 4: Add Redirect Script

Add this script to your checkout to redirect customers to SentinelGate for payment:

```html
<script>
  if (window.Shopify && Shopify.checkout) {
    fetch('https://sentinelgate.biz/v1/hosted/create', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        order_id: Shopify.checkout.order_id,
        amount: Shopify.checkout.payment_due,
        currency: Shopify.checkout.currency,
        customer_email: Shopify.checkout.email,
        success_url: window.location.href,
      })
    })
    .then(r => r.json())
    .then(d => { if (d.redirect_url) window.location = d.redirect_url; });
  }
</script>
```

---

## Step 5: Test

1. Place a test order on your Shopify store
2. Verify the webhook fires to SentinelGate
3. Complete the payment
4. Check that the Shopify order is marked as paid

---

## Webhook Security

All Shopify webhooks are verified using:
- **HMAC-SHA256** signature in the `X-Shopify-Hmac-Sha256` header
- Request body is hashed against your app's API secret
- Invalid signatures are rejected with 401

SentinelGate also implements:
- **Idempotency** — Duplicate webhooks are safely ignored via Redis-backed idempotency keys
- **Rate limiting** — Prevents webhook replay attacks
- **BullMQ queue** — Webhooks are processed asynchronously for reliability

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| Webhook not firing | Check Shopify Notifications → Webhooks for delivery failures |
| Order not updating | Verify the Shopify access token has write_orders permission |
| HMAC validation failing | Ensure the webhook secret matches your Shopify app secret |
| Duplicate charges | SentinelGate uses idempotency keys — this should not happen |

---

© 2026 SentinelGate — Whyte AG Group
