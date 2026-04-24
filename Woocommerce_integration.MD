# SentinelGate — WooCommerce Integration Guide

## Quick Start

### 1. Install the Plugin

Upload `sentinelgate-psp.zip` to your WordPress site:

```
WordPress Admin → Plugins → Add New → Upload Plugin → Choose sentinelgate-psp.zip
```

Activate the plugin after installation.

### 2. Configure

Navigate to **WooCommerce → Settings → Payments → SentinelGate PSP**

| Setting | Value |
|---------|-------|
| **Enable** | ✅ Checked |
| **Title** | `Credit/Debit Card` (shown to customers) |
| **Description** | `Pay securely with Visa or Mastercard` |
| **API URL** | `https://sentinelgate.biz` |
| **API Key** | Your merchant API key (e.g., `sg_key_...`) |
| **API Secret** | Your merchant API secret (e.g., `sg_secret_...`) |
| **Integration Mode** | `Redirect` (recommended) |
| **Debug Log** | Enable during testing, disable in production |

Save changes.

### 3. Test

1. Add a product to cart
2. Proceed to checkout
3. Select **Credit/Debit Card** as payment method
4. Click **Place Order**
5. You should be redirected to the payment page
6. Complete payment with a real card
7. After payment, you're redirected back to the order confirmation page
8. Order status should update to **Processing** or **Completed**

---

## Plugin Modes

### Redirect Mode (Recommended)

Customer is redirected to the payment provider's hosted page. No PCI requirements.

```
Checkout → SentinelGate API → Provider hosted page → Payment → Redirect back to store
```

### iFrame Mode

Payment form loads inside an iframe on your checkout page. No PCI requirements.

```
Checkout → iFrame loads provider page → Payment → Callback updates order
```

### Direct Mode (PCI Required)

Card form embedded directly in your checkout page. **Requires PCI DSS Level 1 SAQ-D.**

```
Checkout → Card entered on your page → POST to SentinelGate → 3DS if needed → Complete
```

---

## How It Works

### Payment Flow

```
1. Customer clicks "Place Order"
2. Plugin calls POST /v1/hosted/create with:
   - Order amount & currency
   - Customer details
   - callback_url (your site's webhook endpoint)
   - return_url (order received page)
3. SentinelGate returns redirect_url
4. Customer is redirected to payment page
5. Customer enters card / confirms mobile money
6. Payment provider processes the charge
7. SentinelGate sends webhook to callback_url:
   {
     "sentinel_transaction_id": "sg_txn_...",
     "wc_order_id": "12345",
     "status": "captured",
     "amount": 125.00,
     "currency": "USD",
     "provider": "pesapal"
   }
8. Plugin receives webhook → marks order as Processing/Completed
9. Customer is redirected to order confirmation page
```

### Webhook Endpoint

The plugin registers a WooCommerce API endpoint at:

```
https://yourstore.com/?wc-api=sentinelgate_callback
```

This endpoint:
- Verifies the webhook signature
- Looks up the WooCommerce order by `wc_order_id`
- Updates order status based on `status` field
- Adds an order note with transaction details

### Order Status Mapping

| SentinelGate Status | WooCommerce Status | Description |
|--------------------|--------------------|-------------|
| `captured` | Processing | Payment received |
| `failed` | Failed | Payment declined |
| `refunded` | Refunded | Payment refunded |
| `cancelled` | Cancelled | Payment cancelled |

---

## Supported Currencies

The plugin automatically routes to the correct provider based on your store's currency:

| Store Currency | Payment Methods |
|---------------|-----------------|
| USD | Visa, Mastercard |
| KES | Visa, Mastercard, M-Pesa | 
| GHS | Visa, Mastercard, MoMo | 
| NGN | Visa, Mastercard, Verve | 

---

## Troubleshooting

### Order stuck on "Pending Payment"

**Cause:** Webhook not received by your store.

**Fix:**
1. Check WooCommerce → Status → Logs for `sentinelgate` entries
2. Verify your site is accessible from the internet (no firewall blocking)
3. Ensure SSL certificate is valid
4. Check that `callback_url` resolves correctly

### "Invalid signature" error

**Cause:** API secret mismatch between plugin and SentinelGate.

**Fix:** Double-check the API Secret in plugin settings matches exactly what was provided.

### Customer redirected but payment page doesn't load

**Cause:** Provider-side issue or invalid currency.

**Fix:** Check SentinelGate logs or contact support with the `sentinel_transaction_id`.

### Debug Logging

Enable debug in plugin settings, then check:

```
WooCommerce → Status → Logs → sentinelgate-psp-[date]
```

Log entries include:
- API request/response payloads
- Webhook received data
- Order status transitions

---

## Plugin Files

```
sentinelgate-psp/
├── sentinelgate-psp.php          # Main plugin file
├── includes/
│   ├── class-sg-gateway.php      # WooCommerce payment gateway
│   ├── class-sg-api.php          # API client
│   └── class-sg-webhook.php      # Webhook handler
├── assets/
│   ├── css/sg-checkout.css       # Checkout styling
│   └── js/sg-checkout.js         # Frontend JS (iFrame/Direct modes)
└── readme.txt                    # WordPress readme
```

---

## Requirements

- WordPress 5.8+
- WooCommerce 7.0+
- PHP 7.4+
- SSL certificate (HTTPS required)
- Valid SentinelGate API credentials
