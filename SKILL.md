---
name: ordiscan
description: Query the Bitcoin blockchain (inscriptions, runes, BRC-20, alkanes, rare sats) and inscribe content on Bitcoin via the Ordiscan API. Pays per-request with USDC on Base using the x402 protocol.
homepage: https://ordiscan.com/docs/api
compatibility: Requires either awal CLI (Coinbase Agentic Wallet) or node + X402_PRIVATE_KEY env var for x402 payments.
metadata:
  openclaw:
    requires:
      anyBins: [awal, node]
    primaryEnv: X402_PRIVATE_KEY
    emoji: "\U0001F7E0"
allowed-tools: Bash(npx awal*) Bash(node:*) Bash(curl:*)
---

# Ordiscan API

Query the Bitcoin blockchain and inscribe content via the Ordiscan API. Every request is paid with USDC on Base using the **x402 payment protocol** -- no API key needed.

## Setup

Two payment modes are supported. The agent should auto-detect which one to use (check for `awal` first).

### Mode A -- Coinbase Agentic Wallet (recommended)

If the `awal` CLI is installed and authenticated, use it. One command handles the full x402 negotiate-sign-pay flow.

```bash
# Check if awal is available
which awal
```

### Mode B -- Signing script (fallback)

Requires `node` and the `X402_PRIVATE_KEY` environment variable (an Ethereum private key with USDC on Base).

```bash
# Install dependencies (only once)
npm install --prefix <skill-dir>

# Verify connectivity
X402_PRIVATE_KEY=$X402_PRIVATE_KEY node <skill-dir>/scripts/x402-sign.mjs balance
```

## Making requests with awal (preferred)

### Querying data

```bash
npx awal x402 pay "https://api.ordiscan.com/v1/inscription/0"
```

### Inscribe content

```bash
npx awal x402 pay "https://api.ordiscan.com/v1/inscribe" \
  --method POST \
  --data '{"contentType":"text/plain","base64_content":"SGVsbG8gd29ybGQ=","recipientAddress":"bc1p..."}' \
  --max-amount 5000000
```

The `--max-amount` flag caps the payment in USDC base units (6 decimals). `5000000` = $5.00 USDC.

## Making requests with signing script (fallback)

This is a 3-step flow: request -> sign -> retry.

### Step 1: Make the request (get 402 + payment header)

```bash
HEADER=$(curl -s -o /tmp/x402_body.json -w '%header{x-payment-required}' \
  -H "Content-Type: application/json" \
  "https://api.ordiscan.com/v1/inscription/0")
```

For POST requests, add `-X POST -d '...'`.

### Step 2: Sign the payment

```bash
PAYMENT=$(X402_PRIVATE_KEY=$X402_PRIVATE_KEY node <skill-dir>/scripts/x402-sign.mjs sign "$HEADER")
```

The script reads the base64-encoded `X-Payment-Required` header, signs an ERC-3009 `TransferWithAuthorization`, and outputs the base64-encoded `X-Payment` header to stdout. Diagnostics go to stderr.

### Step 3: Retry with payment

```bash
curl -s -H "X-Payment: $PAYMENT" \
  -H "Content-Type: application/json" \
  "https://api.ordiscan.com/v1/inscription/0"
```

## Preparing content for inscription

When the user asks to inscribe something, follow these steps:

### 1. Determine the content and MIME type

- **User provides text** (e.g. "inscribe 'Hello world'"): use `text/plain`
- **User provides a file path**: detect the MIME type from the extension:
  | Extension | Content type |
  |---|---|
  | `.txt` | `text/plain` |
  | `.html`, `.htm` | `text/html` |
  | `.svg` | `image/svg+xml` |
  | `.json` | `application/json` |
  | `.png` | `image/png` |
  | `.jpg`, `.jpeg` | `image/jpeg` |
  | `.gif` | `image/gif` |
  | `.webp` | `image/webp` |
  | `.mp3` | `audio/mpeg` |
  | `.mp4` | `video/mp4` |
  | `.pdf` | `application/pdf` |

### 2. Base64-encode the content

For text:
```bash
CONTENT=$(echo -n 'Hello world' | base64)
```

For a file:
```bash
CONTENT=$(base64 -w0 path/to/file)
```

