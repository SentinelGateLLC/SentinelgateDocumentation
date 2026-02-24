# Getting Started with SentinelGate Payments

**For:** Business Owners & Merchants (Non-Technical)
**Time to Complete:** 15 minutes
**Last Updated:** February 24, 2026

---

## What is SentinelGate?

SentinelGate is a payment platform that lets your online store accept payments from customers worldwide. We handle the complicated parts — you just sell your products.

**Payment methods your customers can use:**

- **Cards** — Visa and Mastercard (credit and debit)
- **Mobile Money (Ghana)** — MTN, Vodafone Cash, AirtelTigo
- **M-Pesa (Kenya)** — Direct from customer's phone via STK Push
- **Bank Transfers** — Direct bank payments

**Supported currencies:** USD, GHS (Ghana Cedis), KES (Kenya Shillings), UGX, NGN

---

## Step 1: Get Your Credentials

Your SentinelGate integration team will provide three credentials:

| Credential | What It Looks Like | What It Does |
|-----------|-------------------|-------------|
| **API Key** | `sg_key_yourstore_abc123...` | Identifies your store |
| **API Secret** | `sg_secret_yourstore_def456...` | Authenticates payments |
| **Webhook Secret** | `sg_whsec_yourstore_ghi789...` | Verifies payment notifications |

**Keep these safe:**
- Do not share them in emails, chats, or social media
- Do not post them publicly or in screenshots
- If you think they've been compromised, contact support@sentinelgate.biz immediately for a reset

---

## Step 2: Choose How to Connect Your Store

### Option A: WooCommerce Plugin (Easiest — 10 minutes)

If your store runs on WordPress + WooCommerce:

1. Download `sentinelgate-psp.zip` from your integration team
2. Go to WordPress Admin → **Plugins → Add New → Upload Plugin**
3. Upload the zip file → **Install → Activate**
4. Go to **WooCommerce → Settings → Payments → SentinelGate PSP**
5. Enter your API Key, API Secret, and Webhook Secret
6. Set Integration Mode to **Redirect Hosted Checkout**
7. Click **Save Changes**

Full guide: [WooCommerce Integration](Woocommerce_integration·MD)

### Option B: Shopify (30 minutes)

If your store runs on Shopify:

1. Create a Custom App in Shopify Admin (Settings → Apps → Develop Apps)
2. Share your Shopify Admin API token with SentinelGate
3. Set up webhooks in Shopify pointing to SentinelGate
4. Add a payment redirect script to your checkout
5. Add a manual payment method called "Pay with Card / Mobile Money"

Full guide: [Shopify Integration](./SHOPIFY_INTEGRATION.md)

### Option C: Custom Website (Developer Required)

If you have a custom-built website, share the [API Reference](API_Reference.md) with your developer. They will integrate using our REST API.

### Option D: Payment Links (No Website Needed)

Don't have a website? SentinelGate can create **payment links** for you — shareable URLs that customers click to pay. Each link comes with a QR code you can print or share.

Contact your integration team to set up payment links.

---

## Step 3: Test a Payment

Before going live, do a test purchase:

1. Add a product to your cart on your store
2. Go to checkout
3. Select the SentinelGate payment method
4. Click "Place Order"
5. You'll be redirected to the payment page
6. Complete the payment using a real card or mobile money
7. Verify the order shows as **"Processing"** in your store admin

**Tip:** Start with a small amount (e.g., $1 or GHS 5) for your first test.

---

## Step 4: Go Live

Once your test payment is confirmed:

- ✅ Your store is live and ready to accept real payments
- ✅ Disable any other test payment gateways
- ✅ Inform your team that the payment system is active

That's it. Customers can now pay on your store.

---

## What Your Customers See

When a customer places an order, they are redirected to a secure payment page:

```
Your Store Checkout → Click "Place Order" → Secure Payment Page → Pay → Back to Your Store
```

On the payment page, they can:
- Enter card details (number, expiry, CVV)
- Select mobile money and approve on their phone
- Complete 3D Secure verification if their bank requires it

After payment, they're automatically redirected to your order confirmation page.

The entire process takes 30-60 seconds.

---

## Understanding Your Payments

### Payment Statuses

