---
name: paytrigo-moltbot
description: Use when a Moltbot needs to create or verify PayTrigo payments on Base/USDC without webhooks.
metadata: {"openclaw":{}}
---

# PayTrigo (Moltbot, Base/USDC)

This skill provides a minimal, bot-friendly payment flow on PayTrigo using a **platform API key** and **Base/USDC only**. It is optimized for Moltbots that do **not** run webhooks and instead **poll** for status. A public key is embedded in the helper script for no-setup usage.

## Assumptions
- **Platform key** is used (**`recipientAddress` required**).
- Base/USDC is the only enabled chain/token for this key (from Platform Fee Configs).
- Bots can store `invoiceId` and `checkoutToken` returned from invoice creation.

## No environment variables required
The helper script ships with a public key embedded, so Moltbots can use it immediately.

## Quickstart (CLI scripts)
Use the scenario scripts to test end-to-end flows without additional setup.

### Human-in-the-loop (user pays in browser)
```
node {baseDir}/scripts/moltbot-human-flow.mjs human --amount 0.001 --recipient 0xYourWallet...
```

### Bot pays directly (requires private key)
```
node {baseDir}/scripts/moltbot-bot-flow.mjs bot --amount 0.001 --recipient 0xYourWallet... --pk 0xPRIVATE_KEY
```

See `README.md` in this folder for a short Moltbot-focused guide.

## Core flow (Human-in-the-loop)
1) **Create invoice** (platform key, Base/USDC, recipientAddress required)
2) Send `payUrl` to the user (approval + payment)
3) **Poll** invoice status until `confirmed | expired | invalid | refunded`

## Core flow (Bot pays directly)
1) Create invoice
2) Get intent (approve/pay calldata)
3) Send on-chain tx (approve if needed, then pay)
4) Submit txHash
5) Poll status

> Important: **Direct token transfer is invalid**. Always use the Router `steps.pay` from `/intent`.

---

# API Usage (HTTP)

## 1) Create invoice
**Endpoint**: `POST /v1/invoices`

**Headers**:
- `Authorization: Bearer <platform_key>` (required if calling HTTP directly)
- `Content-Type: application/json`
- `Idempotency-Key: pay_attempt_<uuid>`

**Body (Base/USDC fixed, recipientAddress required)**
```json
{
  "amount": "49.99",
  "recipientAddress": "0xYourWallet...",
  "ttlSeconds": 900,
  "metadata": { "botId": "moltbot_123", "purpose": "checkout" }
}
```

**Response** includes `invoiceId`, `payUrl`, `checkoutToken`, `expiresAt`.

## 2) Get intent (bot-pay)
**Endpoint**: `GET /v1/invoices/{invoiceId}/intent?chain=base&token=usdc`

**Headers** (preferred):
- `X-Checkout-Token: <checkoutToken>`

**Response** includes `steps.approve`, `steps.pay`, `routerAddress`, `grossAmountAtomic`.

## 3) Submit payment intent (txHash)
**Endpoint**: `POST /v1/invoices/{invoiceId}/payment-intents`

**Headers**:
- `X-Checkout-Token: <checkoutToken>`
- `Content-Type: application/json`

**Body**
```json
{ "txHash": "0x...", "payerAddress": "0x..." }
```

## 4) Poll invoice status
**Endpoint**: `GET /v1/invoices/{invoiceId}`

**Headers**:
- `X-Checkout-Token: <checkoutToken>`

**Stop when**: `status` is `confirmed | expired | invalid | refunded`.

---

# Polling policy (safe default)
- `submitted` right after tx: poll every **3-5s** for 2 minutes
- After 2 minutes: poll every **10-15s**
- Stop at `expiresAt + grace` (status will not change after that)
- If you receive **429**, backoff and retry later

---

# Common mistakes
- **Missing `recipientAddress` with platform key** (invalid)
- **Direct token transfer** instead of Router pay
- **Losing checkoutToken** (it is only returned on invoice creation)
