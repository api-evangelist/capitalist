---
name: capitalist-create-payout
description: Send an outbound payout through Capitalist to a card, bank, wallet, mobile operator or cryptocurrency address, sign the request correctly, and confirm the terminal state. Use when a funded Capitalist account must pay a named recipient over a specific payment channel.
api: Capitalist Integration API (v2)
base_url: https://api2.capitalist.net
operations:
  - POST /v1/payment
  - GET /v1/payment/{userRequestId}
  - GET /v1/account/list
generated: '2026-09-05'
method: generated
source: https://docs.capitalist.net/api/integration-api.html
---

# Create a Capitalist payout

The Capitalist Integration API has no operationIds — the provider's OpenAPI
declares none — so every step below names the method and path exactly as
published.

## Before you start

- You need an API key and API secret from https://capitalist.net/security. The
  account must have API access enabled and Google 2FA active.
- If the account uses an IP allowlist, your egress IP must already be in it
  (`GET /v1/whitelist`). A rejected source IP is not distinguishable from a bad
  signature in the error body.
- **There is no sandbox.** Every call below hits production and moves real
  money. Capitalist's own guidance is to start with a minimal amount.

## Sign every request

All three headers are required on every call:

```
API-Key: <your api key>
X-Request-Timestamp: <epoch milliseconds>
Signature: sha256_hex(X-Request-Timestamp + raw_request_body + API_secret)
```

The signature is over the **raw serialized body you actually send**, byte for
byte. Serialize once, sign that string, send that string. Re-serializing between
signing and sending is the failure this scheme punishes. For a GET the body is
empty, so the signed string is `timestamp + "" + secret`.

## 1. Confirm the funding account

```
GET /v1/account/list?currency=USD
```

Returns `[{ "number": "U0123504", "currency": "USD", "name": "...", "balance": 1250.50 }]`.
Pick the `number` to use as `accountFrom` and check `balance` covers the amount
**plus the fee** — the fee is not quoted before submission, only returned after.

## 2. Build the channel payload

`payload.type` selects the payment channel and decides which recipient fields
are required. The channel families are cards (RUCARD, RUCARDP2P, TRCARD, GECARD,
AZCARD, KZCARD, UZCARD, WORLDCARDEUR, WORLDCARDUSD), bank transfers (ARBANK,
BRBANK, COBANK, MALAYSIA_BANK, INDONESIA_BANK, THAILAND_BANK, SOUTH_KOREA_BANK),
mobile operators (MEGAFON, TMOBILE, BEELINE, MTS, TELE2, YOTA), fast payment
systems (SBP, IMPS), e-wallets (PAYONEER, EUR_NETELLER, EUR_SKRILL, PAYTM,
GCASH) and cryptocurrency (BITCOIN, ETH, USDCERC20, USDTERC20, USDTTRC20).

Read section 6 of https://docs.capitalist.net/api/integration-api.html for the
required fields of your channel, or the `components.schemas.<TYPE>` entry in
`openapi/capitalist-integration-api-openapi.json`. Do not guess field names —
each channel schema is distinct.

Capitalist warns that not every documented channel is necessarily live. There is
no capability-discovery endpoint, so treat an unavailable channel as a runtime
error, not a configuration error.

## 3. Create the payment

```
POST /v1/payment
{
  "userRequestId": "<your unique id>",
  "accountFrom": "U0123504",
  "amount": 100,
  "currency": "USD",
  "comment": "invoice 4412",
  "callbackUrl": "https://your-host/capitalist/callback",
  "payload": { "type": "RUCARD", "account": "...", "name": "...", "surname": "...", "midname": "..." }
}
```

Returns `{ "documentId": <int64> }`.

**`userRequestId` is the whole of your safety.** It is the only duplicate guard
Capitalist documents, it must be unique per intended payment, and it must be
generated and persisted *before* the call — not derived from the response. If
the HTTP call fails or times out, do **not** retry with a new id: read the
outcome back with `GET /v1/payment/{userRequestId}` first.

Amounts are decimal major units (100.00 USD), not minor units. Currency codes
are not pure ISO 4217: the rouble is `RUR`, and stablecoins carry a network
qualifier — `USDT` (ERC-20), `USDTt` (TRC-20), `USDTb` (BEP-20), `USDC`
(ERC-20), `USDCb` (BEP-20).

## 4. Take the outcome from the callback

Supply `callbackUrl` and let Capitalist POST you the final state rather than
polling. The callback body carries `state`, `fee`, `documentId`, `amount`,
`currency`, `type`, `accountFrom`, `userRequestId` and `comment`.

Verify it before trusting it: recompute
`sha256_hex(X-Request-Timestamp header + raw body + API secret)` and compare
with the `Signature` header. Treat delivery as at-least-once and dedupe on
`documentId`.

If you must poll instead, use `GET /v1/payment/{userRequestId}` and stay under
20 requests per minute — the provider's stated ceiling.

## Terminal states

`PENDING` → `EXECUTED` or `DECLINED`. Both terminal states are final.

**There is no reversal.** No cancel, void, refund or reverse operation exists in
this API. A `DECLINED` payment returns no reason code — Capitalist publishes no
decline taxonomy — so an agent cannot tell a retryable failure from a permanent
one and must escalate to a human. "Payment return (Wire)" and "Search for
incorrect payment" on https://capitalist.net/fees are chargeable manual support
services with no stated time window; do not model them as an API path.

## Errors

`HTTP 400` with `{ "error": "free text" }`. There are no stable error codes.
`HTTP 401` with the plain-text body ``Missing `API-Key` `` means the auth
headers did not arrive or did not verify. See
`errors/capitalist-problem-types.yml`.
