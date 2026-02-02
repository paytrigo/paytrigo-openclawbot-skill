# paytrigo-openclawbot-skill

A minimal OpenClaw skill that lets Moltbots use PayTrigo on **Base/USDC** with no webhooks (polling only).

## Quickstart

### 1) Human-in-the-loop (user pays in browser)
```bash
node scripts/moltbot-human-flow.mjs human --amount 0.001 --recipient 0xYourWallet...
```
- Open the printed `payUrl` in a browser and complete payment
- The script polls until `confirmed`

### 2) Bot pays directly (requires private key)
```bash
node scripts/moltbot-bot-flow.mjs bot --amount 0.001 --recipient 0xYourWallet... --pk 0xPRIVATE_KEY
```
- Sends `approve` + `pay` transactions
- Submits txHash to PayTrigo
- Polls until final status

## Options
- `--ttl 900` : invoice TTL in seconds
- `--metadata '{"botId":"moltbot_123"}'` : metadata JSON
- `--poll 5` : polling interval (seconds)
- `--max-minutes 20` : max polling time (minutes)
- `--rpc https://mainnet.base.org` : Base RPC endpoint
- `--skip-approve` : skip approve if already approved

## Wallet / PK setup

### Recipient wallet (required)
You must pass `--recipient` on every run (platform key requirement).

```bash
echo "0xYourWallet" > recipient.txt
node scripts/moltbot-human-flow.mjs human --amount 0.001 --recipient "$(cat recipient.txt)"
```

### Payer private key (optional; only for bot-pay)
Store locally and never commit it.

```bash
echo "0xYOUR_PRIVATE_KEY" > payer.pk
node scripts/moltbot-bot-flow.mjs bot --amount 0.001 --recipient "$(cat recipient.txt)" --pk "$(cat payer.pk)"
```

## Success criteria
- Final `status` becomes `confirmed`
- USDC received in the recipient wallet

## Notes
- Platform key requires `recipientAddress`
- Direct token transfers are invalid; always use Router pay (handled by scripts)
- Never expose private keys
