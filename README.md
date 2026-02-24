# SentinelGate PSP

**Secure Payment Gateway Platform**

SentinelGate is a payment gateway that connects online stores to multiple payment providers through a single, unified API. It supports card payments, mobile money, and bank transfers across multiple currencies.

---

## Who Is This For?

| Audience | Start Here |
|----------|------------|
| **Merchants** setting up payments for their store | [Merchant Setup Guide](documentation/guides/MERCHANT_GETTING_STARTED.md) |
| **Developers** integrating SentinelGate into WooCommerce | [WooCommerce Integration](Woocommerce_integration·MD) |
| **Developers** integrating SentinelGate into Shopify | [Shopify Integration](guides/SHOPIFY_INTEGRATION_GUIDE.md) |
| **Developers** building custom integrations | [API Reference](API_Reference.md) |
| **Customers** with questions about paying | [Customer FAQ](CUSTOMER_FAQ.md) |

---

## How It Works

SentinelGate sits between your online store and payment providers. Your store talks to SentinelGate, and SentinelGate handles the complexity of routing payments to the right provider.

```
┌─────────────────┐       ┌──────────────────┐       ┌─────────────────┐
│   Your Store    │       │   SentinelGate   │       │                │
│                 │       │                  │       │                 │
│  WooCommerce    │──────▶│  Hosted Checkout  │──────▶│  PROVIDER     │
│  Shopify        │       │  Payment Links   │       │                │
│  Custom Site    │       │  Webhooks        │       │                 │
│                 │◀──────│  Order Updates   │◀──────│                │
└─────────────────┘       └──────────────────┘       │                 │
                                                     └─────────────────┘
```

**For the merchant:** You install a plugin or add a few lines of code. SentinelGate handles everything else.

**For the customer:** They see a clean, secure payment page. Enter card details or use mobile money. Get a confirmation.

**For the developer:** One API, multiple providers. Automatic failover, webhook callbacks, idempotent processing.

---

## Supported Platforms

| Platform | Integration Type | Setup Time |
|----------|-----------------|------------|
| **WooCommerce** | Plugin (zip upload) | ~10 minutes |
| **Shopify** | Webhook middleware + redirect script | ~30 minutes |
| **Custom Website** | REST API | Varies |

---

## Supported Payment Methods

| Method | Currencies | Providers |
|--------|-----------|-----------|
| **Visa / Mastercard** | USD, GHS | Hubtel, Paystack, Emergent |
| **Mobile Money (Ghana)** | GHS | Hubtel |
| **M-Pesa (Kenya)** | KES | BUNI/KCB |
| **Mobile Money (East Africa)** | KES, UGX, TZS | Pesapal |
| **Bank Transfer** | USD, NGN | Paystack, Korapay |

---

## Quick Start

### WooCommerce (Fastest)

1. Download `sentinelgate-psp.zip`
2. WordPress Admin → Plugins → Upload → Activate
3. WooCommerce → Settings → Payments → SentinelGate PSP
4. Enter your API Key, Secret, and Webhook Secret
5. Save → Test a purchase

Full guide: [WooCommerce Integration](./docs/WOOCOMMERCE_INTEGRATION.md)

### Shopify

1. Create a Custom App in Shopify with order permissions
2. Register your store with SentinelGate
3. Add webhooks pointing to SentinelGate
4. Add redirect script to checkout
5. Test an order

Full guide: [Shopify Integration](./docs/SHOPIFY_INTEGRATION.md)

### Custom Integration

```bash
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "X-API-Key: your_api_key" \
  -H "X-API-Secret: your_api_secret" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50.00",
    "currency": "USD",
    "order_id": "ORD-001",
    "customer_email": "buyer@example.com",
    "callback_url": "https://yoursite.com/payment-callback",
    "return_url": "https://yoursite.com/thank-you"
  }'
```

Redirect the customer to the `redirect_url` in the response. Full guide: [API Reference](API_Reference.md)

---

## Security

SentinelGate is built with security as a core requirement:

- **HTTPS/TLS 1.3** — All communication is encrypted in transit
- **AES-256-GCM** — Sensitive credentials encrypted at rest
- **HMAC-SHA256** — All webhooks are signature-verified
- **No card storage** — Card details are never stored on SentinelGate servers
- **PCI Compliant** — Hosted checkout means merchants don't need PCI certification
- **Idempotent processing** — Duplicate webhooks are safely ignored
- **Rate limiting** — All endpoints protected against abuse

---

## Support

| Channel | Details |
|---------|---------|
| **Email** | support@sentinelgate.biz |
| **Documentation** | https://sentinelgate.biz/docs |
| **Status Page** | https://sentinelgate.biz/status |

---

## Documentation Index

```
docs/
├── MERCHANT_GUIDE.md           # For store owners: setup, dashboard, managing payments
├── WOOCOMMERCE_INTEGRATION.md  # WooCommerce plugin installation and configuration
├── SHOPIFY_INTEGRATION.md      # Shopify webhook and redirect setup
├── API_REFERENCE.md            # Full API documentation for developers
└── CUSTOMER_FAQ.md             # FAQ for end customers making payments
```
