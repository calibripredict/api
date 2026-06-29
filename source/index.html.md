---
title: Calibri API Documentation

# language_tabs: # must be one of https://git.io/vQNgJ
#   - shell

toc_footers:
  - <a href="https://app.calibri.io" target="_blank">Calibri</a>

includes:
  - errors

search: true

code_clipboard: true
---

# Introduction

Welcome to the Calibri API Documentation. Calibri is a **binary prediction-market exchange**: each market resolves to one of two outcomes, **YES** or **NO**, and prices are probabilities between `0.01` and `0.99` where **YES + NO = 1.00**. You trade outcome shares on a single combined order book; at settlement the winning side pays `1.00` per share (minus fee) and the losing side `0.00`.

The data model is **events → markets → contracts**: an *event* is the real-world question, a *market* is its tradable YES/NO book, and a matched buy/sell becomes a *contract* held to settlement.

Calibri supports two ways to trade the **same** order book:

- **Custodial** — you deposit funds, Calibri holds them, and you place orders directly. Simplest integration.
- **Non-custodial** — you connect a self-custody wallet, keep funds in your own on-chain Gnosis Safe, and submit **signed orders** that settle on-chain. See [Non-custodial Trading](#non-custodial-trading).

Use this API to browse events and markets, place and cancel orders (custodial or non-custodial), view your positions and contracts, manage funds, and stream order-book and account updates over WebSocket.

🚀 Looking for features we haven't added yet? <a href="https://support.calibri.io/contact" target="_blank">Reach out to us</a>


# Getting started

The API suite consists of public and private endpoints. Private endpoints require your requests to be signed for authentication. In order to sign your requests, you will need to create and API key and secret, and have 2FA enabled:

1. Sign into your account <a href="https://app.calibri.io" target="_blank">Calibri</a>
2. Head to the <strong>Profile</strong> page by clicking the profile icon at the top right
3. Enable 2FA
4. Create a new API key
5. Save your new API key and Secret and do not share it with anyone

Use your API key and secret to sign your requests for all private endpoints. (Public endpoints do not require authentication)

All requests unless otherwise noted (EG for sub-account / customer management) use the following base URLs:

- Base url REST: `https://app.calibri.io/api/v2/atlas`
- Base url Websocket: `wss://app.calibri.io/api/v2/websocket`

For Sub-account and customer management (noted in the routes below):

- Base url: `https://app.calibri.io/api/v2/persona`

# Authentication

> Authentication:

```javascript
// generateHeaders.js

const crypto = require("crypto");
const apiKey = "123456...";
const secret = "ABCDEF...";

module.exports = async (subAccountUid = {}) => {
  const timeStamp = new Date().getTime();
  const queryString = `${timeStamp}${apikey}`;
  const signature = crypto
    .createHmac("sha256", secret)
    .update(queryString)
    .digest("hex");

  return {
    "Content-Type": "application/json;charset=utf-8",
    "X-Auth-Apikey": apikey,
    "X-Auth-Nonce": timeStamp,
    "X-Auth-Signature": signature,
    "X-Auth-Signature": signature,
    // Only for sub-account / customer requests:
    "X-Auth-Subaccount-UID": subAccountUid,
  };
};
```

<aside class="notice">
Ensure you have <strong>Enabled 2FA</strong> and <strong>Created your API key</strong>.
</aside>

To securely access the private endpoints, each request requires authentication headers.

Generate a new signature for each new request, and add these headers to the request.

1. <strong>X-Auth-Nonce</strong>: Current timestamp in milliseconds UTC time.
2. <strong>X-Auth-Apikey</strong>: Your API key
3. <strong>X-Auth-Signature</strong>: HMAC-SHA256 encrypted string of the combined timestamp, apikey and secret (see JS example)

## Acting on Behalf of Sub-Accounts or Customers

When making requests on behalf of a sub-account or customer, include an additional optional header:

4. <strong>X-Auth-Subaccount-Uid</strong>: The UID of the sub-account or customer you want to act as

This allows the parent account (or merchant/aggregator) to make authenticated requests as if they were the sub-account or customer. The regular authentication headers (API key, nonce, signature) must still be included.

# Example using Authentication

## <span class="request-type__post">POST</span> Create order

```javascript
// generateHeaders.js

const crypto = require("crypto");
const apiKey = "123456...";
const secret = "ABCDEF...";

module.exports = async () => {
  const timeStamp = new Date().getTime();
  const queryString = `${timeStamp}${apikey}`;
  const signature = crypto
    .createHmac("sha256", secret)
    .update(queryString)
    .digest("hex");

  return {
    "Content-Type": "application/json;charset=utf-8",
    "X-Auth-Apikey": apikey,
    "X-Auth-Nonce": timeStamp,
    "X-Auth-Signature": signature,
  };
};
```

> Send the authenticated request:

```javascript
// createOrder.js
const axios = require("axios");
const generateHeaders = require("./generateHeaders");

createOrder = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "POST";
  const path = `/api/v2/pythia/orders`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const orderData = {
    market: "12",
    side: "yes",
    ord_type: "limit",
    volume: "100",
    price: "0.60",
  };

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
    data: orderData,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};
```

> Create order response:

```json
{
  "id": 173757,
  "market": "12",
  "side": "yes",
  "ord_type": "limit",
  "price": "0.60",
  "state": "wait",
  "origin_volume": "100",
  "remaining_volume": "100",
  "locked": "60.0",
  "contracts_count": 0,
  "created_at": "2023-12-08T08:46:59+01:00",
  "updated_at": "2023-12-08T08:46:59+01:00"
}
```

`https://app.calibri.io/api/v2/pythia/orders`

Buys 100 YES shares at `0.60` USDC each, locking `60.00` USDC of collateral. Create a new order on your account by signing your request and submitting it.

## <span class="request-type__post">POST</span> Create order as sub-account

```javascript
// generateHeadersWithSubAccount.js

const crypto = require("crypto");
const apiKey = "123456...";
const secret = "ABCDEF...";
const subAccountUid = "CCT1234567"; // Sub-account or customer UID

module.exports = async () => {
  const timeStamp = new Date().getTime();
  const queryString = `${timeStamp}${apiKey}`;
  const signature = crypto
    .createHmac("sha256", secret)
    .update(queryString)
    .digest("hex");

  return {
    "Content-Type": "application/json;charset=utf-8",
    "X-Auth-Apikey": apiKey,
    "X-Auth-Nonce": timeStamp,
    "X-Auth-Signature": signature,
    "X-Auth-Subaccount-Uid": subAccountUid, // Act on behalf of sub-account/customer
  };
};
```

> Send the authenticated request as sub-account:

```javascript
// createOrderAsSubAccount.js
const axios = require("axios");
const generateHeaders = require("./generateHeadersWithSubAccount");

createOrder = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "POST";
  const path = `/api/v2/pythia/orders`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const orderData = {
    market: "12",
    side: "no",
    ord_type: "limit",
    volume: "50",
    price: "0.40",
  };

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
    data: orderData,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};
```

`https://app.calibri.io/api/v2/pythia/orders`

When the `X-Auth-Subaccount-Uid` header is included, the order is created on behalf of the specified sub-account or customer. The parent account's API credentials are used for authentication, but the operation is performed in the context of the sub-account.

# Prediction Markets

Calibri's prediction-market endpoints. The model is **events → markets → contracts**:

- An **event** is the real-world question (e.g. *"Will X happen by date Y?"*) and carries one or more markets.
- A **market** is the tradable book for one binary outcome pair (**YES / NO**). Prices are probabilities in `[0.01, 0.99]`; **YES + NO = 1.00**. Volume is denominated in *shares*; each winning share settles to `1.00` of the quote currency (USDC) minus the settlement fee, each losing share to `0.00`.
- A **contract** is a matched position (a YES holder vs a NO holder) held until the event resolves.

**Maker rebate:** the resting (maker) side of a matched trade earns a rebate — a percentage of the taker fee — credited automatically at settlement.

Public endpoints need no authentication. Order placement and account data require [authentication](#authentication) (API key or session).

- Public base URL: `https://app.calibri.io/api/v2/pythia/public`
- Trading base URL: `https://app.calibri.io/api/v2/pythia`
- Account base URL: `https://app.calibri.io/api/v2/atlas/account/predictions`

## <span class="request-type__get">GET</span> List events

`GET /api/v2/pythia/public/events`

### Description

Returns the catalogue of events. Each event includes its title, slug, category, status (`active`, `closed`, `settled`, …) and its child markets.

### Parameters

| Name     | In    | Description                                   | Required |
| -------- | ----- | --------------------------------------------- | -------- |
| category | query | Filter by category                            | No       |
| status   | query | Filter by event status                        | No       |
| limit    | query | Page size                                     | No       |

## <span class="request-type__get">GET</span> Event by slug

`GET /api/v2/pythia/public/events/{slug}`

### Description

Returns a single event (with its markets) by slug.

## <span class="request-type__get">GET</span> List markets

`GET /api/v2/pythia/public/markets`

### Description

Returns all tradable markets with current YES/NO prices, status, quote currency, and the parent event.

## <span class="request-type__get">GET</span> Market detail

`GET /api/v2/pythia/public/markets/{market}`

### Description

Returns a single market: YES/NO last prices, best bid/ask, status, close time, and settlement details once resolved.

## <span class="request-type__get">GET</span> Order book

`GET /api/v2/pythia/public/markets/{market}/order-book`

### Description

Aggregated combined order book for the market. Each level is `{ side, price, volume }` (YES bids and NO bids on one book — no order IDs or owner data are exposed). Use `/depth` for a depth-limited view.

## <span class="request-type__get">GET</span> Recent contracts

`GET /api/v2/pythia/public/markets/{market}/contracts/recent`

### Description

The public trade tape — recently matched contracts (price, volume, time) for the market.

## <span class="request-type__get">GET</span> Ticker / K-line / All tickers

`GET /api/v2/pythia/public/markets/{market}/ticker`<br/>
`GET /api/v2/pythia/public/markets/{market}/k-line`<br/>
`GET /api/v2/pythia/public/tickers`

### Description

Per-market ticker, candlestick (K-line) history, and a tickers snapshot across all markets.

## <span class="request-type__post">POST</span> Place order (custodial)

`POST /api/v2/pythia/orders`

> Body

```json
{
  "market": "12",
  "side": "yes",
  "ord_type": "limit",
  "volume": "100",
  "price": "0.60"
}
```

### Description

Places an order on the market's book using your **custodial** balance. Collateral (`price × volume` for the chosen side) is locked from your account immediately. This is the simplest way to trade — Calibri holds your funds and settles you off-chain at resolution.

Self-custody members must **not** use this endpoint; they place [non-custodial signed orders](#non-custodial-trading) instead, and a direct call returns `market.order.use_signed_endpoint`.

### Parameters

| Name     | Type   | Description                                            | Required |
| -------- | ------ | ------------------------------------------------------ | -------- |
| market   | string | Market id                                              | Yes      |
| side     | string | `yes` or `no`                                          | Yes      |
| ord_type | string | `limit` or `market`                                    | Yes      |
| volume   | string | Number of shares                                       | Yes      |
| price    | string | Limit price `0.01`–`0.99` (required for `limit`)       | For limit |

### Responses

| Code | Meaning                                          |
| ---- | ------------------------------------------------ |
| 201  | Order accepted (returns the order with its `id` and `state`) |
| 422  | Validation error (e.g. `market.order.invalid_price`) |

## <span class="request-type__get">GET</span> Your orders / trades

`GET /api/v2/pythia/orders`<br/>
`GET /api/v2/pythia/orders/{id}`

### Description

List your prediction orders, or fetch one by id. Includes state, filled volume, and contracts formed.

## <span class="request-type__post">POST</span> Cancel order

`POST /api/v2/pythia/orders/{id}/cancel`

### Description

Cancels a resting order and releases its locked collateral. Only the unmatched remainder is cancellable; matched positions are held to settlement.

## <span class="request-type__get">GET</span> Your contracts / positions

`GET /api/v2/atlas/account/predictions/contracts`<br/>
`GET /api/v2/atlas/account/predictions/contracts/{id}`

### Description

Your matched positions. Each contract shows the market, your side (YES/NO), share count, locked collateral, taker fee, and — once the event resolves — the settled state (`settled_yes` / `settled_no` / `voided`) and payout. You can only view contracts you are a party to.

# Non-custodial Trading

Non-custodial trading lets a user **keep their funds in their own on-chain wallet** and trade the **same** Calibri order book as custodial users, without ever depositing into Calibri's custody.

### How it differs from custodial

| | Custodial | Non-custodial |
| --- | --- | --- |
| **Where funds live** | Held by Calibri | In **your own Gnosis Safe** on-chain (USDC) — you can always withdraw, even if Calibri is unavailable |
| **How you order** | Plain `POST /pythia/orders` | You **sign an EIP-712 order**; Calibri validates and relays it |
| **Settlement** | Off-chain ledger | **On-chain** via Gnosis Conditional Tokens (trustless) |
| **Gas** | None | None for trading — Calibri's operator relays your Safe transactions (gasless); you only sign |
| **Order book / prices** | Shared | **Same** book and prices — custodial and non-custodial orders match against each other |

The trust model is wallet-custody: Calibri can never move your funds; it can only match orders you have cryptographically signed, and settlement is enforced by the on-chain Conditional Tokens contracts. You still trust Calibri to report the correct outcome at resolution (a bounded on-chain deadman releases funds if it never does).

### The flow

1. **Connect a wallet** and sign in (Sign-In-With-Ethereum). Your member identity is keyed to your wallet address.
2. **Fetch the market's CTF config** (below). It returns your per-member Safe (`proxy_address`), the exchange address, chain id, the YES/NO ERC-1155 token ids, and a one-time `safe_setup` if your Safe still needs trading approvals.
3. **If `safe_setup.ready` is false**, sign each approval `hash` and relay it (gaslessly) via [Relay Safe transaction](#relay-safe-transaction) to make the Safe trade-ready (USDC allowance + Conditional-Tokens operator approval). This is a one-time setup per Safe.
4. **Build and sign an EIP-712 order** (the Polymarket CTF Exchange `Order` struct — see below) with your wallet EOA: `maker` = your Safe, `signer` = your EOA, `signatureType` = `2` (POLY_GNOSIS_SAFE).
5. **Submit the signed order** via [Place signed order](#place-signed-order). Calibri fully validates it server-side, then places it on the shared book.
6. **Matching and settlement happen on-chain.** Funds move only via the Conditional Tokens contracts; your maker rebate and any winnings are credited the same as custodial.

## <span class="request-type__get">GET</span> CTF config

`GET /api/v2/atlas/account/predictions/markets/{id}/ctf-config`

> Response

```json
{
  "exchange_address": "0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9",
  "chain_id": 137,
  "yes_token_id": "713...",
  "no_token_id": "451...",
  "proxy_address": "0x0DECE7d83f8D47CD8dD8278c0F22c361C58577BD",
  "signature_type": 2,
  "safe_setup": {
    "ready": false,
    "calls": [
      { "to": "0x...", "data": "0x...", "operation": 0, "nonce": 0, "hash": "0x..." }
    ]
  }
}
```

### Description

Everything the browser needs to build and sign a non-custodial order for a CTF-registered market. `signature_type` is `2` (Gnosis Safe) when you have a Safe, or `0` (EOA-direct) otherwise. `safe_setup.calls` is non-empty only while the Safe still needs approvals — sign each `hash` and relay it (next endpoint); when already trade-ready it is `{ "ready": true, "calls": [] }`.

### Responses

| Code | Meaning |
| ---- | ------- |
| 200  | Config returned |
| 404  | `Market is not CTF-registered` |

## <span class="request-type__post">POST</span> Relay Safe transaction

`POST /api/v2/atlas/account/predictions/ctf/relay`

### Description

Gasless relay: submit a Safe-exec for an approval `hash` you signed from `ctf-config`'s `safe_setup`. Calibri's operator pays the gas and broadcasts it. Used to make your Safe trade-ready (one-time) — you sign, the operator executes.

## <span class="request-type__post">POST</span> Place signed order

`POST /api/v2/atlas/account/predictions/orders`

> Body

```json
{
  "market": "12",
  "side": "yes",
  "ord_type": "limit",
  "volume": "100",
  "price": "0.60",
  "signed_order": {
    "salt": "5821...",
    "maker": "0x0DECE7d83f8D47CD8dD8278c0F22c361C58577BD",
    "signer": "0xE1188438C85C98aADFf1a88f97764E607B4978C3",
    "taker": "0x0000000000000000000000000000000000000000",
    "token_id": "713...",
    "maker_amount": "60000000",
    "taker_amount": "100000000",
    "expiration": "1788000000",
    "nonce": "0",
    "fee_rate_bps": "0",
    "side": 0,
    "signature_type": 2,
    "signature": "0x..."
  }
}
```

### Description

Places a **non-custodial** order. The `signed_order` is an EIP-712 signature over the Polymarket CTF Exchange `Order` struct, signed by your wallet. Calibri **fully validates it server-side** before placing — the leg price must equal `price`, `token_id` must be the market's YES/NO token for `side`, `taker` must be the zero address, `fee_rate_bps` must be `0`, `maker` must be your Safe and `signer` your EOA — so a tampered order is rejected, never placed.

**EIP-712 domain:** `{ name: "Polymarket CTF Exchange", version: "1", chainId, verifyingContract: exchange_address }`.

**`Order` struct fields:** `salt` (uint256), `maker` (address — your Safe), `signer` (address — your EOA), `taker` (address — `0x0`), `tokenId` (uint256 — YES or NO token), `makerAmount` (uint256 — collateral you provide, 6-dec USDC), `takerAmount` (uint256 — shares, 6-dec), `expiration` (uint256 — unix seconds), `nonce` (uint256), `feeRateBps` (uint256 — `0`), `side` (uint8 — `0` = BUY), `signatureType` (uint8 — `2` = POLY_GNOSIS_SAFE).

### Parameters

| Name         | Type   | Description                                              | Required |
| ------------ | ------ | ------------------------------------------------------- | -------- |
| market       | string | Market id                                               | Yes      |
| side         | string | `yes` or `no`                                           | Yes      |
| ord_type     | string | `limit` or `market`                                     | Yes      |
| volume       | string | Number of shares                                        | Yes      |
| price        | string | Limit price `0.01`–`0.99`                               | Yes      |
| signed_order | object | The EIP-712-signed `Order` (fields above)               | Yes      |

### Responses

| Code | Meaning |
| ---- | ------- |
| 201  | Signed order accepted and placed |
| 422  | `signed_order with signature is required` or signature/validation failure |

## Funding your wallet

Your trading collateral stays in **your own wallet**, never in Calibri's custody. To fund:

1. Read your Safe/proxy address (`proxy_address`) from [CTF config](#ctf-config).
2. Send **USDC to that address** on-chain (a normal ERC-20 transfer).

Calibri's on-chain deposit watcher detects the transfer and credits your trading balance — the **same** balance custodial users hold — so the funds are immediately usable on the shared order book. The deposit itself is an on-chain action you control; no API call is required for it.

## Matching & settlement

Non-custodial orders match and settle **entirely on-chain** via three contracts:

- **CTF Exchange** (`exchange_address` from `ctf-config`) — matches your signed order against a counterparty (`matchOrders`). A **mixed match** (a non-custodial maker against a custodial taker) is bridged by Calibri's operator acting as the on-chain counterparty for the custodial leg.
- **Gnosis Conditional Tokens** — your YES/NO positions are **ERC-1155 outcome shares** minted/transferred at match time. A complete YES + NO set is always worth `1.00` USDC.
- **PythiaResolver** — at event resolution Calibri reports the outcome on-chain; the winning outcome token redeems for `1.00` USDC per share (minus fee), the losing token for `0.00`.

Because matching is on-chain, an order only fills when its on-chain transaction succeeds — there is no off-chain promise of backing. The maker rebate and any winnings are credited to your balance the same as for custodial trades.

## Withdrawing

Your funds remain in **your own Safe/proxy** the whole time, so you withdraw by moving USDC out of it on-chain — you never need Calibri's permission or co-operation. When you withdraw directly from your wallet, Calibri's watcher reconciles your off-chain trading balance (debits it and cancels any resting orders the withdrawn funds were collateralising).

Only your **available (uncommitted)** balance is freely withdrawable. Collateral locked in an **open (matched) contract** is held until that contract settles.

## Trustless recovery (the 72-hour deadman)

The one thing you trust Calibri for is reporting the **correct outcome on time**. That trust is bounded by a **deadman switch** in `PythiaResolver`:

- If Calibri fails to report a market's outcome by its scheduled resolution time **plus a grace window (default 72 hours)**, the resolver's `forceVoid` becomes **permissionless** — *anyone* can call it to void the market.
- A **voided** market returns every position's collateral (no winners, no losers), redeemable directly from the on-chain contracts.

So even if Calibri goes offline permanently, your funds are always recoverable: you either get paid the correct outcome, or — once the deadman elapses — you reclaim your collateral straight from the contracts. This is the liveness guarantee that makes the model genuinely non-custodial.

## Worked example: a non-custodial trade end-to-end

A complete walk-through — buy **100 YES shares at 0.60** on market `12` from a self-custody wallet. Replace addresses, ids, and amounts with your own.

**Step 1 — Sign in with your wallet (SIWE).** Request a nonce, sign the EIP-4361 message with your wallet, and exchange it for a session cookie.

```shell
# 1a. get a nonce (also sets a temporary cookie)
curl -X POST https://app.calibri.io/api/v2/persona/identity/sessions/web3/nonce \
  -c cookies.txt -H 'content-type: application/json' \
  -d '{"address":"0xE1188438C85C98aADFf1a88f97764E607B4978C3"}'
# → { "nonce": "9f2c..." }

# 1b. sign the SIWE message in your wallet, then exchange it for a session
curl -X POST https://app.calibri.io/api/v2/persona/identity/sessions/web3 \
  -b cookies.txt -c cookies.txt -H 'content-type: application/json' \
  -d '{"message":"<EIP-4361 message>","signature":"0x<siwe signature>"}'
# → { "uid": "CC4BF46C", "csrf_token": "..." }   (session cookie now in cookies.txt)
```

**Step 2 — Fetch the market's CTF config.** This returns your Safe, the exchange, the YES/NO token ids, and any one-time approvals.

```shell
curl https://app.calibri.io/api/v2/atlas/account/predictions/markets/12/ctf-config \
  -b cookies.txt
```

```json
{
  "exchange_address": "0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9",
  "chain_id": 137,
  "yes_token_id": "7138...",
  "no_token_id": "4519...",
  "proxy_address": "0x0DECE7d83f8D47CD8dD8278c0F22c361C58577BD",
  "signature_type": 2,
  "safe_setup": { "ready": false, "calls": [ { "to": "0x...", "data": "0x...", "operation": 0, "nonce": 0, "hash": "0x9ab..." } ] }
}
```

**Step 3 — One-time: make the Safe trade-ready.** Only when `safe_setup.ready` is `false`. Sign each `call.hash` in your wallet and relay it (Calibri pays the gas).

```shell
curl -X POST https://app.calibri.io/api/v2/atlas/account/predictions/ctf/relay \
  -b cookies.txt -H 'content-type: application/json' \
  -d '{"to":"0x...","data":"0x...","operation":0,"signature":"0x<sig over call.hash>"}'
```

**Step 4 — Fund (one-time / top-up).** Send USDC on-chain to your `proxy_address`. Calibri's watcher credits your trading balance — no API call needed.

**Step 5 — Build and sign the EIP-712 order.** `maker` = your Safe, `signer` = your EOA. For a YES buy, use `yes_token_id`; amounts are 6-decimal USDC (`0.60 × 100 = 60` USDC collateral for `100` shares).

```javascript
import { privateKeyToAccount } from 'viem/accounts'
const account = privateKeyToAccount('0x<your-eoa-key>')

const order = {
  salt:          BigInt('0x' + crypto.randomBytes(8).toString('hex')),
  maker:         cfg.proxy_address,                                  // your Safe
  signer:        account.address,                                   // your EOA
  taker:         '0x0000000000000000000000000000000000000000',
  tokenId:       BigInt(cfg.yes_token_id),                          // YES for a YES buy
  makerAmount:   60_000000n,                                        // 0.60 × 100, 6-dec USDC
  takerAmount:   100_000000n,                                       // 100 shares, 6-dec
  expiration:    BigInt(Math.floor(Date.now()/1000) + 7*86400),
  nonce:         0n,
  feeRateBps:    0n,
  side:          0,                                                 // 0 = BUY
  signatureType: cfg.signature_type,                               // 2 = POLY_GNOSIS_SAFE
}

const signature = await account.signTypedData({
  domain: { name: 'Polymarket CTF Exchange', version: '1',
            chainId: cfg.chain_id, verifyingContract: cfg.exchange_address },
  types:  { Order: [
    { name:'salt', type:'uint256' }, { name:'maker', type:'address' },
    { name:'signer', type:'address' }, { name:'taker', type:'address' },
    { name:'tokenId', type:'uint256' }, { name:'makerAmount', type:'uint256' },
    { name:'takerAmount', type:'uint256' }, { name:'expiration', type:'uint256' },
    { name:'nonce', type:'uint256' }, { name:'feeRateBps', type:'uint256' },
    { name:'side', type:'uint8' }, { name:'signatureType', type:'uint8' } ] },
  primaryType: 'Order',
  message: order,
})
```

**Step 6 — Submit the signed order.** Field names in `signed_order` are snake_case.

```shell
curl -X POST https://app.calibri.io/api/v2/atlas/account/predictions/orders \
  -b cookies.txt -H 'content-type: application/json' \
  -d '{
    "market":"12", "side":"yes", "ord_type":"limit", "volume":"100", "price":"0.60",
    "signed_order": {
      "salt":"5821...", "maker":"0x0DECE7...", "signer":"0xE11884...",
      "taker":"0x0000000000000000000000000000000000000000",
      "token_id":"7138...", "maker_amount":"60000000", "taker_amount":"100000000",
      "expiration":"1788000000", "nonce":"0", "fee_rate_bps":"0",
      "side":0, "signature_type":2, "signature":"0x..."
    }
  }'
# → 201 { "id": 305, "state": "wait", "remaining_volume": "100", ... }
```

**Step 7 — Track it.** Your order rests on the shared book until matched; matching and settlement then happen on-chain.

```shell
curl https://app.calibri.io/api/v2/pythia/orders/305 -b cookies.txt           # order state
curl https://app.calibri.io/api/v2/atlas/account/predictions/contracts -b cookies.txt  # your positions
```

When the event resolves, the resolver reports the outcome on-chain and your winning shares redeem to `1.00` USDC each (minus fee). If Calibri never resolves, recover your collateral via the [72-hour deadman](#trustless-recovery-the-72-hour-deadman).

# Public

## <span class="request-type__get">GET</span> Alive

`/public/health/alive`

### Description

Check that the trade engine is running. Occasionally the trade engine will be temporarily taken offline for udpates and general maintenance. Maintenence will usually not take longer than a few minutes.

### Responses

| Code | Description                                   |
| ---- | --------------------------------------------- |
| 200  | Trade engine is running, trading is available |

# Account

## <span class="request-type__get">GET</span> Get transaction history

`/account/transactions`

### Description

Retrieve your complete transaction history with flexible filtering options. This endpoint returns all types of transactions including deposits, withdrawals, trades, internal transfers, referrals, and maker rewards.

Transactions can be filtered by currency (matches both credit and debit currencies), transaction type, time range, and specific transaction ID. Results are paginated and can be ordered by creation time.

### Parameters

| Name      | Located in | Description                                                                                                 | Required | Schema  |
| --------- | ---------- | ----------------------------------------------------------------------------------------------------------- | -------- | ------- |
| currency  | query      | Filter by currency code (matches credit or debit currency, e.g., "btc", "usdc")                             | No       | string  |
| type      | query      | Filter by transaction type: 'deposit', 'withdraw', 'internal_transfer', 'trade', 'referral', 'maker_reward' | No       | string  |
| txid      | query      | Filter by specific transaction ID                                                                           | No       | string  |
| time_from | query      | Start timestamp (Unix timestamp in seconds)                                                                 | No       | integer |
| time_to   | query      | End timestamp (Unix timestamp in seconds)                                                                   | No       | integer |
| order_by  | query      | Sort order: 'asc' or 'desc' (default: 'desc')                                                               | No       | string  |
| page      | query      | Page number (default: 1, min: 1)                                                                            | No       | integer |
| limit     | query      | Results per page (default: 50, min: 1, max: 1000)                                                           | No       | integer |

### Response Headers

| Header   | Description                  |
| -------- | ---------------------------- |
| Total    | Total number of transactions |
| Page     | Current page number          |
| Per-Page | Number of results per page   |

### Responses

| Code | Description                   | Schema                          |
| ---- | ----------------------------- | ------------------------------- |
| 200  | Transaction history retrieved | [ [Transaction](#transaction) ] |
| 401  | Authentication required       | Error array                     |
| 422  | Validation error              | Error array                     |

### Example Request

```javascript
const axios = require("axios");
const generateHeaders = require("./generateHeaders");

getTransactions = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "GET";
  const path = `/api/v2/account/transactions?currency=btc&type=deposit&limit=25&page=1`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};

getTransactions();
```

## <span class="request-type__post">POST</span> Create new beneficiary

`/account/beneficiaries`

### Description

Create a new beneficiary (a saved withdrawal destination — a cryptocurrency wallet address). All beneficiaries created via the API are automatically activated and ready for use.

### Parameters

| Name               | Located in | Description                                                                                   | Required | Schema  | Max Length |
| ------------------ | ---------- | --------------------------------------------------------------------------------------------- | -------- | ------- | ---------- |
| currency           | formData   | Currency code (e.g., "btc", "eth", "usdc", "usdt", "xrp", "sol")                              | Yes      | string  | 10 chars   |
| name               | formData   | Display name for the beneficiary                                                              | Yes      | string  | 64 chars   |
| description        | formData   | Optional notes about the beneficiary                                                          | No       | string  | 255 chars  |
| blockchain_key     | formData   | Blockchain identifier (defaults to currency's blockchain if omitted)                          | No       | string  | 255 chars  |
| withdraw_method_id | formData   | Specific withdrawal method ID (auto-selected if omitted)                                      | No       | integer | -          |
| data               | formData   | Address/account details in JSON format (structure varies by currency type - see tables below) | Yes      | json    | -          |

Cryptocurrency `data` Object Fields

| Field          | Type    | Required | Description                                                             |
| -------------- | ------- | -------- | ----------------------------------------------------------------------- |
| address        | string  | Yes      | Cryptocurrency wallet address                                           |
| tag            | string  | No       | Destination tag (for XRP, Stellar, etc.) - max: 4294967295              |

### Request Examples

**Example 1: Crypto - Bitcoin**

```json
{
  "currency": "btc",
  "name": "My Bitcoin Wallet",
  "description": "Personal cold storage wallet",
  "data": {
    "address": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq"
  }
}
```

**Example 2: Crypto - XRP with Destination Tag**

```json
{
  "currency": "xrp",
  "name": "Exchange XRP Account",
  "description": "My Binance XRP deposit",
  "data": {
    "address": "rN7n7otQDd6FczFgLdlqtyMVrn3qHwuSaDcqC",
    "tag": "123456789"
  }
}
```

**Example 3: Crypto - USDC**

```json
{
  "currency": "usdc",
  "name": "My USDC Wallet",
  "description": "Settlement wallet",
  "data": {
    "address": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e"
  }
}
```

### Important Notes

- **API-created beneficiaries** are automatically activated (state = `active`) and do not require PIN activation
- **Cryptocurrency addresses** are validated against the blockchain service before creation
- **XRP destination tags** must be unsigned int32 (maximum: 4294967295)
- **Duplicate addresses** are not allowed for the same currency/blockchain combination

### Responses

| Code | Description            | Schema                      |
| ---- | ---------------------- | --------------------------- |
| 201  | Create new beneficiary | [Beneficiary](#beneficiary) |

## <span class="request-type__patch">PATCH</span> Activate beneficiary

`/account/beneficiaries/{id}/activate`

### Description

Activates a new beneficiary with pin emailed during beneficiary creation.

**Note:** This activation step is only required for beneficiaries created through the web interface. Beneficiaries created via the API are automatically activated and do not require this step.

### Parameters

| Name | Located in | Description                         | Required | Schema  |
| ---- | ---------- | ----------------------------------- | -------- | ------- |
| id   | path       | Beneficiary Identifier              | Yes      | integer |
| pin  | formData   | Pin code for beneficiary activation | Yes      | integer |

### Responses

| Code | Description                    | Schema                      |
| ---- | ------------------------------ | --------------------------- |
| 200  | Activates beneficiary with pin | [Beneficiary](#beneficiary) |

## <span class="request-type__delete">DELETE</span> Delete beneficiary

` /account/beneficiaries/{id}`

### Description

Delete beneficiary

### Parameters

| Name | Located in | Description            | Required | Schema  |
| ---- | ---------- | ---------------------- | -------- | ------- |
| id   | path       | Beneficiary Identifier | Yes      | integer |

### Responses

| Code | Description        |
| ---- | ------------------ |
| 204  | Delete beneficiary |

## <span class="request-type__get">GET</span> Beneficiary by id

`/account/beneficiaries/{id}`

### Description

Get beneficiary by id

### Parameters

| Name | Located in | Description            | Required | Schema  |
| ---- | ---------- | ---------------------- | -------- | ------- |
| id   | path       | Beneficiary Identifier | Yes      | integer |

### Responses

| Code | Description           | Schema                      |
| ---- | --------------------- | --------------------------- |
| 200  | Get beneficiary by id | [Beneficiary](#beneficiary) |

## <span class="request-type__get">GET</span> List Beneficiaries

`/account/beneficiaries`

### Description

Get list of user beneficiaries

### Parameters

| Name     | Located in | Description                                                                                                                 | Required | Schema |
| -------- | ---------- | --------------------------------------------------------------------------------------------------------------------------- | -------- | ------ |
| currency | query      | Beneficiary currency code.                                                                                                  | No       | string |
| state    | query      | Defines either beneficiary active - user can use it to withdraw moneyor pending - requires beneficiary activation with pin. | No       | string |

### Responses

| Code | Description                    | Schema                          |
| ---- | ------------------------------ | ------------------------------- |
| 200  | Get list of user beneficiaries | [ [Beneficiary](#beneficiary) ] |

## <span class="request-type__post">GET</span> Withdraw quote

`/account/withdraws/quote/{currency}`

`/account/withdraws/quote/{currency}?bolt=ln...`

### Description

Get the expected fee to withdraw the specified currency. Submit the quote_id with your withdraw request payload to honor that fee during the withdrawal. Quotes are currently valid for 90 seconds, ensure you submit the withdrawal request before the expires_at parameter.

### Parameters

| Name     | Located in | Description                                                                      | Required | Schema  |
| -------- | ---------- | -------------------------------------------------------------------------------- | -------- | ------- |
| currency | path       | Currency code.                                                                   | Yes      | string  |
| bolt     | query      | For BTC Lightning payments, supply the Bolt11 invoice. Fees are returned in BTC. | No       | integer |

### Responses

| Code | Description              | Schema                                                               |
| ---- | ------------------------ | -------------------------------------------------------------------- |
| 200  | Returns a withdraw quote | [ [Withdraw quote](#withdraw-quote) ]                                |
| 422  | Unproccesable entity     | Error array containing the reason why the quote could not be created |

## <span class="request-type__post">POST</span> New withdrawal

`/account/withdraws`

### Description

Creates new withdrawal to active beneficiary.

**Note:** OTP is **not required** when creating withdrawals via the API. The OTP requirement only applies to withdrawals created through the web app, however 2FA must still be enabled on the account to enable withdraws.

### Parameters

| Name           | Located in | Description                                                                                                                                                   | Required | Schema  |
| -------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------- |
| beneficiary_id | formData   | ID of active Beneficiary.                                                                                                                                     | Yes      | integer |
| currency       | formData   | The currency code matching the beneficiary.                                                                                                                   | Yes      | string  |
| amount         | formData   | The amount to withdraw.                                                                                                                                       | Yes      | double  |
| note           | formData   | Optional note to attach to the withdrawal.                                                                                                                   | No       | string  |

### Responses

| Code | Description             |
| ---- | ----------------------- |
| 201  | Creates new withdrawal. |

## <span class="request-type__post">GET</span> Withdrawal history

`/account/withdraws`

### Description

Get your withdraw history, paginated.

### Parameters

| Name     | Located in | Description                                                     | Required | Schema  |
| -------- | ---------- | --------------------------------------------------------------- | -------- | ------- |
| currency | query      | Currency code.                                                  | No       | string  |
| limit    | query      | Number of withdraws per page (defaults to 100, maximum is 100). | No       | integer |
| state    | query      | Filter by withdraw state. [ [Withdraw states](#states) ]        | No       | string  |
| rid      | query      | Destination address (blockchain withdraws only).                | No       | string  |
| page     | query      | Page number (defaults to 1).                                    | No       | integer |

### Responses

| Code | Description          | Schema                    |
| ---- | -------------------- | ------------------------- |
| 200  | List your withdraws. | [ [Withdraw](#withdraw) ] |

## <span class="request-type__get">GET</span> Get deposit methods

`/api/v2/payments/deposits/methods`

### Description

Get all available deposit methods for the authenticated user. Returns the supported cryptocurrency deposit methods, filtered based on your remaining transaction limits.

Each method includes minimum and maximum amounts, fees, provider information, and remaining limits for different time periods (daily, weekly, monthly, yearly).

### Responses

| Code | Description                         | Schema                                |
| ---- | ----------------------------------- | ------------------------------------- |
| 200  | Array of available deposit methods. | [ [Deposit Method](#deposit-method) ] |

## <span class="request-type__get">GET</span> Get withdraw methods

`/api/v2/payments/withdraws/methods`

### Description

Get all available withdrawal methods for the authenticated user. Returns the supported cryptocurrency withdrawal methods, filtered based on your remaining transaction limits.

Each method includes minimum and maximum amounts, fees, provider information, processor details, and remaining limits for different time periods (daily, weekly, monthly, yearly).

### Responses

| Code | Description                            | Schema                                  |
| ---- | -------------------------------------- | --------------------------------------- |
| 200  | Array of available withdrawal methods. | [ [Withdraw Method](#withdraw-method) ] |

## <span class="request-type__get">GET</span> Deposit address

`/account/deposit_address/{currency}`

### Description

Fetch the deposit address for the specified currency.

Note: If your currency deposit address has not been created yet, the result may return empty while the address is being created. If it is empty, please try again in a few seconds.

### Parameters

| Name     | Located in | Description              | Required | Schema |
| -------- | ---------- | ------------------------ | -------- | ------ |
| currency | path       | The currency to receive. | Yes      | string |

### Responses

| Code | Description                                                                        | Schema              |
| ---- | ---------------------------------------------------------------------------------- | ------------------- |
| 200  | Deposit address. (May be empty on first request for new accounts, see note above). | [Deposit](#deposit) |

## <span class="request-type__get">GET</span> Deposit by ID

`/account/deposits/{id}`

### Description

Get details of a specific deposit by its internal deposit ID.

### Parameters

| Name | Located in | Description | Required | Schema  |
| ---- | ---------- | ----------- | -------- | ------- |
| id   | path       | Deposit ID  | Yes      | integer |

### Responses

| Code | Description                      | Schema              |
| ---- | -------------------------------- | ------------------- |
| 200  | Get details of specific deposit. | [Deposit](#deposit) |

## <span class="request-type__get">GET</span> Deposit history

`/account/deposits`

### Description

Get your deposit history. You can filter by specific deposit transaction using the `txid` query parameter, or fetch all deposits with optional filters for currency and state

### Parameters

| Name     | Located in | Description                                                    | Required | Schema  |
| -------- | ---------- | -------------------------------------------------------------- | -------- | ------- |
| currency | query      | Currency code                                                  | No       | string  |
| state    | query      | Filter deposits by deposit state [States](#states)             | No       | string  |
| txid     | query      | Deposit transaction id.                                        | No       | string  |
| limit    | query      | Number of deposits per page (defaults to 100, maximum is 100). | No       | integer |
| page     | query      | Page number (defaults to 1).                                   | No       | integer |

### Responses

| Code | Description               | Schema                  |
| ---- | ------------------------- | ----------------------- |
| 200  | Get your deposit history. | [ [Deposit](#deposit) ] |

## <span class="request-type__get">GET</span> Balance

`/account/balances/{currency}`

### Description

Get single account balance by currency code eg. 'btc'

### Parameters

| Name     | Located in | Description        | Required | Schema |
| -------- | ---------- | ------------------ | -------- | ------ |
| currency | path       | The currency code. | Yes      | string |

### Responses

| Code | Description                     | Schema              |
| ---- | ------------------------------- | ------------------- |
| 200  | Get account balance by currency | [Account](#account) |

## <span class="request-type__get">GET</span> All balances

`/account/balances`

### Description

Get account balances for all currencies

### Parameters

| Name    | Located in | Description                                                | Required | Schema  |
| ------- | ---------- | ---------------------------------------------------------- | -------- | ------- |
| limit   | query      | Limit the number of returned paginations. Defaults to 100. | No       | integer |
| page    | query      | Paginated page number.                                     | No       | integer |
| nonzero | query      | Filter non zero balances.                                  | No       | Boolean |

### Responses

| Code | Description       | Schema                  |
| ---- | ----------------- | ----------------------- |
| 200  | Array of balances | [ [Account](#account) ] |

# Sub-Accounts

**Note:** Sub-account endpoints use the `/api/v2/persona` base path instead of `/api/v2/atlas`.

**Sub-Account actions:** To make API requests for a sub-account (e.g., placing trades, checking balances), include the `X-Auth-Subaccount-Uid` header with the sub-account's UID. Your parent account's API credentials are used for authentication, but operations are performed in the sub-account's context.

## <span class="request-type__post">POST</span> Create Sub-Account

```javascript
// createSubAccount.js
const axios = require("axios");
const generateHeaders = require("./generateHeaders");

createSubAccount = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "POST";
  const path = `/api/v2/persona/resource/sub_accounts`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const subAccountData = {
    account_name: "Trading Account 1",
    deposit_action: "transfer_to_parent",
  };

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
    data: subAccountData,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};

createSubAccount();
```

`/persona/resource/sub_accounts`

### Description

Create a new sub-account managed by the parent account. Sub-accounts cannot login independently and are managed by the parent account.

### Parameters

| Name           | Located in | Description                                                                        | Required | Schema |
| -------------- | ---------- | ---------------------------------------------------------------------------------- | -------- | ------ |
| account_name   | formData   | Display name for the sub-account (1-100 characters)                                | Yes      | string |
| deposit_action | formData   | JSON object defining automated deposit handling (see deposit action options below) | No       | json   |
| data           | formData   | Optional additional data in JSON format                                            | No       | json   |

**Deposit Action Options:**

Configure automatic actions for deposits received by this sub-account.

The `deposit_action` parameter accepts a JSON object with the following structure:

```json
{
  "action": "none|aggregate|convert_and_withdraw|aggregate_convert_and_withdraw",
  "target_currency": "currency_code", // Required for convert actions (e.g., "usdc", "btc")
  "beneficiary_id": 123, // Required for withdraw actions
  "lightning_invoice": "lnbc...", // Alternative to beneficiary_id for Lightning withdrawals
  "max_slippage": 2.5, // Optional: Max slippage percentage (0.1-10, default: 2.0)
  "beneficiary_amount": "50.00" // Optional: Exact amount the beneficiary should receive
}
```

**Beneficiary Amount:**

When `beneficiary_amount` is specified (as a string), it defines the exact amount that the beneficiary should receive in the target currency. If, after conversion, the account does not have sufficient funds to cover this amount, the withdrawal will **not** be placed. Instead, the payment will be marked as `completed`, with the converted funds remaining in the account.

**Note:** When using `lightning_invoice` together with `beneficiary_amount`, the invoice amount must match the `beneficiary_amount`.

**Available Actions:**

| Action                           | Description                                                    | Required Fields                                            |
| -------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------- |
| `none`                           | No automatic action (default)                                  | None                                                       |
| `aggregate`                      | Automatically transfer deposits to parent account              | None                                                       |
| `convert_and_withdraw`           | Convert deposit to target currency and withdraw to beneficiary | `target_currency`, `beneficiary_id` or `lightning_invoice` |
| `aggregate_convert_and_withdraw` | Transfer to parent, then convert and withdraw                  | `target_currency`, `beneficiary_id` or `lightning_invoice` |

**Example with Simple Aggregate:**

```json
{
  "account_name": "Trading Account 1",
  "deposit_action": {
    "action": "aggregate"
  }
}
```

**Example with Convert and Withdraw:**

```json
{
  "account_name": "USDC Conversion Account",
  "deposit_action": {
    "action": "convert_and_withdraw",
    "target_currency": "usdc",
    "beneficiary_id": 789,
    "max_slippage": 2.5
  }
}
```

### Responses

| Code | Description                      | Schema                    |
| ---- | -------------------------------- | ------------------------- |
| 201  | Sub-account created successfully | [SubAccount](#subaccount) |
| 422  | Validation error                 | Error array               |

## <span class="request-type__get">GET</span> List Sub-Accounts

```javascript
// listSubAccounts.js
const axios = require("axios");
const generateHeaders = require("./generateHeaders");

listSubAccounts = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "GET";
  const path = `/api/v2/persona/resource/sub_accounts?page=1&limit=100`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};

listSubAccounts();
```

`/persona/resource/sub_accounts`

### Description

Get a paginated list of the user's sub-accounts.

### Parameters

| Name  | Located in | Description                                          | Required | Schema  |
| ----- | ---------- | ---------------------------------------------------- | -------- | ------- |
| page  | query      | Page number (defaults to 1)                          | No       | integer |
| limit | query      | Number of results per page (default: 100, max: 1000) | No       | integer |

### Responses

| Code | Description          | Schema                        |
| ---- | -------------------- | ----------------------------- |
| 200  | List of sub-accounts | [ [SubAccount](#subaccount) ] |

## <span class="request-type__get">GET</span> Get Sub-Account by UID

```javascript
// getSubAccount.js
const axios = require("axios");
const generateHeaders = require("./generateHeaders");

getSubAccount = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "GET";
  const subAccountUid = "CCT1234567";
  const path = `/api/v2/persona/resource/sub_accounts/${subAccountUid}`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};

getSubAccount();
```

`/persona/resource/sub_accounts/{uid}`

### Description

Retrieve a specific sub-account by its unique identifier.

### Parameters

| Name | Located in | Description            | Required | Schema |
| ---- | ---------- | ---------------------- | -------- | ------ |
| uid  | path       | Sub-account identifier | Yes      | string |

### Responses

| Code | Description       | Schema                    |
| ---- | ----------------- | ------------------------- |
| 200  | Sub-account found | [SubAccount](#subaccount) |
| 404  | Not found         | Error array               |

## <span class="request-type__put">PUT</span> Update Sub-Account

```javascript
// updateSubAccount.js
const axios = require("axios");
const generateHeaders = require("./generateHeaders");

updateSubAccount = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "PUT";
  const subAccountUid = "CCT1234567";
  const path = `/api/v2/persona/resource/sub_accounts/${subAccountUid}`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const updateData = {
    deposit_action: "none",
  };

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
    data: updateData,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};

updateSubAccount();
```

`/persona/resource/sub_accounts/{uid}`

### Description

Update sub-account settings.

### Parameters

| Name           | Located in | Description                                         | Required | Schema |
| -------------- | ---------- | --------------------------------------------------- | -------- | ------ |
| uid            | path       | Sub-account identifier                              | Yes      | string |
| deposit_action | formData   | Action for deposits: 'none' or 'transfer_to_parent' | No       | string |

### Responses

| Code | Description                      | Schema                    |
| ---- | -------------------------------- | ------------------------- |
| 200  | Sub-account updated successfully | [SubAccount](#subaccount) |
| 404  | Not found                        | Error array               |
| 422  | Validation error                 | Error array               |

## <span class="request-type__delete">DELETE</span> Delete Sub-Account

```javascript
// deleteSubAccount.js
const axios = require("axios");
const generateHeaders = require("./generateHeaders");

deleteSubAccount = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "DELETE";
  const subAccountUid = "CCT1234567";
  const path = `/api/v2/persona/resource/sub_accounts/${subAccountUid}`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const axiosConfig = {
    method: method,
    url: url,
    headers: headers,
  };

  try {
    const results = await axios(axiosConfig);
    console.log("Sub-account deleted successfully");
  } catch (err) {
    console.log(err);
  }
};

deleteSubAccount();
```

`/persona/resource/sub_accounts/{uid}`

### Description

Soft delete a sub-account (sets state to 'deleted'). The sub-account will no longer be accessible.

### Parameters

| Name | Located in | Description            | Required | Schema |
| ---- | ---------- | ---------------------- | -------- | ------ |
| uid  | path       | Sub-account identifier | Yes      | string |

### Responses

| Code | Description                      |
| ---- | -------------------------------- |
| 204  | Sub-account deleted successfully |
| 404  | Not found                        |

# Websocket

```javascript
// generateHeaders.js

const crypto = require("crypto");
const apiKey = "123456...";
const secret = "ABCDEF...";

module.exports = async () => {
  const timeStamp = new Date().getTime();
  const queryString = `${timeStamp}${apikey}`;
  const signature = crypto
    .createHmac("sha256", secret)
    .update(queryString)
    .digest("hex");

  return {
    "Content-Type": "application/json;charset=utf-8",
    "X-Auth-Apikey": apikey,
    "X-Auth-Nonce": timeStamp,
    "X-Auth-Signature": signature,
  };
};
```

The high performance websocket provides near realtime updates on the orderbook, your orders and balances.

The orderbook endpoint does not require authenticaiton, while the private endpoints (orders, trades, and balances) do require authentication.

To connect to the private endpoints on the websocket, send the authentication headers with the connection request, similar to the REST authentication, and subscribe to the streams you are interested in.
You can combine multiple subscriptions in one request, and also provide the subscriptions at the same time as connecting - ie. you only need 1x request to get fully connected.

You can also combine the public and private streams into one so that you only need to maintain 1x connection for example `?stream=12.ob-inc&stream=order&stream=trade&stream=balance`

## <span class="request-type__get">SUBSCRIBE</span> Orderbook

```javascript
const WebSocket = require("ws");

async function connectWssPublic() {
  try {
    const baseUrl = `wss://app.calibri.io/api/v2`;
    const streams = "?stream=12.ob-inc&stream=15.ob-inc";
    const path = `/websocket/public${streams}`;

    ws = new WebSocket(`${baseUrl}${path}`);
  } catch (err) {
    console.log(err);
  }

  ws.on("open", () => {
    console.log("Connected to websocket");
  });

  ws.on("message", function message(data) {
    data = JSON.parse(data);
    console.log(JSON.stringify(data, undefined, 2));

    if (["ob-snap", "ob-inc"].includes(data)) {
      // handle all subscribed markets
      // updateOrderBook(data);
    }
  });
}

connectWssPublic();
```

`?stream={market}.ob-inc`

### Description

Subscribe to orderbook updates for the specified market. You can subscribe to multiple markets at the same time by adding each market as a param, eg

`?stream=12.ob-inc&15.ob-inc`

Note: if the amount is empty "", the order has been removed from the orderbook

### Parameters

| Name            | Located in | Description                        | Required | Schema |
| --------------- | ---------- | ---------------------------------- | -------- | ------ |
| {market}-ob.inc | query      | Name of the market to subscribe to | yes      | string |

### Responses

| Description | Schema                                          |
| ----------- | ----------------------------------------------- |
| snapshot    | [ [Snapshot](#websocket-orderbook-snapshot) ]   |
| increment   | [ [Increment](#websocket-orderbook-increment) ] |

## <span class="request-type__get">SUBSCRIBE</span> K-Line

```javascript
const WebSocket = require("ws");

async function connectWssPublic() {
  try {
    const baseUrl = `wss://app.calibri.io/api/v2`;
    const streams = "?stream=12.kline-15m";
    const path = `/websocket/public${streams}`;

    ws = new WebSocket(`${baseUrl}${path}`);
  } catch (err) {
    console.log(err);
  }

  ws.on("open", () => {
    console.log("Connected to websocket");
  });

  ws.on("message", function message(data) {
    data = JSON.parse(data);
    console.log(JSON.stringify(data, undefined, 2));

    if (["kline-"].includes(data)) {
      // handle all subscribed markets
      // updateOrderBook(data);
    }
  });
}

connectWssPublic();
```

`?stream={market}.kline-{period}`

### Description

Subscribe to the K-LINE (OHLC) for the specified market to get updates on the OHLC for each period. Similary to the otherendpoints, you can combine streams into a single subscription request.

### Parameters

| Name     | Located in | Description                                                                                            | Required | Schema |
| -------- | ---------- | ------------------------------------------------------------------------------------------------------ | -------- | ------ |
| {market} | query      | Name of the market to subscribe to                                                                     | yes      | string |
| {period} | query      | The period to get updates at "1m", "5m", "15m", "30m", "1h", "2h", "4h", "6h", "12h", "1d", "3d", "1w" | yes      | string |

### Responses

| Name                    | Type     | Description                                        |
| ----------------------- | -------- | -------------------------------------------------- |
| {market}.kline-{period} | object   | Key name of the OHLC for the market and period     |
| OHLC for that period    | [number] | Array of Timestamp, Open, High, Low, Close, Volume |

## <span class="request-type__get">SUBSCRIBE</span> Private

```javascript
async function connectWssPrivate() {
  try {
    const baseUrl = `wss://app.calibri.io/api/v2`;
    const headers = await generateHeaders();
    const streams =
      "?stream=12.ob-inc&stream=15.ob-inc&stream=order&stream=trade&stream=balance";
    const path = `/websocket/private${streams}`;

    ws = new WebSocket(`${baseUrl}${path}`, { headers });
  } catch (err) {
    console.log(err);
  }

  ws.on("open", () => {
    console.log("Connected to websocket");
  });

  ws.on("message", (data) => {
    data = JSON.parse(data);
    console.log(JSON.stringify(data, undefined, 2));

    if (data["12.ob-snap"] || data["12.ob-inc"]) {
      // handle orderbook updates for subscribed markets
      // updateOrderBook(data);
    }
    if (data["order"]) {
      // handleOwnOrderUpdate(data);
    }
    if (data["trade"]) {
      // handleOwnTradeUpdate(data);
    }
    if (data["balance"]) {
      // handleBalanceUpdate(data);
    }
  });
}

connectWssPrivate();
```

Subscribe to your account and receive real-time updates on your orders, trades, and balances.

**Important:** You can subscribe to ALL streams (both public and private) in a single WebSocket connection using the `/websocket/private` endpoint. This is the recommended approach as it minimizes connection overhead and provides a unified stream of all market events.

**Private streams only:**

`?stream=order&stream=trade&stream=balance`

**Combining public orderbook + private streams (recommended):**

`?stream=12.ob-inc&stream=order&stream=trade&stream=balance`

**Benefits of combining streams:**

- Single WebSocket connection for all updates
- Synchronized view of orderbook changes and your order executions
- Real-time balance updates as trades execute
- Reduced latency and connection overhead

### Order Stream

**Subscription:** `?stream=order`

Receive real-time updates when your orders are created, updated, filled, or canceled.

**Response Format:**

```json
{
  "order": {
    "id": 173757,
    "side": "yes",
    "ord_type": "limit",
    "price": "0.60",
    "state": "wait",
    "market": "12",
    "created_at": "2023-12-08T08:46:59+01:00",
    "updated_at": "2023-12-08T08:46:59+01:00",
    "origin_volume": "100",
    "remaining_volume": "100",
    "locked": "60.0",
    "contracts_count": 0
  }
}
```

**Order Update Triggers:**

- Order created and accepted
- Order partially filled
- Order fully filled (state changes to "done")
- Order canceled
- Order rejected

### Trade Stream

**Subscription:** `?stream=trade`

Receive real-time updates when your orders are matched and trades are executed.

**Response Format:**

```json
{
  "trade": {
    "id": 12345,
    "order_id": 173757,
    "price": "0.60",
    "amount": "100",
    "total": "60.0",
    "market": "12",
    "side": "yes",
    "taker_type": "no",
    "fee_currency": "usdc",
    "fee": "0.002",
    "fee_amount": "0.12",
    "created_at": "2023-12-08T08:47:01+01:00"
  }
}
```

**Trade Event Triggers:**

- Your limit order is matched (you are the maker)
- Your market order is executed (you are the taker)
- Your order is partially filled

### Balance Stream

**Subscription:** `?stream=balance`

Receive real-time updates when your account balances change. Balance updates are triggered by trading activity, order placement, and order cancellation.

**Response Format:**

```json
{
  "balance": {
    "currency": "usdc",
    "balance": "10000.00",
    "locked": "60.00",
    "available": "9940.00",
    "updated_at": 1702027621000
  }
}
```

**Response Fields:**

| Field      | Type    | Description                                                      |
| ---------- | ------- | ---------------------------------------------------------------- |
| currency   | string  | Currency code (usdc, btc, eth, usdt, etc.)                       |
| balance    | string  | Total balance (available + locked). String for decimal precision |
| locked     | string  | Amount currently locked in open orders                           |
| available  | string  | Available balance for trading (balance - locked)                 |
| updated_at | integer | Unix timestamp in milliseconds when balance was updated          |

**Balance Update Triggers:**

1. **Order Creation** - Locks USDC collateral when a limit or market order is placed
2. **Order Cancellation** - Unlocks collateral when the unmatched remainder of an order is canceled
3. **Settlement (winning side)** - Credits USDC (`1.00` per winning share, minus fee) when the event resolves
4. **Settlement (losing side)** - Releases the locked collateral (losing shares settle to `0.00`)
5. **Partial Fill** - Each partial fill keeps the matched collateral locked against the new contract
6. **Order Rejection** - Collateral unlock if order validation fails after initial lock

**Example 1: Order Creation (Collateral Lock)**

When you place a YES order for 100 shares @ `0.60` USDC, `60.00` USDC of collateral is locked:

```json
{
  "balance": {
    "currency": "usdc",
    "balance": "10000.00",
    "locked": "60.00",
    "available": "9940.00",
    "updated_at": 1702027621000
  }
}
```

**Example 2: Order Cancellation (Collateral Unlock)**

When you cancel the unmatched remainder, the locked USDC is released:

```json
{
  "balance": {
    "currency": "usdc",
    "balance": "10000.00",
    "locked": "0.00",
    "available": "10000.00",
    "updated_at": 1702027625000
  }
}
```

**Example 3: Settlement (Winning Side)**

When the event resolves in your favour, each winning share redeems to `1.00` USDC (minus the settlement fee) and the locked collateral is released as available balance:

```json
{
  "balance": {
    "currency": "usdc",
    "balance": "10039.80",
    "locked": "0.00",
    "available": "10039.80",
    "updated_at": 1702027630000
  }
}
```

**Example 4: Partial Fill**

When your order is partially filled (50 of 100 shares match), the matched collateral remains locked against the new contract:

```json
{
  "balance": {
    "currency": "usdc",
    "balance": "10000.00",
    "locked": "60.00",
    "available": "9940.00",
    "updated_at": 1702027635000
  }
}
```

**Important Notes:**

- **Atomic Updates**: Balance updates are atomic - you receive one update per currency per event
- **Single Settlement Currency**: Markets are collateralised and settled in **USDC** - balance updates track your USDC collateral and payouts
- **Timestamp Precision**: The `updated_at` field uses **Unix milliseconds** (not seconds)
- **String Values**: Balance values are strings to preserve decimal precision - parse them as decimals, not floats
- **Rapid Updates**: Multiple rapid trades may result in several balance updates in quick succession
- **Fee Inclusion**: Fees are already deducted from the amounts shown in balance updates

**Edge Cases:**

- **Insufficient Funds**: No balance update is sent (order is rejected before locking funds)
- **Self-Trade Prevention**: If a self-trade is prevented, balance may lock and then immediately unlock
- **Partial Fills**: Each partial fill of an order triggers a balance update reflecting the locked collateral
- **Maker rebate vs Taker Fees**: The taker pays the fee; the resting maker earns a rebate (a percentage of the taker fee) credited at settlement

**Integration Example:**

```javascript
ws.on("message", (data) => {
  data = JSON.parse(data);

  if (data["balance"]) {
    const { currency, balance, locked, available, updated_at } = data.balance;

    console.log(`Balance Update: ${currency}`);
    console.log(`  Total: ${balance}`);
    console.log(`  Locked: ${locked}`);
    console.log(`  Available: ${available}`);
    console.log(`  Updated: ${new Date(updated_at).toISOString()}`);

    // Update your UI
    updateBalanceDisplay(currency, available, locked);
  }

  if (data["order"]) {
    // Handle order updates
    handleOrderUpdate(data.order);
  }

  if (data["trade"]) {
    // Handle trade updates - expect balance updates to follow immediately
    handleTradeUpdate(data.trade);
  }
});
```

# Models

### Pagination

All paginated list endpoints return an array of items in the response body, with pagination information provided in HTTP response headers:

| Header   | Description                            | Example |
| -------- | -------------------------------------- | ------- |
| Page     | Current page number                    | 1       |
| Per-Page | Number of items per page               | 100     |
| Total    | Total number of items across all pages | 150     |

### Ticker

| Name   | Type                        | Description                     |
| ------ | --------------------------- | ------------------------------- |
| at     | integer                     | Timestamp of ticker             |
| ticker | [TickerEntry](#tickerentry) | Ticker entry for specified time |

### TickerEntry

| Name                 | Type    | Description                                                                                   |
| -------------------- | ------- | --------------------------------------------------------------------------------------------- |
| low                  | double  | The lowest trade price during last 24 hours (0.0 if no trades executed during last 24 hours)  |
| high                 | double  | The highest trade price during last 24 hours (0.0 if no trades executed during last 24 hours) |
| open                 | double  | Price of the first trade executed 24 hours ago or less                                        |
| last                 | double  | The last executed trade price                                                                 |
| volume               | double  | Total volume of trades executed during last 24 hours                                          |
| amount               | double  | Total amount of trades executed during last 24 hours                                          |
| vol                  | double  | Alias to volume                                                                               |
| avg_price            | double  | Average price more precisely VWAP (volume weighted average price)                             |
| price_change_percent | string  | Price change over the last 24 hours                                                           |
| at                   | integer | Timestamp of ticker                                                                           |

### Orderbook entry

| Name   | Type   | Description                                                                                                           |
| ------ | ------ | --------------------------------------------------------------------------------------------------------------------- |
| price  | number | The price of the order                                                                                                |
| amount | number | The amount of the order at that price. Note: if the amount is empty "", the order has been removed from the orderbook |

### Trade

| Name            | Type    | Description                                             |
| --------------- | ------- | ------------------------------------------------------- |
| id              | string  | Trade ID.                                               |
| price           | double  | Trade price.                                            |
| amount          | double  | Trade amount.                                           |
| total           | double  | Trade total (Amount \* Price).                          |
| fee_currency    | double  | Currency user's fees were charged in.                   |
| fee             | double  | Percentage of fee user was charged for performed trade. |
| fee_amount      | double  | Amount of fee user was charged for performed trade.     |
| market          | string  | Trade market id.                                        |
| created_at      | string  | Trade create time in iso8601 format.                    |
| taker_type      | string  | Trade taker order type (sell or buy).                   |
| side            | string  | Trade side.                                             |
| order_id        | integer | Order id.                                               |
| custom_order_id | integer | Optional custom Order id if supplied.                   |

### Order

| Name             | Type              | Description                                             |
| ---------------- | ----------------- | ------------------------------------------------------- |
| id               | integer           | Unique order id.                                        |
| custom_order_id  | string            | Custom order id if supplied during order creation.      |
| side             | string            | Either 'yes' or 'no'.                                   |
| ord_type         | string            | Type of order, either 'limit' or 'market'.              |
| post_only        | boolean           | If the Limit order was submitted as post_only .         |
| price            | double            | Probability price per share `0.01`–`0.99` (in USDC).    |
| state            | string            | Current state of the order [Order states](#states)      |
| market           | string            | The market in which the order is placed, e.g. '12'.     |
| created_at       | string            | Order create time in iso8601 format.                    |
| updated_at       | string            | Order updated time in iso8601 format.                   |
| origin_volume    | double            | The original number of shares of the order.             |
| remaining_volume | double            | The remaining (unmatched) shares of the order.          |
| filled_volume    | double            | The matched share volume of the order.                  |
| contracts_count  | integer           | Number of contracts formed from this order.             |
| locked           | double            | Collateral currently locked for this order (USDC).      |
| quote_currency   | string            | Settlement currency for the market (e.g. 'usdc').       |

### Market

| Name             | Type   | Description                                  |
| ---------------- | ------ | -------------------------------------------- |
| id               | string | Unique market id eg '12'                      |
| name             | string | Market name.                                 |
| base_unit        | string | Outcome share unit.                          |
| quote_unit       | string | Settlement currency (e.g. 'usdc').           |
| min_price        | double | Minimum order price (probability, e.g. 0.01).|
| max_price        | double | Maximum order price (probability, e.g. 0.99).|
| min_amount       | double | Minimum order size in shares.                |
| amount_precision | double | Precision for order amount.                  |
| price_precision  | double | Precision for order price.                   |
| state            | string | If the market is enabled for trading or not. |

### Currency

| Name                          | Type   | Description                                                           |
| ----------------------------- | ------ | --------------------------------------------------------------------- |
| id                            | string | Currency code.                                                        |
| name                          | string | Currency name                                                         |
| symbol                        | string | Currency symbol                                                       |
| explorer_transaction          | string | Currency transaction exprorer url template                            |
| explorer_address              | string | Currency address exprorer url template                                |
| type                          | string | Currency type                                                         |
| deposit_enabled               | string | Currency deposit possibility status (true/false).                     |
| withdrawal_enabled            | string | Currency withdrawal possibility status (true/false).                  |
| deposit_fee                   | string | Currency deposit fee                                                  |
| min_deposit_amount            | string | Minimal deposit amount                                                |
| withdraw_fee                  | string | Currency withdraw fee                                                 |
| min_withdraw_amount           | string | Minimal withdraw amount                                               |
| withdraw_limit_24h            | string | Currency 24h withdraw limit                                           |
| withdraw_limit_72h            | string | Currency 72h withdraw limit                                           |
| base_factor                   | string | Currency base factor                                                  |
| precision                     | string | Currency precision                                                    |
| position                      | string | Position used for defining currencies order                           |
| icon_url                      | string | Currency icon                                                         |
| min_confirmations             | string | Number of confirmations required for confirming deposit or withdrawal |
| layer_two_withdraw_fee        | string | (Layer two dependant) The base layer two withdraw fee                 |
| layer_two_max_deposit_amount  | string | (Layer two dependant) The maximum deposit amount                      |
| layer_two_max_withdraw_amount | string | (Layer two dependant) The maxumum withdrawal amount                   |

### Withdraw

| Name            | Type    | Description                                             |
| --------------- | ------- | ------------------------------------------------------- |
| id              | integer | The withdrawal id.                                      |
| currency        | string  | The currency code.                                      |
| type            | string  | The withdrawal type                                     |
| amount          | string  | The withdrawal amount                                   |
| fee             | double  | The exchange fee.                                       |
| blockchain_txid | string  | The withdrawal transaction id.                          |
| rid             | string  | The beneficiary ID or wallet address on the Blockchain. |
| state           | string  | The withdrawal state.                                   |
| confirmations   | integer | Number of confirmations.                                |
| note            | string  | Optional withdraw note.                                 |
| created_at      | string  | The datetime for the withdrawal.                        |
| updated_at      | string  | The datetime for the withdrawal.                        |
| done_at         | string  | The datetime when withdraw was completed                |

### Withdraw quote

| Name       | Type   | Description                                                   |
| ---------- | ------ | ------------------------------------------------------------- |
| id         | string | The id of the quote. Submit this with your withdrawal request |
| currency   | string | The currency code.                                            |
| fee        | string | The fee for the withdrawal                                    |
| blockchain | string | The blockchain for this quote                                 |
| created_at | string | The datetime the quote was created.                           |
| expires_at | string | The datetime the quote expires.                               |

### Beneficiary

| Name        | Type    | Description                                                                                                                                     |
| ----------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| uuid        | integer | Beneficiary Identifier                                                                                                                          |
| currency    | string  | Beneficiary currency code.                                                                                                                      |
| name        | string  | Name of the beneficiary.                                                                                                                        |
| description | string  | A personal description of the beneficiary for your records.                                                                                     |
| data        | json    | The beneficiary's blockchain address and/or destination tag, in JSON format. |
| state       | string  | Pending, Active, Archived (0,1,2)                                                                                                               |

### SubAccount

| Name           | Type   | Description                                                                               |
| -------------- | ------ | ----------------------------------------------------------------------------------------- |
| uid            | string | Sub-account unique identifier                                                             |
| account_name   | string | Display name for the sub-account                                                          |
| state          | string | Account state (active, deleted)                                                           |
| deposit_action | string | Action for deposits: 'none' (keep in sub-account) or 'transfer_to_parent' (auto-transfer) |
| created_at     | string | Timestamp when the sub-account was created in ISO 8601 format                             |

### Customer

Customer accounts come in two types with different response structures:

- **Individual Customers** - See [Customer (Individual)](#customer-individual) for personal account structure
- **Business Customers** - See [Customer (Business)](#customer-business) for business account structure

### Customer (Individual)

| Name                      | Type   | Description                                                              |
| ------------------------- | ------ | ------------------------------------------------------------------------ |
| uid                       | string | Customer unique identifier                                               |
| account_name              | string | Display name for the customer (usually custom_id)                        |
| state                     | string | Account state (active, deleted)                                          |
| custom_id                 | string | Custom identifier set by merchant/aggregator                             |
| created_at                | string | Timestamp when customer was created in ISO 8601 format                   |
| profile                   | object | Customer profile information (first_name, last_name, dob, address, etc.) |
| phone                     | object | Phone information (country, number, validated_at)                        |

### Customer (Business)

| Name             | Type    | Description                                                                                                                                                      |
| ---------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id               | integer | Customer database ID                                                                                                                                             |
| uid              | string  | Customer unique identifier                                                                                                                                       |
| email            | string  | Business email address                                                                                                                                           |
| role             | string  | Account role (customer)                                                                                                                                          |
| entity_type      | string  | Entity type (business)                                                                                                                                           |
| level            | integer | Account level                                                                                                                                                    |
| state            | string  | Account state (active, pending, deleted)                                                                                                                         |
| custom_id        | string  | Custom identifier set by merchant/aggregator                                                                                                                     |
| created_at       | string  | Timestamp when customer was created in ISO 8601 format                                                                                                           |
| updated_at       | string  | Timestamp when customer was last updated in ISO 8601 format                                                                                                      |

### Deposit

| Name          | Type    | Description                               |
| ------------- | ------- | ----------------------------------------- |
| id            | integer | Unique deposit id.                        |
| blockchain    | string  | Blockchain id.                            |
| currency      | string  | Deposit currency id.                      |
| amount        | double  | Deposit amount.                           |
| fee           | double  | Deposit fee.                              |
| txid          | string  | Deposit transaction id.                   |
| confirmations | integer | Number of deposit confirmations.          |
| state         | string  | Deposit state.                            |
| created_at    | string  | The datetime when deposit was created.    |
| completed_at  | string  | The datetime when deposit was completed.. |
| tid           | string  | The shared transaction ID                 |

### Transaction

| Name                     | Type    | Description                                                                            |
| ------------------------ | ------- | -------------------------------------------------------------------------------------- |
| id                       | integer | Unique transaction identifier                                                          |
| type                     | string  | Transaction type (deposit, withdraw, internal_transfer, trade, referral, maker_reward) |
| reference_type           | string  | Related entity type (Deposit, Withdraw, Order, etc.)                                   |
| reference_id             | integer | Related entity ID                                                                      |
| credit_currency          | string  | Currency received (if applicable)                                                      |
| credit_amount            | decimal | Amount received                                                                        |
| debit_currency           | string  | Currency sent (if applicable)                                                          |
| debit_amount             | decimal | Amount sent                                                                            |
| fee_currency             | string  | Fee currency                                                                           |
| fee                      | decimal | Fee amount                                                                             |
| blockchain_key           | string  | Blockchain identifier (for crypto transactions)                                        |
| blockchain_name          | string  | Blockchain name                                                                        |
| market_id                | string  | Market ID (for trades)                                                                 |
| order_id                 | string  | Order ID (for trades)                                                                  |
| price                    | decimal | Trade price (for trades)                                                               |
| origin_volume            | decimal | Original order volume (for trades)                                                     |
| payment_method_provider  | string  | Payment provider (for provider-backed transactions)                                    |
| payment_method_name      | string  | Payment method name                                                                    |
| payment_method_precision | integer | Decimal precision for payment method                                                   |
| payment_method_icon_url  | string  | Payment method icon URL                                                                |
| description              | string  | Transaction description                                                                |
| member_description       | string  | Your personal note for this transaction                                                |
| sender                   | string  | Sender information                                                                     |
| receiver                 | string  | Receiver information                                                                   |
| state                    | string  | Transaction state                                                                      |
| address                  | string  | Blockchain address (for crypto transactions)                                           |
| txid                     | string  | Transaction ID (blockchain txid or internal reference)                                 |
| beneficiary_id           | integer | Beneficiary ID (for withdrawals)                                                       |
| paid_at                  | string  | Payment completion timestamp (ISO8601)                                                 |
| created_at               | string  | Transaction creation timestamp (ISO8601)                                               |
| updated_at               | string  | Last update timestamp (ISO8601)                                                        |

### PaymentReference

| Name                 | Type    | Description                                                            |
| -------------------- | ------- | ---------------------------------------------------------------------- |
| id                   | integer | Payment reference ID                                                   |
| reference            | string  | Unique payment reference code (used by customer for deposits)          |
| uid                  | string  | Customer sub-account UID                                               |
| currency             | string  | Currency code (e.g., "btc", "usdc")                                    |
| expected_amount      | string  | Expected deposit amount (null if not specified)                        |
| deposit_action       | object  | Automated action configuration (null if not configured)                |
| state                | string  | Current state (see PaymentReference States)                            |
| expires_at           | string  | ISO8601 expiration timestamp                                           |
| usage_count          | integer | Number of times the reference has been used                            |
| last_used_at         | string  | ISO8601 timestamp of last use (null if never used)                     |
| webhook_url          | string  | Webhook URL (only included if configured)                              |
| webhook_protocol     | string  | Webhook protocol: "hmac", "plain_text", or "none" (only if configured) |
| webhook_secret       | string  | Webhook secret (only included for owner with includeSecret option)     |
| deposit_id           | integer | Linked deposit ID (only included if linked)                            |
| order_id             | integer | Linked order ID (only included if linked)                              |
| withdraw_id          | integer | Linked withdrawal ID (only included if linked)                         |
| internal_transfer_id | integer | Linked internal transfer ID (only included if linked)                  |
| error                | string  | Error message (only included if state is "failed")                     |
| created_at           | string  | ISO8601 creation timestamp                                             |
| updated_at           | string  | ISO8601 last update timestamp                                          |

**Deposit Action Object Structure:**

| Field              | Type    | Description                                                                                                                                                                                                                                                                                          |
| ------------------ | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| action             | string  | Action type: "none", "aggregate", "convert_and_withdraw", "aggregate_convert_and_withdraw"                                                                                                                                                                                                           |
| target_currency    | string  | Target currency for conversion (only for convert actions)                                                                                                                                                                                                                                            |
| beneficiary_id     | integer | Beneficiary ID for withdrawals (only for withdraw actions)                                                                                                                                                                                                                                           |
| lightning_invoice  | string  | Lightning invoice for lightning withdrawals (alternative to beneficiary_id)                                                                                                                                                                                                                          |
| max_slippage       | number  | Maximum slippage percentage                                                                                                                                                                                                                                                                          |
| beneficiary_amount | string  | Exact amount the beneficiary should receive. If specified and account has insufficient funds after conversion, the withdrawal is skipped and payment is marked as completed with converted funds remaining in the account. When using `lightning_invoice`, the invoice amount must match this value. |

**PaymentReference States:**

Payment references progress through various states based on the configured actions. Webhook notifications are sent when the reference reaches `completed` or `failed` state.

| State                     | Description                         |
| ------------------------- | ----------------------------------- |
| pending_deposit           | Waiting for deposit                 |
| pending_internal_transfer | Processing aggregation              |
| pending_buy_order         | Processing conversion (buy)         |
| pending_sell_order        | Processing conversion (sell)        |
| pending_withdraw          | Processing withdrawal               |
| completed                 | All actions completed successfully  |
| failed                    | Processing failed (see error field) |

### Account

| Name     | Type   | Description           |
| -------- | ------ | --------------------- |
| currency | string | Currency code.        |
| balance  | double | Account balance.      |
| locked   | double | Account locked funds. |

### Deposit Method

| Name            | Type    | Description                                                      |
| --------------- | ------- | ---------------------------------------------------------------- |
| uuid            | string  | Unique identifier for the deposit method                         |
| name            | string  | Display name of the deposit method                               |
| description     | string  | Human-readable description of the method                         |
| currencyId      | string  | Currency identifier (e.g., "btc", "usdc")                        |
| symbol          | string  | Currency symbol (e.g., "BTC", "USDC")                            |
| blockchainName  | string  | Blockchain network name (e.g., "bitcoin", "ethereum", "solana") |
| type            | string  | Method type (e.g., "crypto", "address", "lightning")            |
| currencyType    | string  | Currency type: always "crypto"                                  |
| providerId      | string  | Payment provider identifier (e.g., "calibri")                   |
| minAmount       | number  | Minimum deposit amount (after baseFactor conversion)             |
| maxAmount       | number  | Maximum deposit amount (derived from remaining limits)           |
| baseFactor      | number  | Conversion factor (divide stored amounts by this for display)    |
| precision       | number  | Decimal precision for display                                    |
| feeCape         | number  | Platform fee amount or percentage                               |
| feeProvider     | number  | Provider's fee amount or percentage                              |
| position        | number  | Display order position                                           |
| iconUrl         | string  | Icon URL for UI display                                          |
| verified        | boolean | Whether user is verified for this method                         |
| remainingLimits | object  | Remaining transaction limits (see Remaining Limits object below) |

### Withdraw Method

| Name            | Type    | Description                                                      |
| --------------- | ------- | ---------------------------------------------------------------- |
| uuid            | string  | Unique identifier for the withdraw method                        |
| id              | number  | Database ID (included for legacy support)                        |
| name            | string  | Display name of the withdraw method                              |
| description     | string  | Human-readable description of the method                         |
| currencyId      | string  | Currency identifier (e.g., "btc", "usdc")                        |
| symbol          | string  | Currency symbol (e.g., "BTC", "USDC")                            |
| blockchainName  | string  | Blockchain network name (e.g., "bitcoin", "ethereum", "solana") |
| type            | string  | Method type (e.g., "crypto", "address", "wallet")               |
| currencyType    | string  | Currency type: always "crypto"                                  |
| providerId      | string  | Payment provider identifier (e.g., "calibri")                   |
| processorId     | string  | Processor that handles withdrawals for this method               |
| minAmount       | number  | Minimum withdrawal amount (after baseFactor conversion)          |
| maxAmount       | number  | Maximum withdrawal amount (derived from remaining limits)        |
| baseFactor      | number  | Conversion factor (divide stored amounts by this for display)    |
| precision       | number  | Decimal precision for display                                    |
| feeCape         | number  | Platform withdrawal fee amount or percentage                    |
| feeProvider     | number  | Provider's withdrawal fee amount or percentage                   |
| position        | number  | Display order position                                           |
| iconUrl         | string  | Icon URL for UI display                                          |
| verified        | boolean | Whether user is verified for this method                         |
| remainingLimits | object  | Remaining transaction limits (see Remaining Limits object below) |

### Remaining Limits

The `remainingLimits` object is included in both Deposit Method and Withdraw Method responses.

| Name            | Type   | Description                                   |
| --------------- | ------ | --------------------------------------------- |
| transaction_max | number | Maximum amount allowed per single transaction |
| day             | number | Remaining limit for the current day           |
| week            | number | Remaining limit for the current week          |
| month           | number | Remaining limit for the current month         |
| year_to_date    | number | Remaining limit for the current year          |

### Lightning payment

| Name               | Type                       | Description                                                     |
| ------------------ | -------------------------- | --------------------------------------------------------------- |
| bolt               | String                     | The invoice bolt11.                                             |
| address            | String                     | The address of the invoice if not bolt11.                       |
| amount             | number                     | Invoice amount paid in SAT.                                     |
| description        | String                     | Description for this invoice in the bolt11.                     |
| member_description | String                     | Private member description for this invoice provided by member. |
| state              | String                     | Payment state.                                                  |
| created_at         | String (ISO8601 formatted) | The datetime when payment was created.                          |

### Lightning invoice

| Name               | Type                       | Description                                  |
| ------------------ | -------------------------- | -------------------------------------------- |
| bolt               | String                     | The invoice in bolt11.                       |
| amount             | BigDecimal                 | Invoice amount to be paid in SAT.            |
| conversion_percent | BigDecimal                 | The percentage to automatically convert to USDC on receipt. |
| amount_msat        | BigDecimal                 | Invoice amount to be paid in Mili SAT.       |
| description        | String                     | Description for this invoice.                |
| member_description | String                     | Private member Description for this invoice. |
| state              | String                     | Payment state.                               |
| created_at         | String (ISO8601 formatted) | The datetime when invoice was created.       |
| expires_at         | String (ISO8601 formatted) | The datetime when invoice expires.           |
| paid_at            | String (ISO8601 formatted) | The datetime when invoice was paid.          |

### Websocket orderbook snapshot

| Name             | Type                                  | Description                                                                  |
| ---------------- | ------------------------------------- | ---------------------------------------------------------------------------- |
| {market}.ob-snap | Object                                | The object key for the subscription                                          |
| asks             | [[Orderbook entry](#orderbook-entry)] | Array of ask / sell orderbook entries.                                       |
| bids             | [[Orderbook entry](#orderbook-entry)] | Array of bid / buy orderbook entries.                                        |
| sequence         | integer                               | The current sequence of the orderbook, used to ensure updates are sequential |

### Websocket orderbook increment

| Name            | Type                                  | Description                                                                  |
| --------------- | ------------------------------------- | ---------------------------------------------------------------------------- |
| {market}.ob-inc | Object                                | The object key for the subscription                                          |
| asks            | [[Orderbook entry](#orderbook-entry)] | Array of ask / sell orderbook entries.                                       |
| bids            | [[Orderbook entry](#orderbook-entry)] | Array of bid / buy orderbook entries.                                        |
| sequence        | integer                               | The current sequence of the orderbook, used to ensure updates are sequential |

```json
{
  "12.ob-snap": {
    "asks": [
      ["0.62", "150"],
      ["0.63", "80"],
      ["0.65", "200"]
    ],
    "bids": [
      ["0.60", "100"],
      ["0.59", "75"],
      ["0.58", "120"],
      ["0.55", "300"]
    ],
    "sequence": 9712
  }
}
```

### States

| Order states | Description                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------ |
| pending      | Initial state - order has been submitted (but is not yet placed / confirmed)                     |
| wait         | Order has been placed and is available on the orderbook                                          |
| done         | Order has been completely filled                                                                 |
| cancel       | The order has been cancelled (A cancelled limit order may still have executed trades against it) |
| reject       | The order submission was rejected                                                                |

| Deposit states |
| -------------- |
| submitted      |
| canceled       |
| rejected       |
| accepted       |
| collected      |
| skipped        |

| Withdraw states |
| --------------- |
| prepared        |
| submitted       |
| rejected        |
| accepted        |
| skipped         |
| processing      |
| succeed         |
| canceled        |
| failed          |
| errored         |
| confirming      |

