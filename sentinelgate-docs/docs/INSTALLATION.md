# Installation Guide

Self-hosted deployment instructions for the SentinelGate platform.

---

## System Requirements

| Component | Minimum |
|----------|---------|
| OS | Ubuntu 22.04 / 24.04 LTS |
| CPU | 2 vCPUs |
| RAM | 4 GB |
| Disk | 40 GB SSD |
| Node.js | v18.x LTS |
| PostgreSQL | 14+ |
| Redis | 6+ |
| Apache | 2.4+ (reverse proxy) |

---

## Architecture

```
Internet → Apache (443) → API Gateway (3000) → Core Service (3003)
                        → Admin Console (3100)
                        → Merchant Portal (3200)
```

| Service | Port | Runtime |
|---------|------|---------|
| API Gateway | 3000 | Node.js / Express |
| Core Service (svc-rails) | 3003 | Node.js / Express / Prisma |
| Admin Console (web-admin) | 3100 | Next.js |
| Merchant Portal (web-merchant) | 3200 | Next.js |

All services are managed by **PM2** for process management and auto-restart.

---

## Step 1: Install Dependencies

```bash
# System packages
apt update && apt install -y nodejs npm postgresql redis-server apache2

# Node.js 18 (if not already)
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt install -y nodejs

# PM2
npm install -g pm2

# Build tool
npm install -g esbuild
```

---

## Step 2: Database Setup

```bash
sudo -u postgres psql

CREATE DATABASE sentinel;
CREATE USER sentinel_user WITH ENCRYPTED PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE sentinel TO sentinel_user;
\q
```

---

## Step 3: Deploy Services

```bash
# Clone the repository
cd /var/www/sentinel/services

# Install dependencies for each service
cd svc-rails && npm install
cd ../api-gateway && npm install
cd ../web-admin && npm install
cd ../web-merchant && npm install

# Generate Prisma client
cd /var/www/sentinel/services/svc-rails
npx prisma generate
npx prisma migrate deploy

# Build core service
npx esbuild src/main.ts --bundle --platform=node \
  --outfile=dist/main.js \
  --external:@prisma/client --external:prisma --external:bcrypt

# Build Next.js apps
cd /var/www/sentinel/services/web-admin && npx next build
cd /var/www/sentinel/services/web-merchant && npx next build
```

---

## Step 4: Environment Configuration

Create `/var/www/sentinel/services/svc-rails/.env`:

```env
DATABASE_URL=postgresql://sentinel_user:your_password@localhost:5432/sentinel
REDIS_URL=redis://localhost:6379
PSP_ENCRYPTION_KEY_BASE64=your_base64_key
ADMIN_API_KEY=your_admin_key
NODE_ENV=production
PORT=3003
```

---

## Step 5: PM2 Process Management

```bash
pm2 start dist/main.js --name sentinel-svc-rails --cwd /var/www/sentinel/services/svc-rails
pm2 start npm --name sentinel-api-gateway --cwd /var/www/sentinel/services/api-gateway -- start
pm2 start npm --name sentinel-web-admin --cwd /var/www/sentinel/services/web-admin -- start
pm2 start npm --name sentinel-web-merchant --cwd /var/www/sentinel/services/web-merchant -- start

pm2 save
pm2 startup
```

---

## Step 6: Apache Reverse Proxy

Enable required modules:

```bash
a2enmod proxy proxy_http proxy_wstunnel ssl rewrite headers
```

Configure your virtual host (e.g., `/etc/apache2/sites-available/sentinelgate.biz.conf`):

```apache
<VirtualHost *:443>
    ServerName sentinelgate.biz
    
    SSLEngine on
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/key.pem

    # API Gateway
    ProxyPass /v1/ http://localhost:3000/v1/
    ProxyPassReverse /v1/ http://localhost:3000/v1/

    # Webhooks
    ProxyPass /webhooks/ http://localhost:3003/webhooks/
    ProxyPassReverse /webhooks/ http://localhost:3003/webhooks/

    # Payment links
    ProxyPass /pay/ http://localhost:3003/pay/
    ProxyPassReverse /pay/ http://localhost:3003/pay/

    # Admin console
    ProxyPass /console/admin http://localhost:3100/console/admin
    ProxyPassReverse /console/admin http://localhost:3100/console/admin
</VirtualHost>
```

```bash
a2ensite sentinelgate.biz.conf
systemctl reload apache2
```

---

## Step 7: SSL Certificate

```bash
apt install certbot python3-certbot-apache
certbot --apache -d sentinelgate.biz
```

---

## Verification

```bash
# Check all services
pm2 list

# Test core service
curl -s http://localhost:3003/v1/transaction/test | python3 -m json.tool

# Test from outside
curl -s https://sentinelgate.biz/v1/transaction/test | python3 -m json.tool
```

---

## Maintenance

```bash
# View logs
pm2 logs sentinel-svc-rails

# Restart a service
pm2 restart sentinel-svc-rails

# Rebuild after code changes
cd /var/www/sentinel/services/svc-rails
npx esbuild src/main.ts --bundle --platform=node --outfile=dist/main.js \
  --external:@prisma/client --external:prisma --external:bcrypt
pm2 restart sentinel-svc-rails
```

---

© 2026 SentinelGate — Whyte AG Group
