# WooCommerce Integration

Integrate SentinelGate into your WooCommerce store in under 10 minutes.

---

## Prerequisites

- WordPress 5.8 or later
- WooCommerce 7.0 or later
- PHP 7.4 or later
- SSL certificate (HTTPS)
- SentinelGate merchant credentials

---

## Installation

### Option 1: Plugin Upload (Recommended)

1. Download `sentinelgate-psp.zip` from your onboarding package
2. Go to **WordPress Admin → Plugins → Add New → Upload Plugin**
3. Select the ZIP file and click **Install Now**
4. Click **Activate Plugin**

### Option 2: Manual Upload

1. Extract the ZIP file
2. Upload the `sentinelgate-psp` folder to `/wp-content/plugins/`
3. Go to **WordPress Admin → Plugins**
4. Find "SentinelGate PSP" and click **Activate**

---

## Configuration

1. Go to **WooCommerce → Settings → Payments**
2. Find **SentinelGate PSP** and click **Manage**
3. Fill in the settings:

| Setting | Value |
|---------|-------|
| **Enable/Disable** | Check to enable |
| **Title** | "Pay with Card or Mobile Money" (shown to customers) |
| **Description** | "Secure payment via SentinelGate" |
| **API Key** | Your `sk_live_` key |
| **Webhook Secret** | Your webhook signing secret |
| **Mode** | Production |

4. Click **Save Changes**

---

## Plugin Modes

The plugin supports three checkout modes:

| Mode | Description |
|------|-------------|
| **Redirect** | Customer is redirected to SentinelGate hosted checkout |
| **Direct** | Payment form embedded in WooCommerce checkout page |
| **iFrame** | SentinelGate checkout loaded in an iframe overlay |

Redirect mode is the default and recommended option — it requires no PCI compliance on your end.

---

## Webhook Configuration

The plugin automatically registers a webhook endpoint at:

```
https://yoursite.com/?wc-api=sentinelgate_webhook
```

This URL receives payment status updates and automatically marks orders as:
- **Processing** — when payment is captured
- **Failed** — when payment fails
- **On Hold** — when payment is pending

---

## Testing

1. Place a test order on your store
2. Select "SentinelGate PSP" at checkout
3. Complete the payment on the hosted checkout page
4. Verify the order status in WooCommerce → Orders

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| Payment gateway not showing at checkout | Ensure plugin is activated and enabled in WooCommerce Payments settings |
| "Invalid API Key" error | Double-check your API key in the plugin settings |
| Order not updating after payment | Verify your webhook URL is accessible from the internet |
| SSL error | Ensure your site has a valid SSL certificate |
| Redirect loop | Check that success_url and cancel_url are different from the checkout page |

---

## Uninstalling

1. Go to **Plugins → Installed Plugins**
2. Deactivate "SentinelGate PSP"
3. Click **Delete**

This removes the plugin but does not affect past orders.

---

© 2026 SentinelGate — Whyte AG Group
