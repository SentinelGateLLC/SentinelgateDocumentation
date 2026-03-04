# Apple Pay & Google Pay Integration

Accept payments via Apple Pay and Google Pay through the SentinelGate Wallet Decryption API. This guide covers client-side setup, server-side token submission, and end-to-end payment flow.

---

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│  Customer    │────▶│  Your App    │────▶│  SentinelGate   │────▶│  Card Network│
│  (iOS/Web)   │     │  (Frontend)  │     │  Wallet API     │     │  (Auth)      │
└─────────────┘     └──────────────┘     └─────────────────┘     └──────────────┘
       │                    │                      │
   Biometric           Payment Token         Decrypt + ISO8583
   Auth / Face ID      (encrypted)           Map → Authorize
```

**Flow:**
1. Customer taps Apple Pay / Google Pay on your app or website
2. Device SDK returns an encrypted payment token
3. Your backend sends the token to SentinelGate's `/v1/wallets/decrypt` endpoint
4. SentinelGate decrypts, verifies, maps to ISO 8583, and authorizes
5. You receive a webhook with the payment result

---

## Apple Pay

### Prerequisites

- Apple Developer account with Apple Pay merchant ID
- Payment Processing Certificate (`.p12` file) from Apple
- Domain verified with Apple Pay

### Client-Side Setup (Web — Apple Pay JS)

```html
<script>
async function onApplePayClicked() {
  if (!window.ApplePaySession) {
    alert("Apple Pay is not available on this device.");
    return;
  }

  const request = {
    countryCode: "US",
    currencyCode: "USD",
    supportedNetworks: ["visa", "masterCard", "amex"],
    merchantCapabilities: ["supports3DS"],
    total: {
      label: "Your Store",
      amount: "49.99"
    }
  };

  const session = new ApplePaySession(3, request);

  session.onvalidatemerchant = async (event) => {
    // Call your backend to get the merchant session
    const resp = await fetch("/api/apple-pay/validate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ validationURL: event.validationURL })
    });
    const merchantSession = await resp.json();
    session.completeMerchantValidation(merchantSession);
  };

  session.onpaymentauthorized = async (event) => {
    const token = event.payment.token;

    // Send token to your backend → SentinelGate
    const resp = await fetch("/api/apple-pay/process", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        payment_token: token.paymentData,
        amount: 49.99,
        currency: "USD"
      })
    });

    const result = await resp.json();
    session.completePayment(
      result.ok
        ? ApplePaySession.STATUS_SUCCESS
        : ApplePaySession.STATUS_FAILURE
    );
  };

  session.begin();
}
</script>

<button onclick="onApplePayClicked()">
  Pay with Apple Pay
</button>
```

### Client-Side Setup (iOS — Swift)

```swift
import PassKit

func presentApplePay() {
    let request = PKPaymentRequest()
    request.merchantIdentifier = "merchant.com.yourstore"
    request.supportedNetworks = [.visa, .masterCard, .amex]
    request.merchantCapabilities = .capability3DS
    request.countryCode = "US"
    request.currencyCode = "USD"
    request.paymentSummaryItems = [
        PKPaymentSummaryItem(label: "Your Store", amount: NSDecimalNumber(string: "49.99"))
    ]

    guard let controller = PKPaymentAuthorizationViewController(paymentRequest: request) else { return }
    controller.delegate = self
    present(controller, animated: true)
}

// In PKPaymentAuthorizationViewControllerDelegate:
func paymentAuthorizationViewController(_ controller: PKPaymentAuthorizationViewController,
                                         didAuthorizePayment payment: PKPayment,
                                         handler completion: @escaping (PKPaymentAuthorizationResult) -> Void) {
    let tokenData = payment.token.paymentData

    // Send tokenData to your backend → SentinelGate
    YourAPI.processApplePay(tokenData: tokenData) { success in
        completion(PKPaymentAuthorizationResult(
            status: success ? .success : .failure,
            errors: nil
        ))
    }
}
```

### Server-Side: Submit to SentinelGate

```python
import requests
import json