`-w0` disables line wrapping (important -- the API expects a single unbroken base64 string). On macOS, `base64` doesn't wrap by default, so `-w0` can be omitted.

### 3. Get the recipient Bitcoin address

Ask the user for their Bitcoin address if not already provided. This is the address that will own the inscription.

### 4. Call the inscribe endpoint

See the worked examples below.

## Inscribe endpoint (deep dive)

`POST /v1/inscribe` creates a Bitcoin inscription. The server builds and broadcasts the commit + reveal transactions.

### Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `contentType` | string | yes | MIME type (e.g. `text/plain`, `image/png`, `text/html`) |
| `base64_content` | string | yes | Base64-encoded content (max 400KB decoded) |
| `recipientAddress` | string | yes | Bitcoin address to receive the inscription |
| `feeRate` | number | no | Custom fee rate in sat/vB (defaults to medium) |

### 402 response (no payment)

When called without an `X-Payment` header, the server returns 402 with dynamic pricing:

```json
{
  "error": { "message": "Payment required..." },
  "priceUsdc": 1.23,
  "totalSats": 4567,
  "feeRate": 12,
  "btcPriceUsd": 100000
}
```

The `X-Payment-Required` header contains the base64-encoded x402 payment details.

### Success response (200)

```json
{
  "data": {
    "commitTxid": "abc123...",
    "revealTxid": "def456...",
    "inscriptionId": "def456...i0"
  }
}
```

### Worked example with awal

```bash
# Encode content
CONTENT=$(echo -n "Hello Bitcoin!" | base64)

# Inscribe (awal handles the 402 flow automatically)
npx awal x402 pay "https://api.ordiscan.com/v1/inscribe" \
  --method POST \
  --data "{\"contentType\":\"text/plain\",\"base64_content\":\"$CONTENT\",\"recipientAddress\":\"bc1p...\"}" \
  --max-amount 5000000
```

### Worked example with signing script

```bash
# Encode content
CONTENT=$(echo -n "Hello Bitcoin!" | base64)
BODY="{\"contentType\":\"text/plain\",\"base64_content\":\"$CONTENT\",\"recipientAddress\":\"bc1p...\"}"

# Step 1: Get price and payment header
HEADER=$(curl -s -o /tmp/x402_body.json -w '%header{x-payment-required}' \
  -X POST -H "Content-Type: application/json" \
  -d "$BODY" \
  "https://api.ordiscan.com/v1/inscribe")

# Check the price
cat /tmp/x402_body.json

# Step 2: Sign
PAYMENT=$(X402_PRIVATE_KEY=$X402_PRIVATE_KEY node <skill-dir>/scripts/x402-sign.mjs sign "$HEADER")

# Step 3: Send with payment
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "X-Payment: $PAYMENT" \
  -d "$BODY" \
  "https://api.ordiscan.com/v1/inscribe"
```

### Inscribing images

```bash
CONTENT=$(base64 -w0 image.png)   # -w0 to avoid line wraps

npx awal x402 pay "https://api.ordiscan.com/v1/inscribe" \
  --method POST \
  --data "{\"contentType\":\"image/png\",\"base64_content\":\"$CONTENT\",\"recipientAddress\":\"bc1p...\"}" \
  --max-amount 10000000
```

### Inscribing HTML

```bash
CONTENT=$(echo -n '<html><body><h1>On-chain page</h1></body></html>' | base64)

npx awal x402 pay "https://api.ordiscan.com/v1/inscribe" \
  --method POST \
  --data "{\"contentType\":\"text/html\",\"base64_content\":\"$CONTENT\",\"recipientAddress\":\"bc1p...\"}" \
  --max-amount 5000000
```

## API endpoint reference

Base URL: `https://api.ordiscan.com`

### Address balance

| Method | Path | Description |
|---|---|---|
| GET | `/v1/address/{address}/utxos` | UTXOs with inscriptions and runes |
| GET | `/v1/address/{address}/inscription-ids` | Owned inscription IDs |
| GET | `/v1/address/{address}/inscriptions` | Owned inscriptions |
| GET | `/v1/address/{address}/runes` | Rune balances |
| GET | `/v1/address/{address}/brc20` | BRC-20 balances |
| GET | `/v1/address/{address}/rare-sats` | Rare sat balance |

