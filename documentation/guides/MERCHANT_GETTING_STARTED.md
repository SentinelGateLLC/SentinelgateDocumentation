# Getting Started with SentinelGate Payments

**For:** Business Owners & Merchants (Non-Technical)  
**Time to Complete:** 15 minutes

---

## What is SentinelGate?

SentinelGate is a payment platform that lets you accept payments from customers using:
- **M-Pesa** - Mobile money payments
- **Credit/Debit Cards** - Visa, Mastercard
- **Bank Transfers** - Direct bank payments

---

## Step 1: Get Your Credentials

You should have received an email with:
- **Merchant ID** - Your unique identifier (e.g., `brooks-ud-zone`)
- **API Key** - Starts with `sk_live_...`
- **API Secret** - Starts with `secret_...`
- **Webhook Secret** - Starts with `whsec_...`

⚠️ **IMPORTANT:** Keep these credentials secure. Never share them publicly or commit them to code repositories.

---

## Step 2: Choose Your Integration Method

### Option A: Use Our Plugins (Easiest)

**For Shopify Users:**
1. Download the SentinelGate Shopify plugin
2. Install it in your Shopify store
3. Enter your credentials
4. Done! Payments will work automatically

**For WooCommerce Users:**
1. Download the SentinelGate WooCommerce plugin
2. Install via WordPress admin
3. Configure your credentials
4. Activate payment methods

### Option B: Hire a Developer

Share this documentation package with your developer. They will integrate the API into your website or app.

### Option C: Use Our Integration Service

Contact us at support@sentinelgate.com for professional integration assistance.

---

## Step 3: Test Your Integration

### Test Payments

Use these test details to verify everything works:

**Test M-Pesa Number:**
```
254712345678
```

**Test Card:**
```
Card Number: 4111 1111 1111 1111
Expiry: 12/25
CVV: 123
```

**Expected Result:** You should receive a payment confirmation within 1-2 minutes.

---

## Step 4: Go Live

Once testing is complete:
1. Ensure you're using **production credentials** (not test/sandbox)
2. Update your webhook URL to your live website
3. Process a small real transaction to verify
4. You're ready to accept payments!

---

## Understanding Payment Flow
```
Customer → Your Website → SentinelGate → Payment Provider → Customer's Bank/Phone
                                    ↓
                            Confirmation Sent Back
```

**Timeline:**
- **M-Pesa:** 10-30 seconds
- **Cards:** Instant to 2 minutes
- **Bank Transfer:** 1-24 hours

---

## Webhooks Explained (Simple)

**What is a webhook?**
Think of it like a phone call. When a payment is complete, SentinelGate "calls" your website to let it know.

**Why do I need it?**
So your website can automatically:
- Mark orders as paid
- Send confirmation emails
- Update inventory

**Your developer will handle this technical part.**

---

## Payment Methods Available

### M-Pesa (Kenya)
- **Min Amount:** 10 KES
- **Max Amount:** 150,000 KES
- **Speed:** 10-30 seconds
- **Fee:** 1.5% + KES 5

### Cards (BUNI)
- **Supported:** Visa, Mastercard
- **Min Amount:** 100 KES
- **Max Amount:** 1,000,000 KES
- **Speed:** Instant
- **Fee:** 2.9% + KES 10

### Bank Transfer (KareenHub)
- **Min Amount:** 100 KES
- **Max Amount:** 5,000,000 KES
- **Speed:** 1-24 hours
- **Fee:** 1% (max KES 500)

---

## Common Questions

**Q: How long does it take to get paid out?**
A: Typically T+1 (next business day) to your settlement account.

**Q: What happens if a payment fails?**
A: The customer is notified immediately and can retry. No charges are made.

**Q: Can I refund a customer?**
A: Yes! Contact support@sentinelgate.com or use the refund API.

**Q: Is there a transaction limit?**
A: Yes, limits vary by payment method (see above).

**Q: Who do I contact for support?**
A: Email support@sentinelgate.com or call +254-XXX-XXXXXX

---

## Dashboard Access (Coming Soon)

You will soon be able to:
- View all transactions
- Download reports
- Issue refunds
- Manage settings

Access at: http://185.229.224.244:3002

---

## Next Steps

- [ ] Received credentials
- [ ] Chose integration method
- [ ] Completed test transaction
- [ ] Went live
- [ ] Processed first real payment

**Need Help?** Contact support@sentinelgate.com

---

**© 2026 SentinelGate. All rights reserved.**
