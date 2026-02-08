# Tutorial: Integrate M-Pesa Payments (Step-by-Step)

**For:** Developers  
**Time:** 30 minutes  
**Prerequisites:** Basic knowledge of REST APIs and your programming language

---

## What You'll Build

A simple checkout page that accepts M-Pesa payments and confirms when payment is received.

---

## Step 1: Set Up Your Project

### Node.js/Express
```bash
mkdir mpesa-integration
cd mpesa-integration
npm init -y
npm install express axios dotenv body-parser
```

Create `.env` file:
```env
SENTINELGATE_API_KEY=sk_live_your_api_key
SENTINELGATE_API_SECRET=secret_your_secret
SENTINELGATE_WEBHOOK_SECRET=whsec_your_webhook_secret
PORT=3000
```

---

## Step 2: Create the SentinelGate Client

Create `sentinelgate.js`:
```javascript
const axios = require('axios');

class SentinelGate {
  constructor(apiKey, apiSecret) {
    this.apiKey = apiKey;
    this.apiSecret = apiSecret;
    this.baseUrl = 'http://185.229.224.244:3000';
  }

  async createMpesaPayment(amount, phoneNumber, description) {
    try {
      const response = await axios.post(
        `${this.baseUrl}/api/payments/create`,
        {
          amount_cents: Math.round(amount * 100),
          currency: 'KES',
          provider: 'MPESA_SAFARICOM',
          metadata: {
            phone_number: phoneNumber,
            description: description
          },
          callback_url: process.env.WEBHOOK_URL || 'http://localhost:3000/webhook'
        },
        {
          headers: {
            'X-API-Key': this.apiKey,
            'X-API-Secret': this.apiSecret,
            'Content-Type': 'application/json'
          },
          timeout: 30000
        }
      );

      return response.data;
    } catch (error) {
      console.error('SentinelGate API Error:', error.response?.data || error.message);
      throw error;
    }
  }

  async getPaymentStatus(paymentId) {
    const response = await axios.get(
      `${this.baseUrl}/api/payments/${paymentId}`,
      {
        headers: {
          'X-API-Key': this.apiKey,
          'X-API-Secret': this.apiSecret
        }
      }
    );

    return response.data;
  }
}

module.exports = SentinelGate;
```

---

## Step 3: Create the Server

Create `server.js`:
```javascript
require('dotenv').config();
const express = require('express');
const bodyParser = require('body-parser');
const crypto = require('crypto');
const SentinelGate = require('./sentinelgate');

const app = express();
const client = new SentinelGate(
  process.env.SENTINELGATE_API_KEY,
  process.env.SENTINELGATE_API_SECRET
);

// Store payments in memory (use a database in production!)
const payments = new Map();

// Parse JSON for most routes
app.use(bodyParser.json());

// Landing page
app.get('/', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
    <head>
      <title>M-Pesa Payment Demo</title>
      <style>
        body { font-family: Arial; max-width: 500px; margin: 50px auto; padding: 20px; }
        input, button { width: 100%; padding: 10px; margin: 10px 0; font-size: 16px; }
        button { background: #00A651; color: white; border: none; cursor: pointer; }
        button:hover { background: #008C44; }
        .result { margin-top: 20px; padding: 15px; border-radius: 5px; }
        .success { background: #d4edda; color: #155724; }
        .error { background: #f8d7da; color: #721c24; }
        .pending { background: #fff3cd; color: #856404; }
      </style>
    </head>
    <body>
      <h1>M-Pesa Payment Demo</h1>
      <form id="paymentForm">
        <input type="number" id="amount" placeholder="Amount (KES)" required min="10">
        <input type="tel" id="phone" placeholder="Phone (254712345678)" required>
        <input type="text" id="description" placeholder="Description" required>
        <button type="submit">Pay with M-Pesa</button>
      </form>
      <div id="result"></div>

      <script>
        document.getElementById('paymentForm').onsubmit = async (e) => {
          e.preventDefault();
          
          const amount = document.getElementById('amount').value;
          const phone = document.getElementById('phone').value;
          const description = document.getElementById('description').value;
          const resultDiv = document.getElementById('result');
          
          resultDiv.innerHTML = '<div class="pending">Processing payment...</div>';
          
          try {
            const response = await fetch('/create-payment', {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ amount, phone, description })
            });
            
            const data = await response.json();
            
            if (data.success) {
              resultDiv.innerHTML = \`
                <div class="pending">
                  <h3>Payment Initiated!</h3>
                  <p>Payment ID: \${data.payment_id}</p>
                  <p>Check your phone for M-Pesa prompt...</p>
                  <p>Status: <span id="status">PENDING</span></p>
                </div>
              \`;
              
              // Poll for status
              checkPaymentStatus(data.payment_id);
            } else {
              resultDiv.innerHTML = \`
                <div class="error">
                  <h3>Payment Failed</h3>
                  <p>\${data.error.message}</p>
                </div>
              \`;
            }
          } catch (error) {
            resultDiv.innerHTML = \`
              <div class="error">
                <h3>Error</h3>
                <p>\${error.message}</p>
              </div>
            \`;
          }
        };
        
        async function checkPaymentStatus(paymentId) {
          const maxAttempts = 60; // 5 minutes
          let attempts = 0;
          
          const interval = setInterval(async () => {
            attempts++;
            
            try {
              const response = await fetch(\`/payment-status/\${paymentId}\`);
              const data = await response.json();
              
              document.getElementById('status').textContent = data.status;
              
              if (data.status === 'SUCCESS') {
                clearInterval(interval);
                document.getElementById('result').innerHTML = \`
                  <div class="success">
                    <h3>✓ Payment Successful!</h3>
                    <p>Amount: KES \${data.amount_cents / 100}</p>
                    <p>Reference: \${data.provider_reference}</p>
                  </div>
                \`;
              } else if (data.status === 'FAILED') {
                clearInterval(interval);
                document.getElementById('result').innerHTML = \`
                  <div class="error">
                    <h3>✗ Payment Failed</h3>
                    <p>Please try again</p>
                  </div>
                \`;
              } else if (attempts >= maxAttempts) {
                clearInterval(interval);
                document.getElementById('result').innerHTML = \`
                  <div class="error">
                    <h3>Payment Timeout</h3>
                    <p>Please check your M-Pesa messages</p>
                  </div>
                \`;
              }
            } catch (error) {
              console.error('Status check error:', error);
            }
          }, 5000); // Check every 5 seconds
        }
      </script>
    </body>
    </html>
  `);
});

