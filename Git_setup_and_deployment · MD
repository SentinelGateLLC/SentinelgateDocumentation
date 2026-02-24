# SentinelGate — Git Setup & Deployment Guide

**For the development team: setting up the repository, cloning, and deploying**

---

## Table of Contents

1. [Repository Structure](#repository-structure)
2. [Initial Git Setup (On Server)](#initial-git-setup-on-server)
3. [Cloning & Local Development](#cloning--local-development)
4. [Environment Configuration](#environment-configuration)
5. [Database Setup](#database-setup)
6. [Building & Running](#building--running)
7. [Deployment to Production](#deployment-to-production)
8. [PM2 Process Management](#pm2-process-management)
9. [Apache Reverse Proxy](#apache-reverse-proxy)
10. [SSL Certificates](#ssl-certificates)
11. [Troubleshooting](#troubleshooting)

---

## Repository Structure

```
svc-rails/
├── .env.example                  # Environment template (copy to .env)
├── .gitignore                    # Git ignore rules
├── package.json                  # Node.js dependencies
├── package-lock.json             # Locked dependency versions
├── tsconfig.json                 # TypeScript configuration
├── Dockerfile                    # Container build file
│
├── prisma/
│   ├── schema.prisma             # Database schema (PostgreSQL)
│   ├── migrations/               # Database migration history
│   ├── seed.ts                   # Database seed script
│   └── seed/                     # Seed data files
│
├── scripts/
│   ├── onboard-merchant.ts       # Register new merchant
│   ├── create-api-key-for-existing.ts
│   ├── seed-providers-fastflow.ts
│   ├── setup-fastflow.ts
│   └── setup-maveriq.ts
│
├── src/
│   ├── main.ts                   # Application entry point (Express server)
│   │
│   ├── routes/                   # API route handlers
│   │   ├── psp.routes.ts         # PSP endpoints (/v1/charge, /v1/hosted/*)
│   │   ├── payment-links.routes.ts   # Payment links (/pay/:token)
│   │   ├── buni.routes.ts        # BUNI M-Pesa callbacks
│   │   ├── hubtel.routes.ts      # Hubtel callbacks
│   │   ├── pesapal.routes.ts     # Pesapal callbacks
│   │   ├── paystack.routes.ts    # Paystack callbacks
│   │   ├── korapay.routes.ts     # Korapay callbacks
│   │   ├── emergent.routes.ts    # Emergent callbacks
│   │   ├── brooks.routes.ts      # Brooks routing & merchants
│   │   ├── shopify.webhooks.ts   # Shopify order webhooks
│   │   ├── shopify.redirect.ts   # Shopify payment redirect
│   │   ├── admin.shopify.ts      # Shopify tenant admin
│   │   ├── admin-dashboard.routes.ts  # BackOffice dashboards
│   │   ├── backoffice.auth.routes.ts  # BackOffice authentication
│   │   ├── ach.routes.ts         # ACH bank transfers
│   │   └── payment-methods*.ts   # Payment method management
│   │
│   ├── providers/                # Payment provider adapters
│   │   ├── cards/                # Card processors
│   │   │   ├── paystack/paystack.adapter.ts
│   │   │   ├── emergent/emergent.adapter.ts
│   │   │   ├── stripe/stripe.adapter.ts
│   │   │   ├── adyen/adyen.adapter.ts
│   │   │   ├── buni/buni.adapter.ts
│   │   │   └── ...
│   │   ├── mobile-money/         # Mobile money providers
│   │   │   ├── hubtel/hubtel.adapter.ts
│   │   │   ├── buni/buni-mpesa.adapter.ts
│   │   │   ├── buni/buni-mpesa-multi.adapter.ts
│   │   │   ├── mpesa/mpesa.adapter.ts
│   │   │   └── ...
│   │   ├── apms/                 # Alternative payment methods
│   │   │   ├── korapay/korapay.adapter.ts
│   │   │   ├── chapa/chapa.adapter.ts
│   │   │   ├── dpo/dpo.adapter.ts
│   │   │   └── ...
│   │   ├── crypto/               # Cryptocurrency providers
│   │   ├── banks/                # Bank integrations
│   │   ├── open-banking/         # Open banking providers
│   │   ├── pesapal/              # Pesapal adapter + types
│   │   ├── registry.ts           # Provider registry
│   │   ├── base.adapter.ts       # Base adapter class
│   │   └── types.ts              # Shared provider types
│   │
│   ├── shopify/                  # Shopify middleware modules
│   │   ├── shopifyHmac.ts        # HMAC webhook verification
│   │   ├── shopifyAdmin.ts       # Shopify Admin API client
│   │   ├── queue.ts              # BullMQ job queue
│   │   ├── crypto.ts             # AES-256-GCM encryption
│   │   ├── idempotency.ts        # Redis idempotency keys
│   │   ├── rateLimit.ts          # Rate limiting
│   │   ├── redis.ts              # Redis client
│   │   └── authorizeNet.ts       # Authorize.Net integration
│   │
│   ├── brooks/                   # Brooks merchant routing
│   ├── admin/                    # Admin controllers & services
│   ├── auth/                     # Authentication (guards, JWT)
│   ├── backoffice/               # BackOffice auth middleware
│   ├── routing/                  # Payment routing engine
│   ├── limits/                   # Daily limits & rollover
│   ├── merchant/                 # Merchant portal controllers
│   ├── prisma/                   # Prisma client module
│   ├── eligibility/              # Payment eligibility checks
│   ├── vault/                    # Secret vault service
│   └── wallets/                  # Wallet service
│
├── docs/                         # Documentation (for Git)
│   ├── MERCHANT_GUIDE.md
│   ├── WOOCOMMERCE_INTEGRATION.md
│   ├── SHOPIFY_INTEGRATION.md
│   ├── API_REFERENCE.md
│   └── CUSTOMER_FAQ.md
│
└── merchant-packages/            # Per-merchant config packages
    ├── brooks-ap-elements/
    ├── brooks-ip-elements/
    └── brooks-ud-zone/
```

---

## Initial Git Setup (On Server)

Run these commands on the production server (`164.92.213.22`) to initialize the repo:

```bash
cd /var/www/sentinel/services/svc-rails

# Initialize Git repository
git init

# Configure Git identity
git config user.name "SentinelGate Dev"
git config user.email "dev@sentinelgate.biz"

# Ensure .gitignore and .env.example exist (see repo files)

# Stage all tracked files
git add .

# Verify what's being committed (should NOT include .env, node_modules, dist)
git status

# If you see .env or node_modules listed, check .gitignore
# To remove accidentally tracked files:
# git rm --cached .env
# git rm -r --cached node_modules/
# git rm -r --cached dist/

# Initial commit
git commit -m "Initial commit: SentinelGate svc-rails v1.0.0

- 161 TypeScript source files
- 7 live payment providers (BUNI, Emergent, Korapay, Hubtel, Pesapal, Paystack, Brooks)
- WooCommerce PSP plugin integration
- Shopify webhook middleware
- Payment links with QR codes
- BackOffice auth (6 dashboards, PBKDF2+JWT)
- Prisma schema with PostgreSQL
- BullMQ async queue with Redis"

# Add remote (replace with your actual repo URL)
git remote add origin git@github.com:YOUR_ORG/sentinelgate-svc-rails.git

# Push
git push -u origin main
```

### What Gets Committed (and What Doesn't)

| Included | Excluded |
|----------|----------|
| `src/` (all TypeScript source) | `node_modules/` (287MB, installed via npm) |
| `prisma/schema.prisma` | `.env` (contains secrets) |
| `prisma/migrations/` | `.env.bak*` (backup secrets) |
| `prisma/seed.ts` | `dist/` (built from src) |
| `scripts/` | `*.tar.gz`, `*.zip` (archives) |
| `package.json` + lock file | `backups/` |
| `tsconfig.json` | Credential files (`*.json` except package/tsconfig) |
| `.env.example` | `.turbo/` |
| `docs/` | Old doc packages |
| `Dockerfile` | |
| `merchant-packages/` | |

---

## Cloning & Local Development

### Prerequisites

| Software | Version | Install |
|----------|---------|---------|
| Node.js | 18+ | `nvm install 18` |
| npm | 9+ | Comes with Node.js |
| PostgreSQL | 14+ | `apt install postgresql` |
| Redis | 7+ | `apt install redis-server` |
| Git | 2.30+ | `apt install git` |

### Clone the Repository

```bash
git clone git@github.com:YOUR_ORG/sentinelgate-svc-rails.git
cd sentinelgate-svc-rails
```

### Install Dependencies

```bash
npm install
```

This installs all packages from `package-lock.json` (~287MB).

### Generate Prisma Client

```bash
npx prisma generate
```

This creates the Prisma client from `prisma/schema.prisma`. Required before building.

---

## Environment Configuration

```bash
# Copy the template
cp .env.example .env

# Edit with your values
nano .env
```

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `PORT` | Server port | `3003` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://sentinel:pass@localhost:5432/sentinel` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` |
| `ADMIN_API_KEY` | Admin endpoint auth key | `sg_admin_xxxx` |
| `API_BASE_URL` | Public URL of this service | `https://sentinelgate.biz` |

### Provider Credentials

Each payment provider needs its own set of credentials. See `.env.example` for the full list. At minimum, configure one provider (e.g., Hubtel) to process payments.

### Security Notes

- **Never commit `.env`** — it contains API keys, database passwords, and secrets
- **Never log credentials** — mask sensitive values in log output
- **Rotate keys regularly** — especially after team member changes
- **Use environment variables** — not hardcoded values in source code

---

## Database Setup

### Create the Database

```bash
# As postgres user
sudo -u postgres psql

CREATE USER sentinel WITH PASSWORD 'YOUR_SECURE_PASSWORD';
CREATE DATABASE sentinel OWNER sentinel;
GRANT ALL PRIVILEGES ON DATABASE sentinel TO sentinel;
\q
```

### Run Migrations

```bash
# Apply all pending migrations
npx prisma migrate deploy

# Verify tables were created
npx prisma db pull
```

### Seed Initial Data

```bash
# Seed providers, merchants, and test data
npx prisma db seed
```

### Key Database Models

| Model | Purpose |
|-------|---------|
| `Merchant` | Registered merchants |
| `MerchantApiKey` | API credentials per merchant |
| `ProviderConfig` | Payment provider settings |
| `Transaction` | Payment transaction records |
| `PaymentLink` | Shareable payment links |
| `PaymentLinkEvent` | Payment link activity log |
| `ShopifyTenantConfig` | Shopify store configurations |
| `ShopifyCaptureLog` | Shopify payment capture logs |
| `BackofficeUser` | Admin portal users |

---

## Building & Running

### Build TypeScript

```bash
# Compile TypeScript to JavaScript
npm run build
# Output goes to dist/
```

### Run Locally (Development)

```bash
# With auto-restart on file changes
npm run start:dev
```

### Run in Production

```bash
# Using compiled output
npm start
# Or directly:
node dist/main.js
```

### Verify the Server

```bash
# Health check
curl http://localhost:3003/health

# Should return: {"status":"ok","providers":7}
```

---

## Deployment to Production

### Server Details

| Item | Value |
|------|-------|
| **IP** | `164.92.213.22` |
| **OS** | Ubuntu 24 |
| **Path** | `/var/www/sentinel/services/svc-rails` |
| **Process Manager** | PM2 |
| **Reverse Proxy** | Apache |
| **Domain** | `sentinelgate.biz` |
| **SSL** | Let's Encrypt (auto-renew) |
| **Database** | PostgreSQL 16 on `localhost:5432` |
| **Redis** | Redis 7 on `localhost:6379` |

### Deploy a New Version

```bash
# SSH into server
ssh root@164.92.213.22

# Go to project directory
cd /var/www/sentinel/services/svc-rails

# Pull latest code
git pull origin main

# Install any new dependencies
npm install

# Generate Prisma client (if schema changed)
npx prisma generate

# Run migrations (if any new ones)
npx prisma migrate deploy

# Build TypeScript
npm run build

# Restart the service
pm2 restart sentinel-svc-rails --update-env

# Verify it's running
pm2 status
curl -s http://localhost:3003/health
```

### Quick Deploy (No Schema Changes)

```bash
cd /var/www/sentinel/services/svc-rails
git pull origin main
npm run build
pm2 restart sentinel-svc-rails --update-env
```

---

## PM2 Process Management

### Current PM2 Services

| ID | Name | Port | Description |
|----|------|------|-------------|
| 1 | `sentinel-svc-rails` | 3003 | Core payment service |
| 5 | `sentinel-api-gateway` | 3000 | API gateway |
| 2 | `sentinel-web-merchant` | — | Merchant portal |
| 3 | `sentinel-web-admin` | — | Admin portal |
| 4 | `sentinel-web-gamer` | — | Gamer portal |

### Common Commands

```bash
# View all processes
pm2 status

# Restart a specific service
pm2 restart sentinel-svc-rails --update-env

# View logs (live)
pm2 logs sentinel-svc-rails

# View last 50 lines
pm2 logs sentinel-svc-rails --lines 50 --nostream

# Stop a service
pm2 stop sentinel-svc-rails

# Delete and re-add
pm2 delete sentinel-svc-rails
pm2 start dist/main.js --name sentinel-svc-rails
pm2 save
```

### PM2 Startup (Auto-Start on Reboot)

```bash
pm2 startup
pm2 save
```

---

## Apache Reverse Proxy

### Active Configuration File

**IMPORTANT:** The active config is `sentinelgate.biz.conf`, NOT the `-le-ssl.conf` file.

```
/etc/apache2/sites-available/sentinelgate.biz.conf
```

### Key Proxy Rules

| Path | Backend | Service |
|------|---------|---------|
| `/v1/` | `http://localhost:3003/v1/` | PSP endpoints |
| `/webhooks/` | `http://localhost:3003/webhooks/` | Shopify webhooks |
| `/buni/` | `http://localhost:3003/buni/` | BUNI M-Pesa callbacks |
| `/hubtel/` | `http://localhost:3003/hubtel/` | Hubtel callbacks |
| `/pesapal/` | `http://localhost:3003/pesapal/` | Pesapal callbacks |
| `/brooks/` | `http://localhost:3003/brooks/` | Brooks routing |
| `/pay/` | `http://localhost:3003/pay/` | Payment links & Shopify redirect |
| `/backoffice/` | `http://localhost:3003/backoffice/` | BackOffice portal |

### Testing Apache Config

```bash
apache2ctl configtest
systemctl reload apache2
```

---

## SSL Certificates

SSL is managed by Let's Encrypt via Certbot:

```bash
# Check certificate status
certbot certificates

# Renew all certificates
certbot renew

# Auto-renewal is configured via systemd timer
systemctl status certbot.timer
```

---

## Git Workflow

### Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Production — deployed to `164.92.213.22` |
| `develop` | Integration branch for testing |
| `feature/*` | Feature branches (e.g., `feature/stripe-adapter`) |
| `hotfix/*` | Urgent production fixes |

### Making Changes

```bash
# Create a feature branch
git checkout -b feature/new-provider

# Make changes
# ...edit files...

# Stage and commit
git add -A
git commit -m "feat: add new payment provider adapter"

# Push to remote
git push origin feature/new-provider

# Create a Pull Request on GitHub
# After review, merge to main
# Deploy to production (see Deployment section)
```

### Commit Message Convention

```
type: short description

Types:
  feat:     New feature
  fix:      Bug fix
  docs:     Documentation
  refactor: Code refactoring
  chore:    Maintenance tasks
  hotfix:   Urgent production fix

Examples:
  feat: add Emergent card processing adapter
  fix: resolve Hubtel callback double-processing
  docs: update WooCommerce integration guide
  hotfix: fix payment link amount calculation
```

---

## Troubleshooting

### Service Won't Start

```bash
# Check PM2 logs for errors
pm2 logs sentinel-svc-rails --lines 30 --nostream

# Common issues:
# - Missing .env variables → copy from .env.example
# - Database not reachable → check PostgreSQL is running
# - Redis not reachable → check Redis is running
# - Port 3003 in use → kill the other process
# - TypeScript build errors → run npm run build and fix errors
```

### TypeScript Build Errors

```bash
# Build and see all errors
npx tsc --project tsconfig.json 2>&1

# Current known non-blocking errors (safe to ignore):
# - paystack.adapter.ts: Property 'createPayment' does not exist
# - pesapal.adapter.ts: Property 'createPayment' does not exist
# - registry.ts: Type missing properties
# - brooks.routes.ts: Argument type mismatch
# These don't affect runtime because tsconfig has strict=false
```

### Database Connection Failed

```bash
# Check PostgreSQL is running
systemctl status postgresql

# Test connection
psql "postgresql://sentinel:YOUR_PASSWORD@localhost:5432/sentinel" -c "SELECT 1;"

# Check .env DATABASE_URL is correct
grep DATABASE_URL .env
```

### Redis Connection Failed

```bash
# Check Redis is running
systemctl status redis-server

# Test connection
redis-cli ping
# Should return: PONG
```

### Webhooks Not Arriving

```bash
# Check Apache proxy is forwarding correctly
curl -s http://localhost:3003/health

# Check Apache config
grep -r "ProxyPass" /etc/apache2/sites-available/sentinelgate.biz.conf

# Check firewall
ufw status

# Test webhook endpoint directly
curl -X POST http://localhost:3003/webhooks/shopify/order-created \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

### Viewing Live Transactions

```bash
# Watch PM2 logs in real time
pm2 logs sentinel-svc-rails

# Filter for specific provider
pm2 logs sentinel-svc-rails | grep "Hubtel\|Paystack\|BUNI"

# Check database for recent transactions
psql "postgresql://sentinel:YOUR_PASSWORD@localhost:5432/sentinel" \
  -c "SELECT id, status, amount, currency, created_at FROM transactions ORDER BY created_at DESC LIMIT 10;"
```

---

## Team Access

### SSH Access

| User | Host | Purpose |
|------|------|---------|
| `root` | `164.92.213.22` | Full server access |
| `ivy` | `164.92.213.22` | Developer (chroot to sixty40.site) |

### Adding a New Developer

1. Create their SSH key pair
2. Add public key to `~/.ssh/authorized_keys` on the server
3. If they need restricted access, set up a chroot jail (see Ivy's config as reference)
4. Give them read access to the Git repository
5. Share `.env.example` — never share the actual `.env`

---

## Quick Reference

```bash
# Deploy
cd /var/www/sentinel/services/svc-rails && git pull && npm run build && pm2 restart sentinel-svc-rails

# Logs
pm2 logs sentinel-svc-rails --lines 50 --nostream

# Status
pm2 status && curl -s http://localhost:3003/health

# Database
psql "postgresql://sentinel:Sentinel%40DB_2026@localhost:5432/sentinel"

# Restart everything
pm2 restart all
```
