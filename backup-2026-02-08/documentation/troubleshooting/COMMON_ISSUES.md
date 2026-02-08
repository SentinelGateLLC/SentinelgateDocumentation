# Troubleshooting Guide

Common issues and their solutions when integrating with SentinelGate.

---

## Authentication Issues

### ❌ Error: "Invalid API Key"

**Symptoms:**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_API_KEY",
    "message": "API key is missing or invalid"
  }
}
```

**Solutions:**

1. **Verify credentials format:**
   - API Key should start with `sk_live_`
   - API Secret should start with `secret_`

2. **Check headers:**
```javascript
   // ✅ Correct
   headers: {
     'X-API-Key': 'sk_live_...',
     'X-API-Secret': 'secret_...'
   }
   
   // ❌ Wrong
   headers: {
     'Authorization': 'Bearer sk_live_...'  // Wrong!
   }
```

3. **Environment variables:**
```bash
   # Check they're loaded
   echo $SENTINELGATE_API_KEY
```

---

## Payment Creation Issues

### ❌ Error: "Invalid phone number"

**Symptoms:**
```json
{
  "error": {
    "code": "INVALID_PHONE",
    "message": "Phone number must be in format 254XXXXXXXXX"
  }
}
```

**Solutions:**
```javascript
// ❌ Wrong formats
'0712345678'        // Missing country code
'+254712345678'     // Has + sign
'254 712 345 678'   // Has spaces
'254-712-345-678'   // Has dashes

// ✅ Correct format
'254712345678'      // Exactly this format
```

**Phone number validator:**
```javascript
function validateKenyanPhone(phone) {
  // Remove any spaces, dashes, plus signs
  phone = phone.replace(/[\s\-\+]/g, '');
  
  // Convert 0712... to 254712...
  if (phone.startsWith('0')) {
    phone = '254' + phone.substring(1);
  }
  
  // Validate format
  if (!/^254\d{9}$/.test(phone)) {
    throw new Error('Invalid phone format. Must be 254XXXXXXXXX');
  }
  
  return phone;
}
```

---

### ❌ Error: "Amount below minimum"

**Symptoms:**
```json
{
  "error": {
    "code": "INVALID_AMOUNT",
    "message": "Amount must be at least 10 KES"
  }
}
```

**Solutions:**
```javascript
// ❌ Wrong - amount in cents
amount_cents: 5  // 0.05 KES - too small!

// ✅ Correct - minimum 10 KES = 1000 cents
amount_cents: 1000  // 10 KES

// ✅ Better - convert from KES
function toAmountCents(amountKES) {
  const cents = Math.round(amountKES * 100);
  
  if (cents < 1000) {
    throw new Error('Minimum amount is 10 KES');
  }
  
  if (cents > 15000000) {  // 150,000 KES for M-Pesa
    throw new Error('Maximum amount is 150,000 KES');
  }
  
  return cents;
}
```

---

## Webhook Issues

### ❌ Webhooks not being received

**Diagnosis:**

1. **Check webhook URL is accessible:**
```bash
   curl -X POST https://yourdomain.com/webhook \
     -H "Content-Type: application/json" \
     -d '{"test": "data"}'
```

2. **Verify SSL certificate:**
```bash
   curl -v https://yourdomain.com/webhook 2>&1 | grep -i ssl
```

3. **Check server logs:**
```bash
   tail -f /var/log/app/webhooks.log
```

**Common causes:**

✅ **Solution 1: Firewall blocking**
```bash
# Allow incoming connections on webhook port
sudo ufw allow 443/tcp
```

✅ **Solution 2: Wrong content-type parser**
```javascript
// ❌ Wrong - parses body before verification
app.use(express.json());
app.post('/webhook', (req, res) => {
  // req.body is already parsed - can't verify signature!
});

// ✅ Correct - use raw body for webhook
app.post('/webhook',
  express.raw({type: 'application/json'}),
  (req, res) => {
    // req.body is Buffer - can verify signature
    const signature = req.headers['x-webhook-signature'];
    // ...verify signature first
    const event = JSON.parse(req.body);
  }
);
```

✅ **Solution 3: Not returning 200 OK**
```javascript
// ❌ Wrong - no response sent
app.post('/webhook', async (req, res) => {
  await processEvent(req.body);
  // Forgot to send response!
});

// ✅ Correct - always respond quickly
app.post('/webhook', (req, res) => {
  // Send 200 immediately
  res.status(200).send('OK');
  
  // Process async
  processEvent(req.body).catch(console.error);
});
```

---

### ❌ Error: "Invalid webhook signature"

**Solutions:**
```javascript
// ✅ Correct signature verification
const crypto = require('crypto');

