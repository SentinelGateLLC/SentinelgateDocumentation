# SentinelGate — Merchant Guide

**For store owners and business managers**

This guide covers everything you need to know about accepting payments through SentinelGate, from getting your credentials to understanding your transactions.

---

## Getting Started

### What You Need

Before accepting payments, you need a SentinelGate merchant account. Contact the SentinelGate team to get set up. They will provide:

| Credential | What It Is | Looks Like |
|-----------|-----------|------------|
| **API Key** | Identifies your store | `sg_key_yourstore_abc123...` |
| **API Secret** | Authenticates requests | `sg_secret_yourstore_def456...` |
| **Webhook Secret** | Verifies payment notifications | `sg_whsec_yourstore_ghi789...` |

Keep these credentials **private**. Never share them publicly, commit them to source code, or send them over unencrypted channels.

### Choosing Your Platform

| If Your Store Runs On | Follow This Guide |
|-----------------------|-------------------|
| WooCommerce (WordPress) | [WooCommerce Integration](./WOOCOMMERCE_INTEGRATION.md) |
| Shopify | [Shopify Integration](./SHOPIFY_INTEGRATION.md) |
| Custom website | [API Reference](./API_REFERENCE.md) |

---

## How Payments Work

When a customer places an order on your store, here's what happens:

1. **Customer clicks "Pay"** — Your store creates a payment session with SentinelGate
2. **Customer is redirected** — They land on a secure payment page
3. **Customer enters payment details** — Card number, mobile money, etc.
4. **Payment is processed** — The payment provider handles the transaction
5. **Your store is notified** — SentinelGate sends a webhook to update the order
6. **Customer returns to your store** — They see the order confirmation

The entire process typically takes 30-60 seconds.

### What the Customer Sees

The payment page is clean and professional. Depending on your provider, customers may see:

- A card entry form (number, expiry, CVV)
- Mobile money options (MTN, Vodafone, AirtelTigo)
- 3D Secure verification (OTP sent to their phone)

After payment, they are automatically returned to your store with their order confirmed.

---

## Payment Methods and Currencies

### Available Payment Methods

| Method | Where It Works | How It Works |
|--------|---------------|-------------|
| **Visa / Mastercard** | Global | Customer enters card details on checkout page |
| **Mobile Money** | Ghana | Customer pays from their MoMo wallet (MTN, Vodafone, AirtelTigo) |
| **M-Pesa** | Kenya | STK Push sent to customer's phone |
| **Bank Transfer** | Nigeria, Global | Customer transfers to a generated bank account |

### Currency Support

| Currency | Code | Payment Methods |
|----------|------|-----------------|
| US Dollar | USD | Card, Bank Transfer |
| Ghana Cedi | GHS | Card, Mobile Money |
| Kenya Shilling | KES | M-Pesa, Card |
| Uganda Shilling | UGX | Mobile Money |
| Nigerian Naira | NGN | Card, Bank Transfer |

**Note on currency conversion:** If your store prices in USD and your provider operates in a local currency (e.g., GHS), the payment provider handles conversion at their current exchange rate. The customer sees the converted amount on the payment page.

---

## Understanding Transaction Statuses

Every payment goes through a series of statuses:

| Status | Meaning | What To Do |
|--------|---------|-----------|
| **Pending** | Payment session created, waiting for customer | Nothing — customer is on the payment page |
| **Processing** | Payment submitted, waiting for provider confirmation | Wait — usually resolves in seconds |
| **Captured / Success** | Payment received successfully | Fulfill the order |
| **Failed / Declined** | Payment was not successful | Customer can retry with a different card or method |
| **Refunded** | Payment was returned to the customer | Process the return/exchange |
| **Expired** | Customer never completed payment | Contact customer or cancel the order |

---

## Managing Your Payments

### Viewing Transactions

Transactions can be viewed in your store's admin panel:

**WooCommerce:** Go to WooCommerce → Orders. Each order shows the SentinelGate transaction ID in the order notes.

**Shopify:** Go to Orders. Transaction details appear in the order timeline.

### Transaction IDs

Every payment gets a unique SentinelGate transaction ID:

```
sg_txn_1771888643979_cfe07b9d7fe6
```

Use this ID when contacting support about a specific transaction.

### Refunds

To refund a payment:

**WooCommerce:** Open the order → Click "Refund" → Enter the amount → Submit. The plugin will process the refund through SentinelGate.

**Shopify:** Open the order → Click "Refund" → Enter the amount → Submit. SentinelGate processes the refund automatically.

**Note:** Refunds typically take 5-10 business days to appear on the customer's statement, depending on their bank.

---

## Common Scenarios

### Customer says "I was charged but didn't get a confirmation"

1. Check your orders for a matching transaction
2. If the order exists but wasn't confirmed, the webhook may not have been received
3. Check your WooCommerce/Shopify logs for webhook errors
4. Contact SentinelGate support with the transaction ID

### Customer wants to pay with a different method

Customers choose their payment method on the hosted checkout page. If they want to switch, they can click "Cancel" on the payment page and return to your checkout to try again.

### Payment keeps declining

Common reasons for declined payments:

| Reason | Solution |
|--------|---------|
| Insufficient funds | Customer should try a different card |
| Card not enabled for online payments | Customer should contact their bank |
| 3D Secure failed | Customer entered wrong OTP |
| Currency not supported | Ensure your store currency matches provider capabilities |
| Daily limit exceeded | Customer should try the next business day |

### I need to change my payment provider

Contact the SentinelGate team. Provider changes are handled server-side — no changes needed in your store. Your API credentials remain the same.

---

## Fees and Settlements

Payment processing fees depend on your agreement with SentinelGate and the underlying payment provider. Typical fee structures:

| Component | Description |
|-----------|------------|
| **Transaction fee** | Percentage of each successful transaction |
| **Fixed fee** | Small fixed amount per transaction |
| **Currency conversion** | Applied by the provider when converting currencies |
| **Chargeback fee** | Applied if a customer disputes a charge with their bank |

Settlement timelines vary by provider:

| Provider | Settlement Period |
|----------|------------------|
| Hubtel | Next business day |
| Pesapal | 1-3 business days |
| Paystack | Next business day |

Contact SentinelGate for your specific fee schedule and settlement terms.

---

## Security Best Practices

1. **Keep credentials private** — Never share API keys in emails, chat, or source code
2. **Use HTTPS** — Ensure your store has a valid SSL certificate
3. **Monitor transactions** — Review your orders regularly for unusual activity
4. **Update your plugin** — Install updates when they become available
5. **Enable debug logging during setup** — Turn it off once everything is working
6. **Test before going live** — Always do a test transaction after setup changes

---

## Getting Help

| I Need... | Contact |
|-----------|---------|
| **Help setting up** | Your SentinelGate integration team |
| **A transaction investigated** | support@sentinelgate.biz (include transaction ID) |
| **My credentials reset** | support@sentinelgate.biz |
| **To change payment providers** | Your SentinelGate account manager |
| **Technical API help** | [API Reference](./API_REFERENCE.md) |
