# SentinelGate — WooCommerce Integration

**For developers and store administrators installing SentinelGate on WordPress/WooCommerce**

---

## Overview

SentinelGate integrates with WooCommerce via a payment gateway plugin. The plugin creates payment sessions through the SentinelGate API and redirects customers to a secure hosted checkout page to complete payment. After payment, the order is automatically updated in WooCommerce.

---

## Requirements

| Requirement | Minimum Version |
|------------|-----------------|
| WordPress | 6.0+ |
| WooCommerce | 7.0+ |
| PHP | 7.4+ |
| SSL Certificate | Required (HTTPS) |

You also need SentinelGate merchant credentials. See the [Merchant Guide](./MERCHANT_GUIDE.md) for how to obtain them.

---

## Installation

### Step 1 — Download and Install

1. Obtain `sentinelgate-psp.zip` from your SentinelGate integration team
2. In WordPress Admin, go to **Plugins → Add New → Upload Plugin**
3. Select the zip file and click **Install Now**
4. Click **Activate Plugin**

### Step 2 — Configure

1. Go to **WooCommerce → Settings → Payments**
2. Find **SentinelGate PSP** and click **Manage**
3. Fill in the settings:

| Setting | Value | Notes |
|---------|-------|-------|
| **Enable/Disable** | ✅ Checked | Enables the gateway at checkout |
| **Title** | `Pay Now` | Displayed to customers at checkout |
| **Description** | `Secure payment via SentinelGate` | Subtitle below the title |
| **Integration Mode** | `Redirect Hosted Checkout` | Recommended for all merchants |
| **API Base URL** | `https://sentinelgate.biz` | Do not include trailing slash |
| **API Key** | Your `sg_key_...` credential | From your SentinelGate account |
| **API Secret** | Your `sg_secret_...` credential | From your SentinelGate account |
| **Webhook Secret** | Your `sg_whsec_...` credential | Used to verify callbacks |
| **Debug Log** | ✅ Checked (during setup) | Disable after confirming payments work |

4. Click **Save Changes**

### Step 3 — Disable Other Gateways (Recommended)

If you have other payment gateways enabled (like the default "Credit Card Payment"), disable them to avoid customer confusion:

1. On the same **Payments** settings page
2. Toggle off any gateways you don't want active
3. Save changes

### Step 4 — Verify the Webhook Endpoint

The plugin automatically creates a webhook endpoint:

```
https://yourstore.com/wc-api/sentinelgate_callback/
```

Verify it's accessible by visiting the URL in your browser. You should see a blank page or a short response — not a 404 error. If you get a 404:

- Check that WooCommerce permalinks are set (Settings → Permalinks → Save)
- Check that no security plugin is blocking the URL
- Try flushing rewrite rules (Settings → Permalinks → Save, even without changes)

---

## Payment Flow

### What happens when a customer places an order:

```
1. Customer fills checkout form → clicks "Place Order"
                    ↓
2. Plugin calls POST /v1/hosted/create on SentinelGate
   Sends: amount, currency, order ID, customer email,
          callback URL, return URL, cancel URL
                    ↓
3. SentinelGate returns a redirect URL
                    ↓
4. Customer is redirected to the payment page
   They see payment options (card form, mobile money, etc.)
                    ↓
5. Customer completes payment
                    ↓
6. Payment provider confirms to SentinelGate
                    ↓
7. SentinelGate sends webhook POST to:
   https://yourstore.com/wc-api/sentinelgate_callback/
   Payload: { status: "captured", transaction_id, amount, ... }
                    ↓
8. Plugin verifies webhook → updates order status to "Processing"
   Adds transaction ID to order notes
                    ↓
9. Customer is redirected to order confirmation page
```

### Order Status Mapping

| SentinelGate Status | WooCommerce Status | Meaning |
|--------------------|-------------------|---------|
| `captured` | Processing | Payment received, ready to fulfill |
| `failed` | Failed | Payment declined |
| `refunded` | Refunded | Money returned to customer |

---

## Integration Modes (Technical Detail)

The plugin supports three modes. **Redirect** is recommended for nearly all merchants.

### Mode B: Redirect Hosted Checkout (Recommended)

- Customer is redirected to the SentinelGate/provider payment page
- No PCI compliance required on your end
- Supports all payment methods (card, mobile money, etc.)
- 3D Secure and OTP handled automatically

### Mode A: Direct API

- Card form rendered on your checkout page
- **Requires PCI DSS Level 1 compliance** — most merchants should NOT use this
- Card data passes through your server to SentinelGate

### Mode C: iFrame Token

