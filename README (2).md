# SentinelGate Payment Gateway

**Enterprise-grade payment processing platform** — Accept payments globally via cards, mobile money, bank transfers, crypto, and digital wallets.

---

## Overview

SentinelGate provides a unified API for merchants to accept payments across 25+ payment methods and 12+ currencies. Integrate once and access card processing, mobile money, bank transfers, digital wallets, and crypto payments through a single REST API.

## Quick Start

```bash
# 1. Get your API credentials from the SentinelGate merchant portal
# 2. Create a hosted checkout session
curl -X POST https://sentinelgate.biz/v1/hosted/create \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "X-API-Secret: YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50.00,
    "currency": "USD",
    "description": "Order #1234",
    "callback_url": "https://yoursite.com/webhook",
    "return_url": "https://yoursite.com/success"
  }'
```

## Integration Modes

| Mode | Use Case | Complexity |
|------|----------|------------|
| **Hosted Checkout** | Redirect to SentinelGate payment page | Low |
| **Server-to-Server** | Direct API charge from your backend | Medium |
| **iFrame** | Embed payment form in your site | Medium |
| **Payment Links** | Share payment URLs via email/SMS | Low |
| **WooCommerce Plugin** | WordPress e-commerce integration | Low |
| **Shopify Middleware** | Shopify store integration | Low |

## Documentation

| Document | Description |
|----------|-------------|
| [Supported Payment Methods](docs/PAYMENT-METHODS.md) | All payment methods, currencies, and regions |
| [API Reference](docs/API-REFERENCE.md) | Complete endpoint reference |
| [Apple Pay & Google Pay](docs/WALLET-PAY.md) | Digital wallet integration guide |
| [Webhooks](docs/WEBHOOKS.md) | Payment status notifications |
| [Sample Code](docs/SAMPLES.md) | Code examples in Python, Node.js, PHP, cURL |
| [WooCommerce Plugin](docs/WOOCOMMERCE.md) | WordPress integration guide |
| [Shopify Integration](docs/SHOPIFY.md) | Shopify middleware setup |

## Supported Currencies

`USD` `EUR` `GBP` `CAD` `NGN` `MXN` `BRL` `JPY` `KES` `ZAR` `HKD` `CNY` `GHS` `UGX` `TZS` `RWF`

## Base URL

```
https://sentinelgate.biz
```

## Authentication

All API requests require two headers:

```
X-API-Key: sg_key_xxxxx
X-API-Secret: sg_secret_xxxxx
```

## Support

- **Portal**: https://sentinelgate.biz/merchant
- **Email**: support@sentinelgate.biz

---

© SentinelGate — Secure payments, everywhere.
