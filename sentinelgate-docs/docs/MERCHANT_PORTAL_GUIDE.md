# Merchant Portal Guide

The Merchant Portal is a self-service dashboard for managing your payments, transactions, and settings.

---

## Access

**URL:** `https://sentinelgate.biz:3200`

**Login Credentials:**
- **Tenant ID:** `default` (or your assigned tenant)
- **API Key:** Your merchant API key (`sk_live_...`)

---

## Dashboard Pages

### Overview

Your payment KPIs at a glance:
- Total transactions
- Total captured volume
- Success rate percentage
- Active payment rails (MIDs)
- Daily transaction breakdown chart
- Currency breakdown

### Transactions

Full transaction history with:
- Search by reference, amount, or status
- Filter by status (CAPTURED, FAILED, PENDING)
- Filter by currency
- Date range picker
- Paginated list with transaction details

### Payment Links

Create and manage payment links:
- **New Payment Link** button to create links from the portal
- List of all links with status, amount, and URL
- Copy link to clipboard
- QR code download
- Disable/expire links

### Routing

View your configured payment rails:
- Active provider configurations
- Supported currencies per rail
- Priority ordering

### MIDs & Limits

View your MID profiles and transaction limits:
- Daily transaction limits
- Per-transaction caps
- Currency restrictions

### Settlements

Settlement tracking:
- Settlement summary with totals
- Daily settlement breakdown
- Expected vs actual amounts

### Disputes

Dispute and chargeback management:
- Active disputes with status
- Dispute statistics
- Response tracking

### Reports

Generate and download reports:
- Transaction reports by date range
- Revenue summaries
- Settlement reconciliation

### Developers

API integration details:
- Your API keys (masked)
- Webhook configuration
- API endpoint reference
- Code examples

### Audit Log

Activity history:
- All transaction events
- Status changes with timestamps
- Webhook delivery logs

### Support

Submit and track support requests directly from the portal.

---

## Creating a Payment Link (Step by Step)

1. Click **Payment Links** in the sidebar
2. Click **New Payment Link**
3. Fill in:
   - Title (e.g., "Monthly Subscription")
   - Type (Standard / Invoice / Gaming)
   - Amount (or leave blank for open amount)
   - Currency
   - Description (optional)
   - Expiry date (optional)
4. Click **Create Payment Link**
5. Copy the URL or scan the QR code
6. Share with your customer

---

## Tips

- Use the date filters on Transactions to quickly find specific payments
- Export transaction data from the Developers page for reconciliation
- Set up webhook endpoints in the Developers section
- Monitor your success rate on the Overview dashboard

---

© 2026 SentinelGate — Whyte AG Group
