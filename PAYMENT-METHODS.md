# Supported Payment Methods

SentinelGate supports 25+ payment methods across cards, bank transfers, mobile money, digital wallets, and crypto. All methods are accessed through the unified `/v1/hosted/create` or `/v1/charge` API.

> **Important**: Payment methods are routed to the optimal acquirer automatically. Merchants do not need to specify or manage acquirer/provider details.

---

## Card Payments

### Visa / Mastercard

Accept Visa and Mastercard payments globally.

| Field | Value |
|-------|-------|
| **Method Code** | `CARD` |
| **Currencies** | `USD`, `EUR`, `GBP`, `CAD`, `NGN`, `MXN`, `BRL`, `JPY`, `KES`, `ZAR`, `HKD`, `CNY` |
| **Regions** | Global |
| **3D Secure** | Supported (automatic) |

```json
{
  "method": "CARD",
  "amount": 100.00,
  "currency": "USD",
  "description": "Premium subscription",
  "callback_url": "https://yoursite.com/webhook",
  "return_url": "https://yoursite.com/success"
}
```

### Apple Pay

Accept payments via Apple Pay on iOS devices and Safari.

| Field | Value |
|-------|-------|
| **Method Code** | `APPLE_PAY` |
| **Currencies** | `USD`, `EUR`, `GBP`, `CAD`, `JPY`, `HKD`, `CNY` |
| **Regions** | Global (where Apple Pay is available) |

```json
{
  "method": "APPLE_PAY",
  "amount": 49.99,
  "currency": "USD",
  "payment_token": { "...Apple Pay token from client SDK..." },
  "callback_url": "https://yoursite.com/webhook"
}
```

See [Apple Pay & Google Pay Integration Guide](WALLET-PAY.md) for full setup instructions.

### Google Pay

Accept payments via Google Pay on Android and Chrome.

| Field | Value |
|-------|-------|
| **Method Code** | `GOOGLE_PAY` |
| **Currencies** | `USD`, `EUR`, `GBP`, `CAD`, `JPY`, `HKD`, `CNY` |
| **Regions** | Global (where Google Pay is available) |

```json
{
  "method": "GOOGLE_PAY",
  "amount": 49.99,
  "currency": "USD",
  "payment_token": { "...Google Pay token from client SDK..." },
  "callback_url": "https://yoursite.com/webhook"
}
```

See [Apple Pay & Google Pay Integration Guide](WALLET-PAY.md) for full setup instructions.

### China UnionPay

Accept China UnionPay card payments.

| Field | Value |
|-------|-------|
| **Method Code** | `UNIONPAY` |
| **Currencies** | `CNY`, `USD`, `HKD` |
| **Regions** | China, Hong Kong, Global (where UnionPay is accepted) |

```json
{
  "method": "UNIONPAY",
  "amount": 500.00,
  "currency": "CNY",
  "callback_url": "https://yoursite.com/webhook",
  "return_url": "https://yoursite.com/success"
}
```

---

## Mobile Money

### M-Pesa (Kenya)

Collect payments via M-Pesa STK Push.

| Field | Value |
|-------|-------|
| **Method Code** | `MPESA` |
| **Currencies** | `KES` |
| **Regions** | Kenya |

```json
{
  "method": "MPESA",
  "amount": 1000,
  "currency": "KES",
  "phone": "254712345678",
  "description": "Order payment",
  "callback_url": "https://yoursite.com/webhook"
}
```

### Mobile Money (Ghana)

Collect payments via MTN MoMo, Vodafone Cash, AirtelTigo Money.

| Field | Value |
|-------|-------|
| **Method Code** | `MOMO_GH` |
| **Currencies** | `GHS` |
| **Regions** | Ghana |

```json
{
  "method": "MOMO_GH",
  "amount": 50.00,
  "currency": "GHS",
  "phone": "233241234567",
  "network": "MTN",
  "callback_url": "https://yoursite.com/webhook"
}
```

