# Troubleshooting Guide — Common Issues

**For:** Developers & Technical Teams
**Last Updated:** February 24, 2026

---

## Authentication Issues

### ❌ Error: "Invalid API Key" or 401 Unauthorized

**Symptoms:**
```json
{
  "error": "MISSING_API_KEY",
  "message": "API key is missing or invalid"
}
```

**Solutions:**

1. **Check credential format** — SentinelGate credentials use the `sg_` prefix:
   ```
   API Key:        sg_key_yourstore_abc123...
   API Secret:     sg_secret_yourstore_def456...
   Webhook Secret: sg_whsec_yourstore_ghi789...
   ```

2. **Check headers** — must be `X-API-Key` and `X-API-Secret`, not `Authorization`:
   ```javascript
   // ✅ Correct
   headers: {
     'X-API-Key': 'sg_key_yourstore_abc123',
     'X-API-Secret': 'sg_secret_yourstore_def456',
     'Content-Type': 'application/json'
   }

   // ❌ Wrong — not Bearer auth
   headers: {
     'Authorization': 'Bearer sg_key_...'
   }

   // ❌ Wrong — lowercase header names may fail
   headers: {
     'x-api-key': 'sg_key_...'
   }
   ```

3. **Verify credentials are loaded:**
   ```bash
   # Check environment variables
   echo "Key starts with: ${SENTINELGATE_API_KEY:0:20}..."
   echo "Secret starts with: ${SENTINELGATE_API_SECRET:0:25}..."
   ```

4. **Check for whitespace** — copy-paste can add leading/trailing spaces:
   ```javascript
   const key = process.env.SENTINELGATE_API_KEY.trim();
   ```

---

## Payment Creation Issues

### ❌ No `redirect_url` in response

**Symptoms:** The `/v1/hosted/create` call returns successfully but the redirect URL is empty or the customer isn't redirected.

**Solutions:**

1. **Check required fields:**
   ```json
   {
     "amount": "50.00",        // Required — string, decimal format
     "currency": "USD",         // Required — ISO 4217
     "order_id": "ORD-001",    // Required — your unique order ID
     "callback_url": "https://yoursite.com/webhook",  // Required — HTTPS
     "return_url": "https://yoursite.com/confirmed"    // Required — HTTPS
   }
   ```

2. **Amount must be a string in dollars**, not cents:
   ```javascript
   // ✅ Correct
   { "amount": "191.00" }

   // ❌ Wrong — integer
   { "amount": 191 }

   // ❌ Wrong — cents
   { "amount": "19100" }
   ```

3. **Check provider is configured** — if no provider is available for the currency/method, session creation fails. Verify with:
   ```bash
   curl -s https://sentinelgate.biz/health
   # Should show {"status":"ok","providers":7}
   ```

### ❌ Customer redirected to wrong provider / sees wrong merchant name

SentinelGate routes payments based on merchant configuration. If the customer sees an unexpected provider page or merchant name:

1. The payment is being routed to the correct provider — the merchant name shown is from the **provider dashboard**, not SentinelGate
2. To change the displayed name: update the business name in the provider's dashboard (e.g., Hubtel → Business Settings)
3. If routing to the wrong provider entirely, contact SentinelGate to check your merchant routing config

---

## Webhook Issues

### ❌ Webhooks not being received

**Step-by-step diagnosis:**

1. **Is your endpoint publicly accessible?**
   ```bash
   curl -X POST https://yoursite.com/webhook/sentinelgate \
     -H "Content-Type: application/json" \
     -d '{"test": true}'
   # Should NOT return 404 or connection refused
   ```

2. **Is HTTPS working?**
   ```bash
   curl -v https://yoursite.com 2>&1 | grep "SSL certificate"
   # Must have valid SSL — self-signed certificates are rejected
   ```

3. **Is a firewall or WAF blocking POST requests?**
   ```bash
   # Check if Cloudflare, Sucuri, or Wordfence is blocking
   # Whitelist SentinelGate server IP: 164.92.213.22
   ```

4. **Is your webhook responding fast enough?**
   SentinelGate expects a 2xx response within **15 seconds**. If your handler takes longer, the webhook is marked as failed and retried.

   ```javascript
   // ❌ Wrong — processes before responding (may timeout)
   app.post('/webhook', async (req, res) => {
     await processPayment(req.body);  // Takes 20 seconds
     res.status(200).send('OK');       // Too late
   });

   // ✅ Correct — respond first, process async
   app.post('/webhook', (req, res) => {
     res.status(200).send('OK');       // Immediate response
     processPayment(req.body).catch(console.error);  // Async
   });
   ```

5. **WooCommerce specific:** The webhook URL is `https://yoursite.com/wc-api/sentinelgate_callback/`. If you get 404:
   ```
   Go to WordPress → Settings → Permalinks → click Save (re-flushes rewrite rules)
   ```

### ❌ Error: "Invalid webhook signature"

