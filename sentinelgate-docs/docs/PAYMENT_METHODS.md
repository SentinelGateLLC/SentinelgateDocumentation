# Supported Payment Methods

SentinelGate supports 25+ payment methods across cards, mobile money, real-time payments, bank transfers, crypto, and gaming tokens.

---

## Method Directory

### Cards & Wallets
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 1 | UPI (Unified Payments Interface) | `UPI` | INR | [UPI Guide](methods/UPI.md) |
| 2 | China UnionPay | `CUP` | CNY, USD | [UnionPay Guide](methods/CHINA_UNIONPAY.md) |

### ACH & Direct Debit
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 3 | ACH (USA) | `ACH_US` | USD | [ACH US Guide](methods/ACH_US.md) |
| 4 | ACH (Ghana) | `ACH_GH` | GHS | [ACH Ghana Guide](methods/ACH_GHANA.md) |
| 5 | eCheck (USA) | `ECHECK_US` | USD | [eCheck Guide](methods/ECHECK_US.md) |
| 6 | ACH (Dominican Republic) | `ACH_DO` | DOP | [ACH DR Guide](methods/ACH_DOMINICAN.md) |

### Real-Time Payment Networks
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 7 | SEPA (Europe) | `SEPA` | EUR | [SEPA Guide](methods/SEPA.md) |
| 8 | PIX (Brazil) | `PIX` | BRL | [PIX Guide](methods/PIX.md) |
| 9 | EFT (Canada) | `EFT_CA` | CAD | [EFT Guide](methods/EFT_CANADA.md) |
| 10 | Interac (Canada) | `INTERAC` | CAD | [Interac Guide](methods/INTERAC_CANADA.md) |

### Africa Real-Time
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 11 | NIP — NIBSS Instant Payment (Nigeria) | `NIP_NG` | NGN | [NIP Guide](methods/NIP_NIGERIA.md) |
| 12 | GIP — GhIPSS Instant Pay (Ghana) | `GIP_GH` | GHS | [GIP Guide](methods/GIP_GHANA.md) |
| 13 | MMI — Mobile Money Interop (Ghana) | `MMI_GH` | GHS | [MMI Guide](methods/MMI_GHANA.md) |
| 14 | PesaLink (Kenya) | `PESALINK` | KES | [PesaLink Guide](methods/PESALINK.md) |
| 15 | RNDPS (Rwanda) | `RNDPS_RW` | RWF | [RNDPS Guide](methods/RNDPS_RWANDA.md) |
| 16 | TISS (Tanzania) | `TISS_TZ` | TZS | [TISS Guide](methods/TISS_TANZANIA.md) |

### Latin America
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 17 | SNPE Móvil (Costa Rica) | `SINPE_CR` | CRC | [SINPE Guide](methods/SINPE_MOVIL.md) |
| 18 | Snipe Movil | `SNIPE` | MXN | [Snipe Guide](methods/SNIPE_MOVIL.md) |
| 19 | SIPARD / LBTR (Dominican Republic) | `SIPARD_DO` | DOP | [SIPARD Guide](methods/SIPARD_DOMINICAN.md) |

### Bank-to-Bank
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 20 | Bank-to-Bank Transfer Collections | `B2B_COLLECT` | Multi | [B2B Guide](methods/BANK_TO_BANK.md) |

### Crypto
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 21 | Crypto Processing (Collect) | `CRYPTO_PAY` | BTC, ETH, USDT, USDC | [Crypto Pay Guide](methods/CRYPTO_PROCESSING.md) |
| 22 | Crypto Transfer (Send) | `CRYPTO_SEND` | BTC, ETH, USDT, USDC | [Crypto Transfer Guide](methods/CRYPTO_TRANSFER.md) |
| 23 | Buy Crypto (For Merchants) | `CRYPTO_BUY` | BTC, ETH, USDT, USDC | [Buy Crypto Guide](methods/CRYPTO_BUY.md) |

### Gaming & Tokens
| # | Method | Rail Code | Currencies | Guide |
|---|--------|-----------|-----------|-------|
| 24 | eChip Token Processing | `ECHIP_PAY` | eCHIP | [eChip Pay Guide](methods/ECHIP_PROCESSING.md) |
| 25 | eChip Exchange | `ECHIP_SWAP` | eCHIP ↔ BTC/USD | [eChip Exchange Guide](methods/ECHIP_EXCHANGE.md) |

---

## Quick Integration

Every method follows the same API pattern:

```bash
curl -X POST https://sentinelgate.biz/v1/charge \
  -H "x-api-key: sk_live_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "amount_cents": 5000,
    "currency": "USD",
    "merchant_id": "your-merchant-id",
    "rail": "RAIL_CODE",
    "email": "customer@example.com",
    "reference": "ref_001",
    "callback_url": "https://yoursite.com/webhook"
  }'
```

Replace `RAIL_CODE` with the method's rail code from the tables above.

---

## Testing

Every method has a sandbox mode. Use test credentials:
- API Key prefix: `sk_test_`
- Test amounts, phone numbers, and account numbers are documented per method
- Sandbox transactions are free and never settle

---

© 2026 SentinelGate — Whyte AG Group