# Token received from Apple Pay client SDK
apple_pay_token = {
    "version": "EC_v1",
    "data": "base64-encrypted-data...",
    "signature": "base64-signature...",
    "header": {
        "ephemeralPublicKey": "base64-key...",
        "publicKeyHash": "base64-hash...",
        "transactionId": "abc123..."
    }
}

response = requests.post(
    "https://sentinelgate.biz/v1/wallets/decrypt",
    headers={
        "Content-Type": "application/json",
        "X-API-Key": "sg_key_xxxxx",
        "X-API-Secret": "sg_secret_xxxxx",
        "Idempotency-Key": "unique-request-id-123"
    },
    json={
        "provider": "APPLE_PAY",
        "payment_token": apple_pay_token,
        "amount_cents": 4999,
        "currency": "USD"
    }
)

result = response.json()
print(result)
# {
#   "provider": "APPLE_PAY",
#   "transaction_id": "abc123...",
#   "iso8583": { "2": "****", "4": "000000004999", "14": "2812", ... },
#   "card": { "last4": "1234", "network": "Visa", "expiryYYMM": "2812", "eci": "05" },
#   "latency_ms": 42
# }
```

---

## Google Pay

### Prerequisites

- Google Pay Merchant ID from the Google Pay Business Console
- EC private key for decryption (generated via Google Pay console)

### Client-Side Setup (Web)

```html
<script src="https://pay.google.com/gp/p/js/pay.js"></script>
<script>
const paymentsClient = new google.payments.api.PaymentsClient({
  environment: "PRODUCTION"  // or "TEST"
});

const paymentDataRequest = {
  apiVersion: 2,
  apiVersionMinor: 0,
  allowedPaymentMethods: [{
    type: "CARD",
    parameters: {
      allowedAuthMethods: ["PAN_ONLY", "CRYPTOGRAM_3DS"],
      allowedCardNetworks: ["VISA", "MASTERCARD", "AMEX"]
    },
    tokenizationSpecification: {
      type: "DIRECT",
      parameters: {
        protocolVersion: "ECv2",
        publicKey: "YOUR_EC_PUBLIC_KEY_BASE64"
      }
    }
  }],
  merchantInfo: {
    merchantId: "YOUR_GOOGLE_MERCHANT_ID",
    merchantName: "Your Store"
  },
  transactionInfo: {
    totalPriceStatus: "FINAL",
    totalPrice: "49.99",
    currencyCode: "USD",
    countryCode: "US"
  }
};

async function onGooglePayClicked() {
  try {
    const paymentData = await paymentsClient.loadPaymentData(paymentDataRequest);
    const token = JSON.parse(
      paymentData.paymentMethodData.tokenizationData.token
    );

    // Send token to your backend → SentinelGate
    const resp = await fetch("/api/google-pay/process", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        payment_token: token,
        amount: 49.99,
        currency: "USD"
      })
    });

    const result = await resp.json();
    if (result.ok) {
      window.location.href = "/success";
    }
  } catch (err) {
    console.error("Google Pay error:", err);
  }
}
</script>

<button onclick="onGooglePayClicked()">
  Pay with Google Pay
</button>
```

### Client-Side Setup (Android — Kotlin)

```kotlin
val paymentDataRequest = PaymentDataRequest.fromJson("""
{
  "apiVersion": 2,
  "apiVersionMinor": 0,
  "allowedPaymentMethods": [{
    "type": "CARD",
    "parameters": {
      "allowedAuthMethods": ["PAN_ONLY", "CRYPTOGRAM_3DS"],
      "allowedCardNetworks": ["VISA", "MASTERCARD"]
    },
    "tokenizationSpecification": {
      "type": "DIRECT",
      "parameters": {
        "protocolVersion": "ECv2",
        "publicKey": "$EC_PUBLIC_KEY"
      }
    }
  }],
  "transactionInfo": {
    "totalPriceStatus": "FINAL",
    "totalPrice": "49.99",
    "currencyCode": "USD"
  }
}
""".trimIndent())

AutoResolveHelper.resolveTask(
    paymentsClient.loadPaymentData(paymentDataRequest),
    activity,
    LOAD_PAYMENT_DATA_REQUEST_CODE
)
```

### Server-Side: Submit to SentinelGate

```javascript
const axios = require("axios");