### Mobile Money (East Africa)

Collect payments via mobile money across East Africa.

| Field | Value |
|-------|-------|
| **Method Code** | `MOMO_EA` |
| **Currencies** | `KES`, `UGX`, `TZS` |
| **Regions** | Kenya, Uganda, Tanzania |

```json
{
  "method": "MOMO_EA",
  "amount": 5000,
  "currency": "UGX",
  "phone": "256701234567",
  "callback_url": "https://yoursite.com/webhook"
}
```

### MMI — Mobile Money Interoperability (Ghana)

Cross-network mobile money transfers in Ghana via GhIPSS interoperability.

| Field | Value |
|-------|-------|
| **Method Code** | `MMI` |
| **Currencies** | `GHS` |
| **Regions** | Ghana |

```json
{
  "method": "MMI",
  "amount": 100.00,
  "currency": "GHS",
  "phone": "233201234567",
  "callback_url": "https://yoursite.com/webhook"
}
```

### SINPE Móvil

Mobile payments in Costa Rica via SINPE Móvil.

| Field | Value |
|-------|-------|
| **Method Code** | `SINPE_MOVIL` |
| **Currencies** | `CRC`, `USD` |
| **Regions** | Costa Rica |

```json
{
  "method": "SINPE_MOVIL",
  "amount": 25000,
  "currency": "CRC",
  "phone": "50688881234",
  "callback_url": "https://yoursite.com/webhook"
}
```

---

## Bank Transfers & Direct Debit

### ACH (USA)

Direct ACH bank transfers within the United States.

| Field | Value |
|-------|-------|
| **Method Code** | `ACH_US` |
| **Currencies** | `USD` |
| **Regions** | United States |

```json
{
  "method": "ACH_US",
  "amount": 250.00,
  "currency": "USD",
  "bank_account": {
    "routing_number": "021000021",
    "account_number": "123456789",
    "account_type": "checking"
  },
  "callback_url": "https://yoursite.com/webhook"
}
```

### ACH (Ghana)

Direct ACH bank transfers in Ghana.

| Field | Value |
|-------|-------|
| **Method Code** | `ACH_GH` |
| **Currencies** | `GHS` |
| **Regions** | Ghana |

```json
{
  "method": "ACH_GH",
  "amount": 500.00,
  "currency": "GHS",
  "bank_code": "030100",
  "account_number": "0012345678",
  "callback_url": "https://yoursite.com/webhook"
}
```

### ACH (Dominican Republic)

ACH transfers in the Dominican Republic.

| Field | Value |
|-------|-------|
| **Method Code** | `ACH_DO` |
| **Currencies** | `DOP`, `USD` |
| **Regions** | Dominican Republic |

```json
{
  "method": "ACH_DO",
  "amount": 5000,
  "currency": "DOP",
  "bank_code": "BPD",
  "account_number": "0012345678",
  "callback_url": "https://yoursite.com/webhook"
}
```

### eCheck (USA)

Electronic check payments within the United States.

| Field | Value |
|-------|-------|
| **Method Code** | `ECHECK_US` |
| **Currencies** | `USD` |
| **Regions** | United States |

```json
{
  "method": "ECHECK_US",
  "amount": 175.00,
  "currency": "USD",
  "bank_account": {
    "routing_number": "021000021",
    "account_number": "123456789",
    "account_type": "checking",
    "check_number": "1001"
  },
  "payer": { "name": "John Doe", "email": "john@example.com" },
  "callback_url": "https://yoursite.com/webhook"
}
```

### EFT (Canada)

Electronic Funds Transfer for Canadian bank payments.

| Field | Value |
|-------|-------|
| **Method Code** | `EFT_CA` |
| **Currencies** | `CAD` |
| **Regions** | Canada |