```javascript
// ✅ CRITICAL: You must use the raw body for signature verification

// ❌ Wrong — body is already parsed by express.json()
app.use(express.json());
app.post('/webhook', (req, res) => {
  verifySignature(JSON.stringify(req.body), ...);  // WRONG — re-serialized body ≠ raw body
});

// ✅ Correct — use express.raw() to get the original bytes
app.post('/webhook',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const isValid = verifySignature(req.body, ...);  // req.body is Buffer
  }
);
```

**Signature header:** `X-Sentinel-Signature` (not `X-Webhook-Signature`)

**Format:** `sha256=<hex_digest>`

```javascript
function verifySignature(rawBody, signature, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );
}
```

### ❌ Duplicate webhook events

SentinelGate may retry webhooks if your server returns a non-2xx response. This means you might receive the same event 2-3 times.

**Solution:** Use `sentinel_transaction_id` as an idempotency key:

```javascript
async function processWebhook(event) {
  const existing = await db.payments.findFirst({
    where: { sentinelTxnId: event.sentinel_transaction_id }
  });

  if (existing && existing.status === 'paid') {
    console.log('Already processed, skipping');
    return;
  }

  // Process payment...
}
```

---

## Provider-Specific Issues

### ❌ Hubtel: "Sixty Forty" shown as merchant name

Hubtel displays the merchant name from their dashboard, not from SentinelGate. To fix:

1. Log in to your Hubtel merchant dashboard
2. Go to Business Settings
3. Update the business name
4. Changes take effect on the next checkout session

### ❌ Hubtel: USD to GHS conversion

This is expected behavior. Hubtel operates in Ghana Cedis (GHS). When you send a USD amount, Hubtel converts at their current exchange rate. The customer sees the GHS amount on the checkout page.

You cannot control the exchange rate. If exact USD amounts matter, use a provider that supports USD natively (Emergent, Paystack, Pesapal — when activated).

### ❌ Emergent: 500 Internal Server Error

```
HTTP 500 — IIS Configuration Error
Server: api.interpayafrica.com
```

This is a server-side issue on Emergent's infrastructure (ASP.NET web.config error, line 136). SentinelGate credentials may be valid but cannot be verified until their server is fixed. This requires escalation to InterPay Africa's infrastructure team — it is not fixable from SentinelGate's side.

### ❌ Paystack: "Invalid key" on transaction endpoints

Paystack keys work for read endpoints (`/bank`, `/resolve`) but return "Invalid key" on transaction endpoints (`/transaction/initialize`, `/charge`).

**Cause:** Paystack account business verification is incomplete.

