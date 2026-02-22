# Ordiscan

Inscribe content on Bitcoin and query ordinals data via the Ordiscan API. Pays per-request with USDC on Base using the x402 protocol — no API key needed.

## Install from ClawHub

```bash
clawhub install ordiscan
```

## Install from git

Clone into your OpenClaw skills directory:

```bash
git clone https://github.com/ordiscan/ordiscan-skill.git ~/.openclaw/skills/ordiscan
```

## Requirements

One of the following payment methods is needed to pay for API requests via the x402 protocol:

- **Signing script (default)**: Requires `node` and an Ethereum private key with USDC on Base. Works with any EVM wallet skill (e.g. `evm-wallet-skill`).
- **[`awal` CLI](https://github.com/coinbase/awal)** (Coinbase Agentic Wallet): Alternative if already installed and authenticated. Verify with `npx awal status`.

## Docs

Full API documentation: https://ordiscan.com/docs/api