```json
{
  "method": "EFT_CA",
  "amount": 300.00,
  "currency": "CAD",
  "bank_account": {
    "institution_number": "001",
    "transit_number": "12345",
    "account_number": "1234567"
  },
  "callback_url": "https://yoursite.com/webhook"
}
```

### Interac (Canada)

Interac e-Transfer for Canadian payments.

| Field | Value |
|-------|-------|
| **Method Code** | `INTERAC` |
| **Currencies** | `CAD` |
| **Regions** | Canada |

```json
{
  "method": "INTERAC",
  "amount": 150.00,
  "currency": "CAD",
  "email": "customer@example.com",
  "callback_url": "https://yoursite.com/webhook",
  "return_url": "https://yoursite.com/success"
}
```

### SEPA

Single Euro Payments Area transfers across Europe.

| Field | Value |
|-------|-------|
| **Method Code** | `SEPA` |
| **Currencies** | `EUR` |
| **Regions** | EU/EEA |

```json
{
  "method": "SEPA",
  "amount": 200.00,
  "currency": "EUR",
  "iban": "DE89370400440532013000",
  "payer": { "name": "Hans Mueller", "email": "hans@example.de" },
  "callback_url": "https://yoursite.com/webhook"
}
```

### PIX (Brazil)

Instant payments via PIX in Brazil.

| Field | Value |
|-------|-------|
| **Method Code** | `PIX` |
| **Currencies** | `BRL` |
| **Regions** | Brazil |

```json
{
  "method": "PIX",
  "amount": 150.00,
  "currency": "BRL",
  "payer": {
    "cpf": "12345678901",
    "name": "Maria Silva",
    "email": "maria@example.com.br"
  },
  "callback_url": "https://yoursite.com/webhook"
}
```

### UPI (India)

Unified Payments Interface for Indian bank payments.

| Field | Value |
|-------|-------|
| **Method Code** | `UPI` |
| **Currencies** | `INR` |
| **Regions** | India |

```json
{
  "method": "UPI",
  "amount": 999.00,
  "currency": "INR",
  "vpa": "customer@upi",
  "callback_url": "https://yoursite.com/webhook"
}
```

### Bank to Bank Transfer Collections

Direct bank-to-bank collection transfers.

| Field | Value |
|-------|-------|
| **Method Code** | `BANK_TRANSFER` |
| **Currencies** | `USD`, `NGN` |
| **Regions** | Nigeria, United States |

```json
{
  "method": "BANK_TRANSFER",
  "amount": 50000,
  "currency": "NGN",
  "bank_code": "058",
  "account_number": "0123456789",
  "callback_url": "https://yoursite.com/webhook"
}
```

### SIPARD / LBTR (Dominican Republic)

Real-time gross settlement in the Dominican Republic.

| Field | Value |
|-------|-------|
| **Method Code** | `SIPARD` |
| **Currencies** | `DOP`, `USD` |
| **Regions** | Dominican Republic |

```json
{
  "method": "SIPARD",
  "amount": 100000,
  "currency": "DOP",
  "bank_code": "BPD",
  "account_number": "0012345678",
  "callback_url": "https://yoursite.com/webhook"
}
```

---

## Instant Payment Systems

### NIP — NIBSS Instant Payment (Nigeria)

Real-time interbank transfers in Nigeria.

| Field | Value |
|-------|-------|
| **Method Code** | `NIP` |
| **Currencies** | `NGN` |
| **Regions** | Nigeria |

```json
{
  "method": "NIP",
  "amount": 25000,
  "currency": "NGN",
  "bank_code": "058",
  "account_number": "0123456789",
  "payer": { "name": "Chinedu Obi" },
  "callback_url": "https://yoursite.com/webhook"
}
```

### GIP — GhIPSS Instant Pay (Ghana)

Real-time interbank payments in Ghana.

| Field | Value |
|-------|-------|
| **Method Code** | `GIP` |
| **Currencies** | `GHS` |
| **Regions** | Ghana |