### Address activity

| Method | Path | Description |
|---|---|---|
| GET | `/v1/address/{address}/activity` | Inscription activity |
| GET | `/v1/address/{address}/activity/runes` | Runes activity |
| GET | `/v1/address/{address}/activity/brc20` | BRC-20 activity |

### Transaction

| Method | Path | Description |
|---|---|---|
| GET | `/v1/tx/{txid}` | Transaction info |
| GET | `/v1/tx/{txid}/inscriptions` | New inscriptions in TX |
| GET | `/v1/tx/{txid}/inscription-transfers` | Transferred inscriptions in TX |
| GET | `/v1/tx/{txid}/runes` | Runes in TX |
| GET | `/v1/tx/{txid}/alkanes` | Alkanes in TX |

### Inscriptions

| Method | Path | Description |
|---|---|---|
| GET | `/v1/inscription/{id_or_number}` | Inscription info |
| GET | `/v1/inscription/{id_or_number}/traits` | Inscription traits |
| GET | `/v1/inscription/{id_or_number}/activity` | Transfer activity |
| GET | `/v1/inscriptions` | List inscriptions (with filters) |

### Runes

| Method | Path | Description |
|---|---|---|
| GET | `/v1/runes` | List runes |
| GET | `/v1/rune/{name}` | Rune info |
| GET | `/v1/rune/{name}/market` | Rune market info |

### Collections

| Method | Path | Description |
|---|---|---|
| GET | `/v1/collections` | List collections |
| GET | `/v1/collection/{slug}` | Collection info |
| GET | `/v1/collection/{slug}/inscriptions` | Collection inscriptions |
| GET | `/v1/collection/{slug}/market` | Collection market info |

### Alkanes

| Method | Path | Description |
|---|---|---|
| GET | `/v1/alkanes` | List alkanes |
| GET | `/v1/alkane/{id}` | Alkane info |
| GET | `/v1/alkane/{id}/meta` | Alkane metadata |
| GET | `/v1/address/{address}/alkanes` | Alkane address balance |
| GET | `/v1/address/{address}/utxos/alkanes` | Alkane address UTXOs |

### BRC-20

| Method | Path | Description |
|---|---|---|
| GET | `/v1/brc20` | List BRC-20 tokens |
| GET | `/v1/brc20/{tick}` | BRC-20 token info |

### Sats & UTXOs

| Method | Path | Description |
|---|---|---|
| GET | `/v1/sat/{number}` | Sat info |
| GET | `/v1/utxo/{txid}:{vout}/rare-sats` | Rare sats for UTXO |
| GET | `/v1/utxo/{txid}:{vout}/sat-ranges` | Sat ranges for UTXO |

### Blocks

| Method | Path | Description |
|---|---|---|
| GET | `/v1/block/{hash_or_height}` | Block info |
| GET | `/v1/block/{hash_or_height}/rune_txids` | Rune TXIDs in block |

### Inscribe

| Method | Path | Description |
|---|---|---|
| POST | `/v1/inscribe` | Inscribe content on Bitcoin (x402 payment) |

## Response format

**Success:**
```json
{ "data": { ... } }
```

**Error:**
```json
{ "error": { "message": "..." } }
```

## Error handling

| Status | Meaning | Action |
|---|---|---|
| 400 | Bad request (invalid params) | Fix the request body or parameters |
| 402 | Payment required | Sign and send payment via x402 |
| 429 | Rate limited | Wait and retry (max 10 requests/min for inscribe) |
| 503 | Service unavailable | Server issue, retry later |

## Tips

- **Auto-detect payment mode**: Check if `awal` is on PATH first (`which awal`). If available, use it. Otherwise fall back to the signing script with `X402_PRIVATE_KEY`.
- **GET requests cost ~$0.01 USDC each.** Inscription requests vary based on content size and Bitcoin fee rates.
- **For inscriptions, always check the 402 response** to see the price before paying. The `priceUsdc` field tells you the exact cost.
- **Content limit is 400KB** (decoded). For images, keep them reasonable in size.
- **The inscribe endpoint returns `commitTxid` and `revealTxid`**. You can track them on `https://mempool.space/tx/{txid}` or `https://ordiscan.com/inscription/{inscriptionId}`.