| Status | What It Means | What You Do |
|--------|--------------|-------------|
| **Processing** | Payment received successfully | Fulfill the order |
| **Pending** | Customer hasn't completed payment yet | Wait or follow up |
| **Failed** | Payment was declined | Customer can retry |
| **Refunded** | Money returned to customer | Process the return |

### Where to See Your Orders

- **WooCommerce:** WordPress Admin → WooCommerce → Orders
- **Shopify:** Shopify Admin → Orders

Each order includes a SentinelGate transaction ID (e.g., `sg_txn_1771888643979_cfe07b9d7fe6`) in the order notes. Use this ID when contacting support.

---

## Payment Methods — What to Know

### Cards (Visa / Mastercard)

- Works globally
- Customer enters card details on the payment page
- 3D Secure (OTP) may be required by the customer's bank
- Processing time: Instant

### Mobile Money (Ghana)

- MTN Mobile Money, Vodafone Cash, AirtelTigo Money
- Customer selects their provider and approves on their phone
- Processing time: 10-30 seconds
- Customers pay in GHS (converted from USD if your store is in dollars)

### M-Pesa (Kenya)

- Customer receives an STK Push notification on their phone
- They enter their M-Pesa PIN to approve
- Processing time: 10-30 seconds
- Currency: KES

---

## Refunds

To refund a customer:

**WooCommerce:** Open the order → Click "Refund" → Enter amount → Submit

**Shopify:** Open the order → Click "Refund" → Enter amount → Submit

Refund timelines:
- **Cards:** 5-10 business days
- **Mobile Money:** 1-3 business days
- **M-Pesa:** 1-2 business days

---

## Currency and Conversion

If your store prices in USD but your payment provider operates in a local currency:

- The provider converts USD → local currency automatically
- The customer sees the converted amount on the payment page
- You receive settlement in the provider's currency
- Exchange rates are set by the provider, not SentinelGate

**Example:** A $50 USD order processed through Hubtel (Ghana) will show approximately GHS 750 on the payment page (at current rates).

---

## Fees and Settlements

Fees depend on your agreement with SentinelGate. Typical structure:

| Component | Description |
|-----------|------------|
| Transaction fee | Percentage of each successful payment |
| Fixed fee | Small amount per transaction |
| Conversion fee | Applied when currencies differ |

**Settlement:** Funds are deposited to your account, typically next business day (T+1). Settlement timelines vary by provider.

Contact your SentinelGate account manager for your specific fee schedule.

---

## Common Questions

**Q: How quickly do I get my money?**
Typically next business day (T+1). Varies by provider.

**Q: What if a customer's payment fails?**
They're notified immediately and can retry. No charge is made for failed attempts.

**Q: Can I accept payments from multiple countries?**
Yes. Card payments work globally. Mobile money is region-specific (Ghana, Kenya).

**Q: What if I need to change my payment provider?**
Contact SentinelGate. Provider changes happen server-side — no changes needed on your store.

**Q: Can I see all my transactions in one place?**
Yes. Through your WooCommerce or Shopify admin. A dedicated SentinelGate merchant dashboard is in development.

**Q: What if a customer says they paid but I don't see the order?**
Ask them for the SentinelGate transaction ID (starts with `sg_txn_`). Contact support@sentinelgate.biz with this ID.

**Q: Is my store secure?**
Yes. SentinelGate uses bank-grade encryption (AES-256), all communication is over HTTPS, and card details never touch your server. We are PCI compliant.

---

## Checklist

Use this to track your setup:

- [ ] Received API Key, API Secret, and Webhook Secret
- [ ] Chose integration method (WooCommerce / Shopify / Custom / Payment Links)
- [ ] Installed plugin or configured integration
- [ ] Completed a test payment
- [ ] Verified order appears in store admin
- [ ] Disabled other test payment gateways
- [ ] Ready to accept real payments

---

## Need Help?

| I Need... | Contact |
|-----------|---------|
| Help setting up my store | Your SentinelGate integration team |
| Help with a specific transaction | support@sentinelgate.biz (include transaction ID) |
| My credentials reset | support@sentinelgate.biz |
| To change payment providers | Your SentinelGate account manager |
| Technical documentation | [API Reference](./API_REFERENCE.md) |

---

*© 2026 SentinelGate. All rights reserved.*
