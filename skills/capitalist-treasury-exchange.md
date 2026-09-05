---
name: capitalist-treasury-exchange
description: Move funds between Capitalist currency accounts — quote a rate, convert fiat and crypto balances, and obtain an ephemeral cryptocurrency deposit address. Use to fund a payout currency before paying out, or to sweep an incoming crypto balance.
api: Capitalist Integration API (v2)
base_url: https://api2.capitalist.net
operations:
  - GET /v1/rate
  - POST /v1/exchange
  - GET /v1/account/list
  - GET /v1/depositAddress/{currency}
  - GET /v1/depositAddressAutoUSDTt/{account}
generated: '2026-09-05'
method: generated
source: https://docs.capitalist.net/api/integration-api.html
---

# Capitalist treasury: quote, convert, and receive crypto

Sign every request with `API-Key`, `X-Request-Timestamp` and
`Signature = sha256_hex(timestamp + raw_body + secret)`.

## 1. Quote the rate

```
GET /v1/rate?from=USD&to=EUR
```

The response body is a **bare JSON number** (`1.0842`), not an object. Do not
expect a field to read it from.

This is an indicative read, not a held quote. Capitalist publishes no quote id,
no expiry and no rate-lock mechanism, so the rate you convert at may differ from
the rate you read. Do not build a flow that promises a customer the quoted rate.

## 2. Convert between your own accounts

```
POST /v1/exchange
{ "fromAccount": "U0123504", "toAccount": "E0123505", "amount": 100.00 }
```

Both accounts must belong to the authenticated Capitalist account —
`GET /v1/account/list` gives you the `number` values.

Read this carefully before automating it:

- The call returns **`null`** on success. There is no document id, so a
  conversion cannot be looked up afterwards by its own key. Your only evidence
  it happened is the balance change and the resulting rows in
  `GET /v1/transactions`.
- It accepts **no idempotency key**. `userRequestId` exists on payment creation,
  not here. A retried or duplicated call converts twice.
- There is **no reversal**. Converting back is a new priced conversion at the
  then-current rate, not an undo.

Because of those three together, treat `POST /v1/exchange` as a single-shot,
non-retryable, human-confirmed operation. On a timeout or connection failure, do
**not** retry — read `GET /v1/account/list` and `GET /v1/transactions` to find
out whether it landed, then decide.

## 3. Receive cryptocurrency

```
GET /v1/depositAddress/{currency}
```

`currency` must be one of `BTC`, `ETH`, `USDT` (ERC-20), `USDTt` (TRC-20),
`USDC` (ERC-20), `USDTb` (BEP-20) or `USDCb` (BEP-20). Returns
`{ "address": "..." }`.

**Addresses are ephemeral.** Capitalist guarantees them for a maximum of 8 hours
and explicitly instructs integrators not to save or cache them. Fetch an address
at the moment you show it to a payer, never from storage. Sending to a stale
address is an unrecoverable loss — there is no reversal path anywhere in this
API.

To have incoming USDT TRC-20 converted automatically into a chosen account:

```
GET /v1/depositAddressAutoUSDTt/{account}
```

Same 8-hour rule.

## Currency codes

`RUR` is the Russian rouble (not ISO 4217's RUB). The stablecoin codes carry the
network: `USDT` = ERC-20, `USDTt` = TRC-20, `USDTb` = BEP-20, `USDC` = ERC-20,
`USDCb` = BEP-20. Choosing the wrong qualifier sends funds over the wrong
network.

## Errors

`HTTP 400` with `{ "error": "free text" }`, no stable codes. `HTTP 401` with the
plain-text body ``Missing `API-Key` `` for an auth failure.