```json
{
  "method": "GIP",
  "amount": 200.00,
  "currency": "GHS",
  "bank_code": "030100",
  "account_number": "0012345678",
  "callback_url": "https://yoursite.com/webhook"
}
```

### PesaLink (Kenya)

Real-time interbank transfers in Kenya.

| Field | Value |
|-------|-------|
| **Method Code** | `PESALINK` |
| **Currencies** | `KES` |
| **Regions** | Kenya |

```json
{
  "method": "PESALINK",
  "amount": 5000,
  "currency": "KES",
  "bank_code": "01",
  "account_number": "1234567890",
  "callback_url": "https://yoursite.com/webhook"
}
```

### RNDPS (Rwanda)

Rwanda National Digital Payment System.

| Field | Value |
|-------|-------|
| **Method Code** | `RNDPS` |
| **Currencies** | `RWF` |
| **Regions** | Rwanda |

```json
{
  "method": "RNDPS",
  "amount": 50000,
  "currency": "RWF",
  "phone": "250781234567",
  "callback_url": "https://yoursite.com/webhook"
}
```

### TISS — Tanzania Interbank Settlement System

Interbank settlement in Tanzania.

| Field | Value |
|-------|-------|
| **Method Code** | `TISS` |
| **Currencies** | `TZS` |
| **Regions** | Tanzania |

```json
{
  "method": "TISS",
  "amount": 100000,
  "currency": "TZS",
  "bank_code": "CRDB",
  "account_number": "0123456789",
  "callback_url": "https://yoursite.com/webhook"
}
```

---

## Crypto Payments

### Crypto Processing

Accept cryptocurrency payments for goods and services. Customers pay in crypto, you receive settlement in fiat or crypto.

| Field | Value |
|-------|-------|
| **Method Code** | `CRYPTO_PAY` |
| **Currencies** | `BTC`, `ETH`, `USDT`, `USDC` (settlement in `USD`, `EUR`, `GBP`) |
| **Regions** | Global |

```json
{
  "method": "CRYPTO_PAY",
  "amount": 100.00,
  "currency": "USD",
  "settlement_currency": "USD",
  "accepted_coins": ["BTC", "ETH", "USDT", "USDC"],
  "callback_url": "https://yoursite.com/webhook",
  "return_url": "https://yoursite.com/success"
}
```

### Crypto Transfer

Merchant-initiated crypto transfers to any wallet address.

| Field | Value |
|-------|-------|
| **Method Code** | `CRYPTO_TRANSFER` |
| **Currencies** | `BTC`, `ETH`, `USDT`, `USDC` |
| **Regions** | Global |

```json
{
  "method": "CRYPTO_TRANSFER",
  "amount": 0.05,
  "currency": "ETH",
  "destination_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f2bD18",
  "network": "ethereum",
  "callback_url": "https://yoursite.com/webhook"
}
```

### Buy Crypto

Enable merchants to purchase cryptocurrency.

| Field | Value |
|-------|-------|
| **Method Code** | `CRYPTO_BUY` |
| **Currencies** | `USD`, `EUR`, `GBP` → `BTC`, `ETH`, `USDT`, `USDC` |
| **Regions** | Global (subject to compliance) |

```json
{
  "method": "CRYPTO_BUY",
  "amount": 500.00,
  "currency": "USD",
  "target_coin": "BTC",
  "destination_address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "callback_url": "https://yoursite.com/webhook"
}
```

---

## Gaming & Digital Tokens

### eChip Token Processing

Collect eChips from gamers or digital buyers.

| Field | Value |
|-------|-------|
| **Method Code** | `ECHIP_COLLECT` |
| **Currencies** | `ECHIP` (internal token) |
| **Regions** | Global |

```json
{
  "method": "ECHIP_COLLECT",
  "amount": 500,
  "currency": "ECHIP",
  "gamer_id": "gamer_abc123",
  "description": "In-game purchase: Legendary Sword",
  "callback_url": "https://yoursite.com/webhook"
}
```