- SentinelGate iframe embedded in your checkout
- Requires PCI DSS SAQ A-EP compliance
- Card data tokenized in the iframe, token sent to your server

---

## Plugin File Structure

```
sentinelgate-psp/
├── sentinelgate-psp.php                          # Main plugin file
├── includes/
│   └── class-sentinelgatepsp-gateway.php         # WooCommerce gateway class
├── assets/
│   └── js/
│       └── sentinelgate.js                       # Frontend JS (iFrame mode only)
├── uninstall.php                                 # Cleanup on delete
└── readme.txt                                    # Plugin metadata
```

---

## API Request/Response (What the Plugin Sends)

### Creating a Payment Session

**Request** (sent by the plugin when customer clicks "Place Order"):

```http
POST /v1/hosted/create HTTP/1.1
Host: sentinelgate.biz
X-API-Key: sg_key_yourstore_abc123
X-API-Secret: sg_secret_yourstore_def456
Content-Type: application/json

{
  "amount": "191.00",
  "currency": "USD",
  "order_id": "7700",
  "description": "Order #7700 from Your Store",
  "customer_email": "buyer@example.com",
  "customer_name": "John Doe",
  "callback_url": "https://yourstore.com/wc-api/sentinelgate_callback/",
  "return_url": "https://yourstore.com/checkout/order-received/7700/",
  "cancel_url": "https://yourstore.com/checkout/"
}
```

**Response:**

```json
{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "session_id": "sg_session_cc7d30bba805dca1c7c0828b",
  "redirect_url": "https://pay.provider.com/checkout-id",
  "status": "pending"
}
```

The plugin then redirects the customer to `redirect_url`.

### Webhook Callback

**Request** (sent by SentinelGate to your store after payment):

```http
POST /wc-api/sentinelgate_callback/ HTTP/1.1
Host: yourstore.com
Content-Type: application/json
X-Sentinel-Signature: sha256=abc123...

{
  "sentinel_transaction_id": "sg_txn_1771888643979_cfe07b9d7fe6",
  "wc_order_id": "7700",
  "status": "captured",
  "amount": 191.00,
  "currency": "USD",
  "provider": "hubtel",
  "gateway_response": "Approved"
}
```

The plugin verifies the `X-Sentinel-Signature` header using your Webhook Secret, then updates the order.

---

## Troubleshooting

### "Transaction Declined (Sandbox Mode)" error

This is NOT from SentinelGate. Another payment gateway (often the default WooCommerce one) is in sandbox mode. Disable all other payment gateways in WooCommerce → Settings → Payments.

### Customer is not redirected after clicking "Place Order"

- Check WooCommerce logs: **WooCommerce → Status → Logs → sentinelgate**
- Verify the API Base URL is correct (`https://sentinelgate.biz`, no trailing slash)
- Verify the API Key and Secret are correct
- Test the connection manually:

```bash
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "X-API-Key: your_key" \
  -H "X-API-Secret: your_secret" \
  -H "Content-Type: application/json" \
  -d '{"amount":"1.00","currency":"USD","order_id":"test-001","customer_email":"test@test.com"}'
```

### Order stays in "Pending Payment" after customer pays

The webhook isn't reaching your store. Check:

1. Is your webhook URL accessible? Visit `https://yourstore.com/wc-api/sentinelgate_callback/` — should not return 404
2. Is a firewall or security plugin blocking POST requests?
3. Is Cloudflare or another CDN blocking the callback?
4. Check SentinelGate server logs for webhook delivery errors

### Customer sees wrong currency on payment page

If your store is in USD but the payment page shows GHS (or another currency), this is because the payment provider operates in a local currency. The provider converts at their current rate. The customer is charged the correct equivalent amount.

### Debug Logging

Enable debug mode in plugin settings. Logs are saved to:

```
wp-content/uploads/wc-logs/sentinelgate-*.log
```

Or view them in **WooCommerce → Status → Logs** and select the `sentinelgate` file.

---

## Updating the Plugin

1. Download the new version of `sentinelgate-psp.zip`
2. In WordPress Admin, go to **Plugins → Installed Plugins**
3. **Deactivate** SentinelGate PSP
4. **Delete** SentinelGate PSP
5. Install the new version (Plugins → Add New → Upload Plugin)
6. **Activate** the plugin
7. Your settings are preserved — verify in WooCommerce → Settings → Payments

---

## Uninstalling

1. Go to **Plugins → Installed Plugins**
2. Click **Deactivate** under SentinelGate PSP
3. Click **Delete**
4. All plugin settings are removed automatically