// Token received from Google Pay client SDK
const googlePayToken = {
  protocolVersion: "ECv2",
  signature: "base64-signature...",
  signedMessage: "{\"encryptedMessage\":\"...\",\"ephemeralPublicKey\":\"...\",\"tag\":\"...\"}"
};

const response = await axios.post(
  "https://sentinelgate.biz/v1/wallets/decrypt",
  {
    provider: "GOOGLE_PAY",
    payment_token: googlePayToken,
    amount_cents: 4999,
    currency: "USD"
  },
  {
    headers: {
      "Content-Type": "application/json",
      "X-API-Key": "sg_key_xxxxx",
      "X-API-Secret": "sg_secret_xxxxx",
      "Idempotency-Key": "unique-request-id-456"
    }
  }
);

console.log(response.data);
// {
//   "provider": "GOOGLE_PAY",
//   "iso8583": { "2": "****", "4": "000000004999", "14": "2812", ... },
//   "card": { "last4": "5678", "network": "Mastercard", "expiryYYMM": "2812", "eci": "05" },
//   "latency_ms": 38
// }
```

---

## API Reference

### `POST /v1/wallets/decrypt`

Decrypt a wallet payment token, verify cryptographic signatures, map to ISO 8583 fields, and authorize the payment.

**Headers:**

| Header | Required | Description |
|--------|----------|-------------|
| `X-API-Key` | Yes | Your merchant API key |
| `X-API-Secret` | Yes | Your merchant API secret |
| `Content-Type` | Yes | `application/json` |
| `Idempotency-Key` | Recommended | Unique request ID (prevents duplicate processing) |

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider` | string | Yes | `APPLE_PAY` or `GOOGLE_PAY` |
| `payment_token` | object | Yes | The encrypted token from the wallet SDK |
| `amount_cents` | integer | Yes | Payment amount in minor units (cents) |
| `currency` | string | Yes | ISO 4217 currency code (e.g., `USD`) |

**Response:**

```json
{
  "provider": "APPLE_PAY",
  "transaction_id": "abc123...",
  "iso8583": {
    "2": "****1234",
    "4": "000000004999",
    "14": "2812",
    "55": "cryptogram-data...",
    "60.1": "05",
    "22": "010",
    "25": "59",
    "49": "USD",
    "123": "APPLEPAY"
  },
  "card": {
    "last4": "1234",
    "network": "Visa",
    "expiryYYMM": "2812",
    "eci": "05"
  },
  "latency_ms": 42
}
```

**Error Codes:**

| Code | Meaning |
|------|---------|
| `WALLET_DECRYPTION_DISABLED` | Wallet decryption is disabled on this account |
| `INVALID_PROVIDER` | Provider must be `APPLE_PAY` or `GOOGLE_PAY` |
| `INVALID_REQUEST` | Missing required fields |
| `APPLEPAY_UNSUPPORTED_VERSION` | Token version not supported (must be EC_v1) |
| `APPLEPAY_SIGNATURE_INVALID` | PKCS#7 signature verification failed |
| `APPLEPAY_LEAF_OID_MISSING` | Apple Pay OID not found in certificate |
| `GOOGLEPAY_UNSUPPORTED_VERSION` | Token must be ECv2 |
| `GOOGLEPAY_MESSAGE_EXPIRED` | Token has expired |
| `GOOGLEPAY_INTERMEDIATE_EXPIRED` | Intermediate signing key expired |
| `GOOGLEPAY_INTERMEDIATE_SIGNATURE_INVALID` | Intermediate key signature invalid |
| `GOOGLEPAY_MAC_INVALID` | HMAC verification failed |

---

## Security

- **No PAN returned**: The response never contains the full card number — only `last4`
- **No PAN logged**: Card numbers are masked in all server logs
- **Idempotency**: Duplicate requests with the same `Idempotency-Key` return cached results
- **Signature verification**: Both Apple Pay and Google Pay tokens are cryptographically verified before decryption
- **Body size limit**: Request body capped at 256KB
- **Token expiry**: Google Pay message expiration is enforced