### eChip Exchange

Exchange eChips for cryptocurrency or fiat currency.

| Field | Value |
|-------|-------|
| **Method Code** | `ECHIP_EXCHANGE` |
| **Currencies** | `ECHIP` → `USD`, `BTC`, `ETH`, `USDT` |
| **Regions** | Global |

```json
{
  "method": "ECHIP_EXCHANGE",
  "amount": 1000,
  "currency": "ECHIP",
  "target_currency": "USDT",
  "destination_address": "0x742d35Cc6634C0532925a3b844Bc9e7595f2bD18",
  "callback_url": "https://yoursite.com/webhook"
}
```

---

## Summary Table

| # | Method | Code | Currencies | Region |
|---|--------|------|-----------|--------|
| 1 | Visa / Mastercard | `CARD` | USD, EUR, GBP, CAD, NGN, MXN, BRL, JPY, KES, ZAR, HKD, CNY | Global |
| 2 | Apple Pay | `APPLE_PAY` | USD, EUR, GBP, CAD, JPY, HKD, CNY | Global |
| 3 | Google Pay | `GOOGLE_PAY` | USD, EUR, GBP, CAD, JPY, HKD, CNY | Global |
| 4 | China UnionPay | `UNIONPAY` | CNY, USD, HKD | China, Global |
| 5 | UPI | `UPI` | INR | India |
| 6 | ACH (USA) | `ACH_US` | USD | United States |
| 7 | ACH (Ghana) | `ACH_GH` | GHS | Ghana |
| 8 | ACH (Dominican Republic) | `ACH_DO` | DOP, USD | Dominican Republic |
| 9 | eCheck (USA) | `ECHECK_US` | USD | United States |
| 10 | EFT (Canada) | `EFT_CA` | CAD | Canada |
| 11 | Interac (Canada) | `INTERAC` | CAD | Canada |
| 12 | SEPA | `SEPA` | EUR | EU/EEA |
| 13 | PIX | `PIX` | BRL | Brazil |
| 14 | Bank to Bank Transfer | `BANK_TRANSFER` | USD, NGN | Nigeria, US |
| 15 | NIP (Nigeria) | `NIP` | NGN | Nigeria |
| 16 | GIP (Ghana) | `GIP` | GHS | Ghana |
| 17 | MMI (Ghana) | `MMI` | GHS | Ghana |
| 18 | PesaLink (Kenya) | `PESALINK` | KES | Kenya |
| 19 | RNDPS (Rwanda) | `RNDPS` | RWF | Rwanda |
| 20 | TISS (Tanzania) | `TISS` | TZS | Tanzania |
| 21 | M-Pesa (Kenya) | `MPESA` | KES | Kenya |
| 22 | Mobile Money (Ghana) | `MOMO_GH` | GHS | Ghana |
| 23 | Mobile Money (East Africa) | `MOMO_EA` | KES, UGX, TZS | Kenya, Uganda, Tanzania |
| 24 | SINPE Móvil | `SINPE_MOVIL` | CRC, USD | Costa Rica |
| 25 | SIPARD / LBTR (DR) | `SIPARD` | DOP, USD | Dominican Republic |
| 26 | Crypto Processing | `CRYPTO_PAY` | BTC, ETH, USDT, USDC | Global |
| 27 | Crypto Transfer | `CRYPTO_TRANSFER` | BTC, ETH, USDT, USDC | Global |
| 28 | Buy Crypto | `CRYPTO_BUY` | USD, EUR, GBP → BTC, ETH, USDT, USDC | Global |
| 29 | eChip Token Processing | `ECHIP_COLLECT` | ECHIP | Global |
| 30 | eChip Exchange | `ECHIP_EXCHANGE` | ECHIP → USD, BTC, ETH, USDT | Global |