**Fix:** Complete business verification on [dashboard.paystack.co](https://dashboard.paystack.co) → Settings → Business → Submit all required documents.

### ❌ Pesapal: "invalid_consumer_key_or_secret_provided"

The consumer key/secret pair is rejected by Pesapal's API.

**Fix:**
1. Log in to [pay.pesapal.com](https://pay.pesapal.com)
2. Go to API Keys section
3. Generate new live keys
4. Share updated keys with SentinelGate

### ❌ BUNI M-Pesa: STK Push not received by customer

1. Verify the phone number is in correct format: `254712345678` (no +, no spaces)
2. Check if BUNI is on UAT or production gateway (currently UAT)
3. STK push requires the phone to be on Safaricom network
4. Customer's phone must have M-Pesa activated and sufficient balance

---

## WooCommerce Plugin Issues

### ❌ "Transaction Declined (Sandbox Mode)" error

This is NOT from SentinelGate. Another payment gateway (often the default WooCommerce one) is active and in sandbox mode.

**Fix:** Go to WooCommerce → Settings → Payments → disable all gateways except SentinelGate PSP.

### ❌ Order stays in "Pending Payment" after customer pays

The webhook isn't reaching WooCommerce. Check:

1. Is the endpoint accessible? Visit `https://yourstore.com/wc-api/sentinelgate_callback/` — should not 404
2. Flush rewrite rules: Settings → Permalinks → Save
3. Check if a security plugin (Wordfence, Sucuri, iThemes) is blocking POST requests
4. Check WooCommerce logs: WooCommerce → Status → Logs → `sentinelgate-*`

### ❌ Plugin settings not saving

1. Clear any page cache (WP Super Cache, W3 Total Cache, LiteSpeed Cache)
2. Try in a private/incognito browser window
3. Check PHP error logs: `wp-content/debug.log` (enable with `WP_DEBUG=true` in `wp-config.php`)

---

## Shopify Integration Issues

### ❌ Webhooks not being received from Shopify

1. Verify webhook URLs in Shopify Admin → Settings → Notifications → Webhooks
2. Check if `https://sentinelgate.biz` is reachable
3. Verify the HMAC secret matches what SentinelGate has on file
4. Check Shopify webhook delivery log for error codes

### ❌ Customer not redirected after Shopify checkout

1. Check that the Additional Scripts are installed: Settings → Checkout → Order status page
2. Open browser developer console (F12) and check for JavaScript errors
3. The redirect script runs on the **order status page**, not the checkout page
4. Test in incognito — browser extensions can interfere

### ❌ Order not marked as paid in Shopify

1. Verify the Shopify Admin API token has `write_orders` scope
2. Check if the token has been revoked or expired
3. Generate a new token: Settings → Apps → Your App → API Credentials

---

## Connection Issues

### ❌ Connection timeout

```javascript
// Set a reasonable timeout (30 seconds for payment creation)
const response = await axios.post(url, data, {
  timeout: 30000
});
```

If timeouts persist:
1. Check if your server can reach `sentinelgate.biz`:
   ```bash
   curl -v https://sentinelgate.biz/health
   ```
2. Check DNS resolution: `nslookup sentinelgate.biz`
3. Check if your hosting provider blocks outbound HTTPS
4. Check if a corporate firewall is blocking the connection

### ❌ Rate limit exceeded (HTTP 429)

| Endpoint | Limit |
|----------|-------|
| `/v1/hosted/create` | 60/min |
| `/v1/charge` | 30/min |
| `/v1/transaction/:id` | 120/min |
| `/v1/refund` | 10/min |

**Solution:** Implement exponential backoff:

```javascript
async function callWithBackoff(fn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (error.response?.status !== 429 || attempt === maxRetries) throw error;
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

---

## Server Administration Issues

### ❌ svc-rails won't start

```bash
# Check PM2 logs
pm2 logs sentinel-svc-rails --lines 30 --nostream

# Common causes:
# 1. Missing .env → copy from .env.example and fill in values
# 2. PostgreSQL not running → systemctl start postgresql
# 3. Redis not running → systemctl start redis-server
# 4. Port 3003 in use → lsof -i :3003
# 5. Build errors → npm run build (check output for errors)
```

### ❌ TypeScript build errors

```bash
npx tsc --project tsconfig.json 2>&1 | head -30
```

Known non-blocking errors (safe to ignore because `tsconfig.json` has `strict: false`):
- `paystack.adapter.ts`: Property 'createPayment' does not exist
- `pesapal.adapter.ts`: Property 'createPayment' does not exist
- `registry.ts`: Type missing properties
- `brooks.routes.ts`: Argument type mismatch

These are type annotation issues that don't affect runtime behavior.

### ❌ Database connection failed

```bash
# Check PostgreSQL is running
systemctl status postgresql

# Test connection
psql "postgresql://sentinel:YOUR_PASSWORD@localhost:5432/sentinel" -c "SELECT 1;"

# Check the DATABASE_URL in .env matches
grep DATABASE_URL .env

# Common fix: restart PostgreSQL
systemctl restart postgresql
```

### ❌ Redis connection failed

```bash
# Check Redis is running
systemctl status redis-server

# Test
redis-cli ping
# Expected: PONG

# Common fix: restart Redis
systemctl restart redis-server
```

### ❌ Apache not proxying correctly

```bash
# Verify Apache config is correct
apache2ctl configtest

# Check the ACTIVE config file (NOT the -le-ssl.conf)
grep "ProxyPass" /etc/apache2/sites-available/sentinelgate.biz.conf

# Reload after changes
systemctl reload apache2

# Test direct backend access (bypassing Apache)
curl -s http://localhost:3003/health
```

---

## Debugging Checklist

When something doesn't work, run through this in order:

1. **Is the service running?**
   ```bash
   pm2 status
   curl -s https://sentinelgate.biz/health
   ```

2. **Are credentials correct?**
   ```bash
   curl -X POST https://sentinelgate.biz/v1/hosted/create \
     -H "X-API-Key: $SENTINELGATE_API_KEY" \
     -H "X-API-Secret: $SENTINELGATE_API_SECRET" \
     -H "Content-Type: application/json" \
     -d '{"amount":"1.00","currency":"USD","order_id":"debug-001","callback_url":"https://yoursite.com/webhook","return_url":"https://yoursite.com"}'
   ```

3. **Check PM2 logs:**
   ```bash
   pm2 logs sentinel-svc-rails --lines 50 --nostream | grep -i "error\|fail\|warn"
   ```

4. **Check the request you're sending:**
   ```javascript
   console.log('Request:', JSON.stringify({
     url, headers: { 'X-API-Key': key.substring(0, 20) + '...' }, body
   }, null, 2));
   ```

5. **Check the response:**
   ```javascript
   try {
     const result = await axios.post(url, data, { headers });
     console.log('Response:', result.status, result.data);
   } catch (error) {
     console.error('Error:', error.response?.status, error.response?.data);
   }
   ```

---

## Getting Help

**Include in your support request:**
- Transaction ID (e.g., `sg_txn_1771888643979_cfe07b9d7fe6`)
- Error message or HTTP status code
- Request payload (remove credentials!)
- Timestamp of the failed request
- Provider if known (Hubtel, BUNI, etc.)

**Contact:** support@sentinelgate.biz

---

*© 2026 SentinelGate. All rights reserved.*
