---
name: capitalist-reconcile-payouts
description: Reconcile Capitalist payouts and account movements — read balances, list ledger transactions and merchant orders over a date window, and resolve any payment back to its originating request. Use for settlement reporting, fee accounting and payout audits.
api: Capitalist Integration API (v2)
base_url: https://api2.capitalist.net
operations:
  - GET /v1/account/list
  - GET /v1/transactions
  - GET /v1/orders
  - GET /v1/payment/document/{documentId}
  - GET /v1/payment/{userRequestId}
generated: '2026-09-05'
method: generated
source: https://docs.capitalist.net/api/integration-api.html
---

# Reconcile Capitalist activity

Read-only. Nothing in this skill moves money.

Sign every request with `API-Key`, `X-Request-Timestamp` and
`Signature = sha256_hex(timestamp + raw_body + secret)`. For GETs the body is
empty, so the signed string is `timestamp + secret`.

## 1. Balances

```
GET /v1/account/list?currency=USD
```

One call per currency; `currency` is required, so there is no "all accounts"
read. Iterate the currencies you hold: RUR, USD, EUR, BTC, ETH, USDT, USDTt,
USDTb, USDC, USDCb.

## 2. Ledger transactions

```
GET /v1/transactions?periodStart=2026-08-01T00:00:00Z&periodEnd=2026-09-01T00:00:00Z&limit=100&offset=0
```

Returns `{ "transactions": [...], "count": <total> }`. Page with `offset` until
you have consumed `count`. Each row carries `transactionId`, `createDate`,
`executeDate`, `type`, `state`, `amount`, `currency`, `planDate`, `version`,
and for crypto movements `txId` and `dstAddress`.

Note `createDate` and `executeDate` differ — settle on `executeDate` for
accounting periods and keep `createDate` for latency analysis.

## 3. Merchant orders (inbound collections)

```
GET /v1/orders?periodStart=...&periodEnd=...&limit=100&offset=0
```

Same `{ orders, count }` envelope and the same offset paging. Order states are
`NEW`, `PARTIALLYPAID`, `PAID`, `FAIL`, `REFUND`, `CHARGEBACK`, `CANCELLED`.
`amount` and `paidAmount` can differ — always reconcile on `paidAmount`.
`REFUND` and `CHARGEBACK` are observable states only; no API operation creates
either, so they arrive without a corresponding call of yours.

## 4. Resolve a payment either way

```
GET /v1/payment/document/{documentId}      # by Capitalist's id
GET /v1/payment/{userRequestId}            # by your id
```

Both return the same shape: `state`, `fee`, `documentId`, `comment`, `amount`,
`currency`, `type`, `accountFrom`, `payload`, `userRequestId`, `callbackUrl`.

`payload` comes back as a **JSON-encoded string**, not a nested object — parse
it a second time before reading recipient details.

`fee` is the authoritative per-transaction cost. Capitalist publishes no
machine-readable price list (the fee tables at https://capitalist.net/fees are
rendered client-side and described as updating every few minutes), so the only
reliable way to account for cost is to read `fee` off each settled document.

## Rate limiting

Stay under 20 status reads per minute. There is no `Retry-After`, no
`RateLimit-*` header and no documented throttling status code, so you get no
runtime signal — pace the job yourself rather than reacting to one. For ongoing
status, consume the signed callback (`asyncapi/capitalist-webhooks.yml`) instead
of polling.

## Gaps to plan around

- `GET /v1/transactions` and `GET /v1/orders` are documented but are **not** in
  the published OpenAPI, so no generated client will have them. Hand-roll these
  two calls.
- There is no cursor and no `updated_since` filter — only offset paging over a
  date window. For an incremental job, re-read the whole window and dedupe on
  `transactionId` / `orderId`.
- No server request-id header, so correlate support tickets on `documentId` or
  your own `userRequestId`.
