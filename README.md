# Ordiscan

Inscribe content on Bitcoin via the Ordiscan API. Pays per-request with USDC on Base using the x402 protocol — no API key needed.

## Install from ClawHub

```bash
clawhub install ordiscan
```

 🎉 Now you can ask your agent to inscribe anything on Bitcoin (start with text, it's simple and cheap)!

## Requirements

Your agent needs to control an EVM wallet loaded with USDC on Base so that it can pay for API requests via
the x402 protocol.

**Recommended wallets:**
- **[evm-wallet skill](https://clawhub.ai/surfer77/evm-wallet)**: Does not require any additional
login. Stores you private key in a JSON file (only use it for small amounts).
- **[`awal` CLI](https://github.com/coinbase/awal)** (Coinbase Agentic Wallet): Alternative if already installed and authenticated with email. Verify with `npx awal status`.