function verifyWebhook(rawBody, signature, secret) {
  // rawBody must be Buffer or string (NOT parsed JSON)
  const hmac = crypto.createHmac('sha256', secret);
  const digest = hmac.update(rawBody).digest('hex');
  
  // Use timing-safe comparison
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(digest)
  );
}

// Usage with Express
app.post('/webhook',
  express.raw({type: 'application/json'}),  // Keep as Buffer
  (req, res) => {
    const signature = req.headers['x-webhook-signature'];
    
    if (!verifyWebhook(req.body, signature, process.env.WEBHOOK_SECRET)) {
      return res.status(401).send('Invalid signature');
    }
    
    // Now parse the body
    const event = JSON.parse(req.body);
    // ... process event
  }
);
```

---

## Connection Issues

### ❌ Error: "Connection timeout"

**Solutions:**

1. **Increase timeout:**
```javascript
   const response = await axios.post(url, data, {
     timeout: 30000  // 30 seconds instead of default 5s
   });
```

2. **Implement retry logic:**
```javascript
   async function createPaymentWithRetry(payload, maxRetries = 3) {
     for (let attempt = 1; attempt <= maxRetries; attempt++) {
       try {
         return await createPayment(payload);
       } catch (error) {
         if (attempt === maxRetries) throw error;
         
         // Exponential backoff
         const delay = Math.pow(2, attempt) * 1000;
         await new Promise(resolve => setTimeout(resolve, delay));
         
         console.log(`Retry attempt ${attempt + 1}...`);
       }
     }
   }
```

---

### ❌ Error: "Rate limit exceeded"

**Symptoms:**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later."
  }
}
```

**Solutions:**
```javascript
// Implement rate limiting on client side
const rateLimit = require('bottleneck');

const limiter = new rateLimit({
  maxConcurrent: 5,        // Max 5 concurrent requests
  minTime: 100             // Min 100ms between requests
});

async function createPayment(data) {
  return limiter.schedule(() => 
    sentinelgate.createPayment(data)
  );
}
```

---

## Testing Issues

### ❌ Test phone not working

**Test numbers:**
```javascript
// Kenya M-Pesa test numbers
const TEST_NUMBERS = {
  // Sandbox environment
  success: '254712345678',      // Always succeeds immediately
  fail: '254700000000',         // Always fails
  timeout: '254711111111',      // Times out (stays pending)
  cancel: '254722222222'        // User cancels
};

// How to use
const result = await createMpesaPayment(
  1000,
  TEST_NUMBERS.success,  // Will succeed
  'Test payment'
);
```

---

## Production Issues

### ❌ Payments work in test but fail in production

**Checklist:**

- [ ] Using production API credentials (not test)
- [ ] Updated webhook URL to production domain
- [ ] SSL certificate is valid
- [ ] Production server can reach SentinelGate API
- [ ] Firewall allows outbound HTTPS connections
- [ ] Using real phone numbers (not test numbers)

**Verify production setup:**
```bash
# Test API connection
curl -X POST http://185.229.224.244:3000/api/payments/create \
  -H "X-API-Key: $PROD_API_KEY" \
  -H "X-API-Secret: $PROD_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_cents": 1000,
    "currency": "KES",
    "provider": "MPESA_SAFARICOM",
    "metadata": {
      "phone_number": "254712345678",
      "description": "Production test"
    },
    "callback_url": "https://yourdomain.com/webhook"
  }'
```

---

## Debugging Checklist

When something goes wrong:

1. **Check API response:**
```javascript
   try {
     const result = await createPayment(data);
   } catch (error) {
     console.log('Status:', error.response?.status);
     console.log('Error:', error.response?.data);
     console.log('Headers:', error.response?.headers);
   }
```

2. **Verify request:**
```javascript
   console.log('Request:', {
     url: url,
     headers: headers,
     body: JSON.stringify(payload, null, 2)
   });
```

3. **Check environment:**
```bash
   echo "API Key: ${SENTINELGATE_API_KEY:0:20}..."
   echo "Base URL: $SENTINELGATE_BASE_URL"
```

4. **Test webhook locally:**
```bash
   # Use ngrok for local testing
   ngrok http 3000
   # Use the ngrok URL as callback_url
```

---

## Getting Help

If you're still stuck:

1. **Check status page:** http://status.sentinelgate.com
2. **Search documentation:** Use Ctrl+F in this guide
3. **Contact support:** support@sentinelgate.com

**Include in your support request:**
- Payment ID (if available)
- Error message
- Request/response logs
- Code snippet (remove credentials!)

