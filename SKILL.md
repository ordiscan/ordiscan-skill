---
name: ordiscan
description: Inscribe content on Bitcoin via the Ordiscan API. Pays per-request with USDC on Base using the x402 protocol.
homepage: https://ordiscan.com/docs/api
metadata: {"openclaw":{"emoji":"🟠","requires":{"bins":["awal"]}}}
---

# Ordiscan API

Inscribe content and query the Bitcoin blockchain via the Ordiscan API. Every request is paid with USDC on Base using the **x402 payment protocol** -- no API key needed.

## What this skill can do

- **Inscribe content on Bitcoin** -- text, images, HTML, SVG, and any other file type. The server builds and broadcasts the Bitcoin transactions for you. You only pay USDC on Base.
- **Query blockchain data** -- look up inscriptions, addresses, runes, BRC-20 tokens, rare sats, and more.

## Setup

This skill uses the `awal` CLI (Coinbase Agentic Wallet) for x402 payments. Make sure it's installed and authenticated:

```bash
npx awal status
```

If not authenticated, run `npx awal` and follow the setup flow.

## Querying data

```bash
npx awal x402 pay "https://api.ordiscan.com/v1/inscription/0"
```

## Inscribing content

```bash
npx awal x402 pay "https://api.ordiscan.com/v1/inscribe" \
  --method POST \
  --data '{"contentType":"text/plain","base64_content":"SGVsbG8gd29ybGQ=","recipientAddress":"bc1p..."}' \
  --max-amount 5000000
```

The `--max-amount` flag caps the payment in USDC base units (6 decimals). `5000000` = $5.00 USDC.

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

```bash
npx awal x402 pay "https://api.ordiscan.com/v1/inscribe" \
  --method POST \
  --data "{\"contentType\":\"text/plain\",\"base64_content\":\"$CONTENT\",\"recipientAddress\":\"bc1p...\"}" \
  --max-amount 5000000
```

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

When called without payment, the server returns 402 with dynamic pricing:

```json
{
  "error": { "message": "Payment required..." },
  "priceUsdc": 1.23,
  "totalSats": 4567,
  "feeRate": 12,
  "btcPriceUsd": 100000
}
```

`awal` handles this automatically -- it reads the 402 response, signs the payment, and retries.

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

### Inscribing images

```bash
CONTENT=$(base64 -w0 image.png)

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

See the [Ordiscan API documentation](https://ordiscan.com/docs/api.md)

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
| 402 | Payment required | `awal` handles this automatically |
| 429 | Rate limited | Wait and retry (max 10 requests/min for inscribe) |
| 503 | Service unavailable | Server issue, retry later |

## Tips

- **GET requests cost ~$0.01 USDC each.** Inscription requests vary based on content size and Bitcoin fee rates.
- **For inscriptions, always check the 402 response** to see the price before paying. The `priceUsdc` field tells you the exact cost.
- **Content limit is 400KB** (decoded). For images, keep them reasonable in size.
- **The inscribe endpoint returns `commitTxid` and `revealTxid`**. Track them on `https://mempool.space/tx/{txid}` or `https://ordiscan.com/inscription/{inscriptionId}`.