// Create payment endpoint
app.post('/create-payment', async (req, res) => {
  try {
    const { amount, phone, description } = req.body;
    
    // Validate input
    if (!amount || !phone || !description) {
      return res.status(400).json({
        success: false,
        error: { message: 'Missing required fields' }
      });
    }
    
    // Validate phone format
    if (!/^254\d{9}$/.test(phone)) {
      return res.status(400).json({
        success: false,
        error: { message: 'Phone must be in format 254XXXXXXXXX' }
      });
    }
    
    // Create payment
    const result = await client.createMpesaPayment(
      parseFloat(amount),
      phone,
      description
    );
    
    // Store payment
    payments.set(result.payment_id, {
      ...result,
      phone,
      description,
      created_at: new Date()
    });
    
    res.json(result);
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: {
        message: error.response?.data?.error?.message || 'Payment creation failed'
      }
    });
  }
});

// Get payment status endpoint
app.get('/payment-status/:paymentId', async (req, res) => {
  try {
    const { paymentId } = req.params;
    
    // Check local cache first
    if (payments.has(paymentId)) {
      const payment = payments.get(paymentId);
      if (payment.status !== 'PENDING') {
        return res.json(payment);
      }
    }
    
    // Fetch from API
    const status = await client.getPaymentStatus(paymentId);
    
    // Update cache
    payments.set(paymentId, status);
    
    res.json(status);
    
  } catch (error) {
    res.status(500).json({
      success: false,
      error: { message: 'Failed to fetch payment status' }
    });
  }
});

// Webhook endpoint (use raw body parser)
app.post('/webhook',
  bodyParser.raw({ type: 'application/json' }),
  (req, res) => {
    try {
      // Verify signature
      const signature = req.headers['x-webhook-signature'];
      const hmac = crypto.createHmac('sha256', process.env.SENTINELGATE_WEBHOOK_SECRET);
      const digest = hmac.update(req.body).digest('hex');
      
      if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest))) {
        console.error('Invalid webhook signature');
        return res.status(401).send('Invalid signature');
      }
      
      // Parse event
      const event = JSON.parse(req.body);
      console.log('Webhook received:', event);
      
      // Update payment in cache
      if (payments.has(event.payment_id)) {
        payments.set(event.payment_id, {
          ...payments.get(event.payment_id),
          status: event.status,
          provider_reference: event.provider_reference,
          updated_at: new Date()
        });
      }
      
      // Send success response immediately
      res.status(200).send('OK');
      
      // Process event asynchronously
      processWebhookEvent(event).catch(console.error);
      
    } catch (error) {
      console.error('Webhook error:', error);
      res.status(500).send('Error processing webhook');
    }
  }
);

async function processWebhookEvent(event) {
  switch (event.event) {
    case 'payment.completed':
      console.log(`Payment ${event.payment_id} completed successfully`);
      // Send confirmation email, update database, etc.
      break;
      
    case 'payment.failed':
      console.log(`Payment ${event.payment_id} failed`);
      // Notify customer, log error, etc.
      break;
      
    case 'payment.cancelled':
      console.log(`Payment ${event.payment_id} was cancelled`);
      // Handle cancellation
      break;
  }
}

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
  console.log('Open browser and visit http://localhost:3000');
});
```

---

## Step 4: Test It

1. **Start the server:**
```bash
   node server.js
```

2. **Open browser:**
```
   http://localhost:3000
```

3. **Test payment:**
   - Amount: 100
   - Phone: 254712345678 (test number)
   - Description: Test payment
   - Click "Pay with M-Pesa"

4. **Expected flow:**
   - Form submits
   - Shows "Payment Initiated"
   - In real scenario: Customer receives M-Pesa prompt on phone
   - After confirmation: Status updates to "SUCCESS"

---

## Step 5: Deploy to Production

1. **Get SSL certificate** (required for webhooks)
2. **Update environment variables** with production credentials
3. **Set webhook URL** to your production domain:
```
   https://yourdomain.com/webhook
```
4. **Deploy** to your hosting provider
5. **Test** with real phone number and small amount

---

## Common Issues & Solutions

### Issue: "Invalid phone number"
**Solution:** Ensure format is `254XXXXXXXXX` (no spaces, dashes, or +)

### Issue: Payment stays "PENDING"
**Solution:** 
- Check customer received M-Pesa prompt
- Verify phone number is correct
- Wait up to 2 minutes

### Issue: Webhook not received
**Solution:**
- Verify webhook URL is publicly accessible
- Check SSL certificate is valid
- Ensure server is running on correct port

---

## Next Steps

- [ ] Add database storage instead of in-memory Map
- [ ] Implement proper error logging
- [ ] Add email notifications
- [ ] Create admin dashboard
- [ ] Add payment reconciliation

---

**Complete code:** https://github.com/sentinelgate/examples/mpesa-integration

