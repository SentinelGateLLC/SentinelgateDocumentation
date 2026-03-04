# Admin Console Guide

The SentinelGate Admin Console provides platform-wide visibility into transactions, merchants, and system health.

---

## Access

**URL:** `https://sentinelgate.biz/console/admin`

**Authentication:** Admin API key via `x-api-key` header. Admin keys are prefixed with `sg_admin_`.

**Roles:**

| Role | Permissions |
|------|------------|
| SUPER_ADMIN | Full platform access — all endpoints and data |
| PLATFORM_OPS | Metrics, analytics, merchant management, provider health |
| PARTNER_ADMIN | Scoped to their partner's merchants and transactions |
| SUPPORT_AGENT | Read-only access to merchant and transaction data |

---

## Dashboard Sections

### Overview

Real-time platform metrics:
- Total transactions (all time)
- Success / Failed / Pending counts
- Total captured volume (USD)
- Daily transaction chart (configurable 7/14/30 days)

### Merchants

- Full list of onboarded merchants
- Per-merchant transaction count, volume, and success rate
- Merchant status (ACTIVE / INACTIVE / SUSPENDED)
- Click through to view merchant's transaction history

### Partners

- Partner/ISO hierarchy (PSP → ISO → Merchants)
- Partner-level aggregate metrics
- Merchant assignments per partner

### Analytics

- Revenue breakdown: daily, weekly, monthly with month-over-month comparison
- Merchant ranking by volume, transaction count, or success rate
- Provider performance metrics
- CSV export for transactions and merchant data

### Support

- Merchant and partner counts
- Total transaction summary
- Quick access to merchant lookup

---

## API Endpoints

All admin endpoints require the `x-api-key` header with an admin key.

| Endpoint | Description |
|----------|-------------|
| `GET /admin/crm/whoami` | Your admin identity and role |
| `GET /admin/crm/metrics/summary` | Platform transaction summary |
| `GET /admin/crm/metrics/daily?days=14` | Daily breakdown |
| `GET /admin/crm/metrics/merchants` | Per-merchant stats |
| `GET /admin/support/metrics` | Merchant/partner/transaction counts |
| `GET /admin/analytics/revenue?days=30` | Revenue with MoM comparison |
| `GET /admin/analytics/merchants?days=30&sort=volume` | Merchant ranking |
| `GET /admin/analytics/providers` | Provider breakdown |
| `GET /admin/analytics/export/transactions` | CSV export |
| `GET /admin/analytics/export/merchants` | CSV export |

---

## BackOffice Authentication

The BackOffice system uses a separate auth flow:

- **Auth method:** PBKDF2 password hashing + JWT tokens
- **Login URL:** `/backoffice/`
- **6 dashboard views:** Admin, Agents, Partners, ISO, Staff, Compliance

---

## Common Tasks

### Look Up a Transaction

```bash
curl https://sentinelgate.biz/admin/support/merchants/{merchant_id}/transactions \
  -H "x-api-key: sg_admin_your_key"
```

### Export Transactions to CSV

```bash
curl https://sentinelgate.biz/admin/analytics/export/transactions \
  -H "x-api-key: sg_admin_your_key" \
  -o transactions.csv
```

### Check Platform Health

```bash
curl https://sentinelgate.biz/admin/crm/whoami \
  -H "x-api-key: sg_admin_your_key"
```

---

© 2026 SentinelGate — Whyte AG Group
