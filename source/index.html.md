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
  const path = `/api/v2/atlas/market/orders`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const orderData = {
    market: "btczar",
    side: "sell",
    volume: "0.00001",
    ord_type: "limit",
    price: "800000",
    post_only: true,
    custom_order_id: "302b0494-1a06-4089-846c-0c6b13f1b661",
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
  "uuid": "7e86a4fa-fcae-4953-b9bc-d6f328beea48",
  "side": "sell",
  "ord_type": "limit",
  "price": "800000.0",
  "avg_price": "0.0",
  "state": "pending",
  "market": "btczar",
  "created_at": "2023-12-08T08:46:59+01:00",
  "updated_at": "2023-12-08T08:46:59+01:00",
  "origin_volume": "0.00001",
  "remaining_volume": "0.00001",
  "executed_volume": "0.0",
  "trades_count": 0,
  "custom_order_id": "302b0494-1a06-4089-846c-0c6b13f1b661"
}
```

`https://app.calibri.io/api/v2/atlas/market/orders`

Create a new order on your account by signing your request and submitting the request

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
  const path = `/api/v2/atlas/market/orders`;
  const url = `${baseUrl}${path}`;
  const headers = await generateHeaders();

  const orderData = {
    market: "btczar",
    side: "buy",
    volume: "0.00001",
    ord_type: "limit",
    price: "750000",
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

`https://app.calibri.io/api/v2/atlas/market/orders`

When the `X-Auth-Subaccount-Uid` header is included, the order is created on behalf of the specified sub-account or customer. The parent account's API credentials are used for authentication, but the operation is performed in the context of the sub-account.

# Public request example

## <span class="request-type__get">GET</span> Get available markets

```javascript
const axios = require("axios");

getMarkets = async () => {
  const baseUrl = "https://app.calibri.io";
  const method = "GET";
  const path = `/api/v2/atlas/public/markets`;
  const url = `${baseUrl}${path}`;

  const axiosConfig = {
    method: method,
    url: url,
  };

  try {
    const results = await axios(axiosConfig);
    console.log(JSON.stringify(results.data, undefined, 2));
  } catch (err) {
    console.log(err);
  }
};

getMarkets();
```

> Markets response:

```json
[
  {
    "id": "btczar",
    "name": "BTC/ZAR",
    "base_unit": "btc",
    "quote_unit": "zar",
    "min_price": "200000.0",
    "max_price": "1200000.0",
    "min_amount": "0.00001",
    "amount_precision": 8,
    "price_precision": 2,
    "state": "enabled"
  },
  {
    "id": "ethzar",
    "name": "ETH/ZAR",
    "base_unit": "eth",
    "quote_unit": "zar",
    "min_price": "10000.0",
    "max_price": "100000.0",
    "min_amount": "0.0001",
    "amount_precision": 8,
    "price_precision": 2,
    "state": "enabled"
  },
  {
    "id": "xrpzar",
    "name": "XRP/ZAR",
    "base_unit": "xrp",
    "quote_unit": "zar",
    "min_price": "1.0",
    "max_price": "100.0",
    "min_amount": "1.0",
    "amount_precision": 6,
    "price_precision": 2,
    "state": "enabled"
  },
  {
    "id": "usdczar",
    "name": "USDC/ZAR",
    "base_unit": "usdc",
    "quote_unit": "zar",
    "min_price": "14.0",
    "max_price": "29.0",
    "min_amount": "0.5",
    "amount_precision": 2,
    "price_precision": 2,
    "state": "enabled"
  }
]
```

`GET https://app.calibri.io/api/v2/atlas/public/markets`

Returns a list of all available markets

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
| currency  | query      | Filter by currency code (matches credit or debit currency, e.g., "btc", "zar")                              | No       | string  |
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

Create a new beneficiary (a saved withdrawal destination — a cryptocurrency wallet address or a bank account for fiat transfers). All beneficiaries created via the API are automatically activated and ready for use.

### Parameters

| Name               | Located in | Description                                                                                   | Required | Schema  | Max Length |
| ------------------ | ---------- | --------------------------------------------------------------------------------------------- | -------- | ------- | ---------- |
| currency           | formData   | Currency code (e.g., "btc", "eth", "zar", "gbp")                                              | Yes      | string  | 10 chars   |
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

Fiat `data` Object Fields

| Field                   | Type   | Required | Description                                                |
| ----------------------- | ------ | -------- | ---------------------------------------------------------- |
| full_name               | string | **Yes**  | Account holder's full legal name (mandatory for fiat)      |
| account_number          | string | No       | Bank account number                                        |
| bank_name               | string | No       | Name of the bank                                           |
| bank_swift_code         | string | No       | SWIFT/BIC code                                             |
| bank_routing_number     | string | No       | Routing number (US banks)                                  |
| iban                    | string | No       | IBAN (International Bank Account Number)                   |
| bank_address            | string | No       | Bank's physical address                                    |
| bank_city               | string | No       | Bank's city                                                |
| intermediary_bank_swift | string | No       | Intermediary bank SWIFT code (for international transfers) |
| account_holder_name     | string | No       | Account holder name (may differ from full_name)            |
| account_type            | string | No       | Type of account (checking, savings, etc.)                  |

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

**Example 3: Fiat - ZAR (South African Rand)**

```json
{
  "currency": "zar",
  "name": "South African Bank Account",
  "description": "Personal savings account",
  "data": {
    "full_name": "Thabo Mbeki",
    "account_number": "1234567890",
    "bank_name": "Standard Bank",
    "bank_swift_code": "SBZAZAJJ",
    "bank_address": "30 Baker Street",
    "bank_city": "Johannesburg",
    "account_type": "savings"
  }
}
```

**Example 4: Fiat - GBP SWIFT Transfer**

```json
{
  "currency": "gbp",
  "name": "UK Business Account",
  "description": "Company bank account",
  "data": {
    "full_name": "John Smith",
    "account_number": "12345678",
    "bank_name": "Barclays Bank",
    "bank_swift_code": "BARCGB22",
    "bank_address": "1 Churchill Place",
    "bank_city": "London",
    "account_type": "checking"
  }
}
```

**Example 5: Fiat - EUR IBAN Transfer**

```json
{
  "currency": "eur",
  "name": "EU Partner Account",
  "description": "European supplier payment",
  "data": {
    "full_name": "Jean Dupont",
    "iban": "FR1420041010050500013M02606",
    "bank_name": "BNP Paribas",
    "bank_swift_code": "BNPAFRPP",
    "bank_city": "Paris"
  }
}
```

**Example 6: Fiat - International with Intermediary Bank**

```json
{
  "currency": "usd",
  "name": "International Vendor",
  "description": "Asian supplier USD payment",
  "data": {
    "full_name": "ABC Trading Company Ltd",
    "account_number": "9876543210",
    "bank_name": "Asia Commercial Bank",
    "bank_swift_code": "ACBVSGSG",
    "intermediary_bank_swift": "CHASUS33",
    "bank_address": "123 Business Street",
    "bank_city": "Singapore"
  }
}
```

### Important Notes

- **API-created beneficiaries** are automatically activated (state = `active`) and do not require PIN activation
- **Fiat beneficiaries** must include `full_name` in the `data` object (mandatory)
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
| note           | formData   | ZAR withdrawals only. Use 'same_day_rtc' for [Faster withdraws](https://calibri.io/help) | No       | string  |

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

Get all available deposit methods for the authenticated user. Returns deposit methods for both cryptocurrencies and fiat currencies, filtered based on your remaining transaction limits.

Each method includes minimum and maximum amounts, fees, provider information, and remaining limits for different time periods (daily, weekly, monthly, yearly).

### Responses

| Code | Description                         | Schema                                |
| ---- | ----------------------------------- | ------------------------------------- |
| 200  | Array of available deposit methods. | [ [Deposit Method](#deposit-method) ] |

## <span class="request-type__get">GET</span> Get withdraw methods

`/api/v2/payments/withdraws/methods`

### Description

Get all available withdrawal methods for the authenticated user. Returns withdrawal methods for both cryptocurrencies and fiat currencies, filtered based on your remaining transaction limits.

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
  "target_currency": "currency_code", // Required for convert actions (e.g., "zar", "usd")
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
  "account_name": "USD Conversion Account",
  "deposit_action": {
    "action": "convert_and_withdraw",
    "target_currency": "usd",
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

You can also combine the public and private streams into one so that you only need to maintain 1x connection for example `?stream=btczar.ob-inc&stream=order&stream=trade&stream=balance`

## <span class="request-type__get">SUBSCRIBE</span> Orderbook

```javascript
const WebSocket = require("ws");

async function connectWssPublic() {
  try {
    const baseUrl = `wss://app.calibri.io/api/v2`;
    const streams = "?stream=btczar.ob-inc&stream=ethzar.ob-inc";
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

`?stream=btczar.ob-inc&ethzar.ob-inc`

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
    const streams = "?stream=btczar.kline-15m";
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
      "?stream=btczar.ob-inc&stream=ethzar.ob-inc&stream=order&stream=trade&stream=balance";
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

    if (data["xrpzar.ob-snap"] || data["xrpzar.ob-inc"]) {
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

`?stream=btczar.ob-inc&stream=order&stream=trade&stream=balance`

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
    "uuid": "7e86a4fa-fcae-4953-b9bc-d6f328beea48",
    "side": "sell",
    "ord_type": "limit",
    "price": "800000.0",
    "avg_price": "0.0",
    "state": "wait",
    "market": "btczar",
    "created_at": "2023-12-08T08:46:59+01:00",
    "updated_at": "2023-12-08T08:46:59+01:00",
    "origin_volume": "0.00001",
    "remaining_volume": "0.00001",
    "executed_volume": "0.0",
    "trades_count": 0
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
    "price": "800000.0",
    "amount": "0.00001",
    "total": "8.0",
    "market": "btczar",
    "side": "sell",
    "taker_type": "buy",
    "fee_currency": "zar",
    "fee": "0.002",
    "fee_amount": "0.016",
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
    "currency": "btc",
    "balance": "0.05000000",
    "locked": "0.00001000",
    "available": "0.04999000",
    "updated_at": 1702027621000
  }
}
```

**Response Fields:**

| Field      | Type    | Description                                                      |
| ---------- | ------- | ---------------------------------------------------------------- |
| currency   | string  | Currency code (btc, eth, zar, usdt, etc.)                        |
| balance    | string  | Total balance (available + locked). String for decimal precision |
| locked     | string  | Amount currently locked in open orders                           |
| available  | string  | Available balance for trading (balance - locked)                 |
| updated_at | integer | Unix timestamp in milliseconds when balance was updated          |

**Balance Update Triggers:**

1. **Order Creation** - Locks balance when a limit or market order is placed
2. **Order Cancellation** - Unlocks balance when an order is canceled
3. **Trade Execution (Maker)** - Updates both base and quote currency balances
4. **Trade Execution (Taker)** - Updates both base and quote currency balances with fee deduction
5. **Partial Fill** - Each partial fill triggers separate balance updates
6. **Order Rejection** - Balance unlock if order validation fails after initial lock

**Example 1: Order Creation (Balance Lock)**

When you place a buy order for 0.001 BTC @ 800,000 ZAR, your ZAR balance is locked:

```json
{
  "balance": {
    "currency": "zar",
    "balance": "10000.00",
    "locked": "800.00",
    "available": "9200.00",
    "updated_at": 1702027621000
  }
}
```

**Example 2: Order Cancellation (Balance Unlock)**

When you cancel the order, the locked ZAR is released:

```json
{
  "balance": {
    "currency": "zar",
    "balance": "10000.00",
    "locked": "0.00",
    "available": "10000.00",
    "updated_at": 1702027625000
  }
}
```

**Example 3: Trade Execution - Sell Order (Base Currency)**

When your sell order for 0.00001 BTC executes, you receive two balance updates. First, the BTC is deducted:

```json
{
  "balance": {
    "currency": "btc",
    "balance": "0.04999000",
    "locked": "0.00000000",
    "available": "0.04999000",
    "updated_at": 1702027630000
  }
}
```

**Example 4: Trade Execution - Sell Order (Quote Currency)**

Immediately after, you receive ZAR (minus the taker fee of 0.2%):

```json
{
  "balance": {
    "currency": "zar",
    "balance": "10007.984",
    "locked": "0.00",
    "available": "10007.984",
    "updated_at": 1702027630001
  }
}
```

**Example 4: Partial Fill**

When your order is partially filled (50% of 0.001 BTC order):

```json
{
  "balance": {
    "currency": "btc",
    "balance": "0.04999500",
    "locked": "0.00000500",
    "available": "0.04999000",
    "updated_at": 1702027635000
  }
}
```

**Important Notes:**

- **Atomic Updates**: Balance updates are atomic - you receive one update per currency per event
- **Dual Updates for Trades**: For each trade, expect **two balance updates** - one for the base currency and one for the quote currency
- **Timestamp Precision**: The `updated_at` field uses **Unix milliseconds** (not seconds)
- **String Values**: Balance values are strings to preserve decimal precision - parse them as decimals, not floats
- **Rapid Updates**: Multiple rapid trades may result in several balance updates in quick succession
- **Fee Inclusion**: Fees are already deducted from the amounts shown in balance updates

**Edge Cases:**

- **Insufficient Funds**: No balance update is sent (order is rejected before locking funds)
- **Self-Trade Prevention**: If a self-trade is prevented, balance may lock and then immediately unlock
- **Partial Fills**: Each partial fill of an order triggers separate balance updates for both currencies
- **Maker vs Taker Fees**: Fee amounts differ based on whether you're the maker (0.1%) or taker (0.2%)

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
| side             | string            | Either 'sell' or 'buy'.                                 |
| ord_type         | string            | Type of order, either 'limit' or 'market'.              |
| post_only        | boolean           | If the Limit order was submitted as post_only .         |
| price            | double            | Price for 1x base unit.                                 |
| avg_price        | double            | Average execution price over all executed trades.       |
| state            | string            | Current state of the order [Order states](#states)      |
| market           | string            | The market in which the order is placed, e.g. 'btczar'. |
| created_at       | string            | Order create time in iso8601 format.                    |
| updated_at       | string            | Order updated time in iso8601 format.                   |
| origin_volume    | double            | The original amount of the order.                       |
| remaining_volume | double            | The remaining amount of the original order volume.      |
| executed_volume  | double            | The executed volume of the original order volume.       |
| trades_count     | integer           | Number of trades executed against this limit order.     |
| trades           | [[Trade](#trade)] | Array of trades executed against this order.            |

### Market

| Name             | Type   | Description                                  |
| ---------------- | ------ | -------------------------------------------- |
| id               | string | Unique market id eg 'btczar'                 |
| name             | string | Market name.                                 |
| base_unit        | string | Market Base unit.                            |
| quote_unit       | string | Market Quote unit.                           |
| min_price        | double | Minimum order price.                         |
| max_price        | double | Maximum order price.                         |
| min_amount       | double | Minimum order amount.                        |
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
| data        | json    | Bank account details for a fiat beneficiary in JSON format. For crypto it's the blockchain address and/or tag |
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
| business_profile | object  | Business details (legal_name, trading_name, registration_number, tax_id, incorporation_date, business_type, industry, address, city, postcode, country, website) |
| ubos             | array   | Array of Ultimate Beneficial Owners with ownership details                                                                                                       |
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
| payment_method_provider  | string  | Payment provider (for fiat transactions)                                               |
| payment_method_name      | string  | Payment method name                                                                    |
| payment_method_precision | integer | Decimal precision for payment method                                                   |
| payment_method_icon_url  | string  | Payment method icon URL                                                                |
| voucher_pin              | string  | Voucher PIN (if applicable)                                                            |
| change_voucher_pin       | string  | Change voucher PIN (if change was issued)                                              |
| change_voucher_expiry    | string  | Change voucher expiry date (ISO8601)                                                   |
| change_voucher_amount    | decimal | Change voucher amount                                                                  |
| description              | string  | Transaction description                                                                |
| member_description       | string  | Your personal note for this transaction                                                |
| sender                   | string  | Sender information                                                                     |
| receiver                 | string  | Receiver information                                                                   |
| state                    | string  | Transaction state                                                                      |
| address                  | string  | Blockchain address (for crypto transactions)                                           |
| txid                     | string  | Transaction ID (blockchain txid or internal reference)                                 |
| beneficiary_id           | integer | Beneficiary ID (for withdrawals)                                                       |
| paid_at                  | string  | Payment completion timestamp (ISO8601)                                                 |
| expires_at               | string  | Expiry timestamp (ISO8601, for vouchers)                                               |
| created_at               | string  | Transaction creation timestamp (ISO8601)                                               |
| updated_at               | string  | Last update timestamp (ISO8601)                                                        |

### PaymentReference

| Name                 | Type    | Description                                                            |
| -------------------- | ------- | ---------------------------------------------------------------------- |
| id                   | integer | Payment reference ID                                                   |
| reference            | string  | Unique payment reference code (used by customer for deposits)          |
| uid                  | string  | Customer sub-account UID                                               |
| currency             | string  | Currency code (e.g., "btc", "zar")                                     |
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
| currencyId      | string  | Currency identifier (e.g., "btc", "zar")                         |
| symbol          | string  | Currency symbol (e.g., "BTC", "ZAR")                             |
| blockchainName  | string  | Blockchain network name (e.g., "bitcoin", "ethereum", "fiat")    |
| type            | string  | Method type (e.g., "bank", "crypto", "voucher")                  |
| currencyType    | string  | Currency type: "fiat" or "crypto"                                |
| providerId      | string  | Payment provider identifier (e.g., "stitch", "xago", "cape")     |
| minAmount       | number  | Minimum deposit amount (after baseFactor conversion)             |
| maxAmount       | number  | Maximum deposit amount (derived from remaining limits)           |
| baseFactor      | number  | Conversion factor (divide stored amounts by this for display)    |
| precision       | number  | Decimal precision for display                                    |
| feeCape         | number  | Cape's fee amount or percentage                                  |
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
| currencyId      | string  | Currency identifier (e.g., "btc", "zar")                         |
| symbol          | string  | Currency symbol (e.g., "BTC", "ZAR")                             |
| blockchainName  | string  | Blockchain network name (e.g., "bitcoin", "ethereum", "fiat")    |
| type            | string  | Method type (e.g., "crypto", "bank", "wallet")                   |
| currencyType    | string  | Currency type: "fiat" or "crypto"                                |
| providerId      | string  | Payment provider identifier (e.g., "xago", "celbux", "cape")     |
| processorId     | string  | Processor that handles withdrawals for this method               |
| minAmount       | number  | Minimum withdrawal amount (after baseFactor conversion)          |
| maxAmount       | number  | Maximum withdrawal amount (derived from remaining limits)        |
| baseFactor      | number  | Conversion factor (divide stored amounts by this for display)    |
| precision       | number  | Decimal precision for display                                    |
| feeCape         | number  | Cape's withdrawal fee amount or percentage                       |
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

### Voucher Redemption

| Name           | Type                              | Description                                                              |
| -------------- | --------------------------------- | ------------------------------------------------------------------------ |
| amount         | number                            | The amount deposited after fees.                                         |
| origin_amount  | number                            | The original amount of the voucher.                                      |
| change_voucher | [Change voucher](#change-voucher) | Optional if provider offers this, a new voucher for the remaining amount |

### Change voucher

| Name          | Type   | Description                                                       |
| ------------- | ------ | ----------------------------------------------------------------- |
| change_amount | number | Amount of change from the redeemed voucher, available to be spent |
| change_expiry | string | Expiry date of the new voucher.                                   |
| change_pin    | number | The new voucher_pin to be used for redemption.                    |

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
| conversion_percent | BigDecimal                 | The amount to automatically convert to FIAT. |
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
  "btczar.ob-snap": {
    "asks": [
      ["841039.73", "0.156431"],
      ["845402.1", "0.004389"],
      ["900000.0", "0.00001"]
    ],
    "bids": [
      ["835956.8", "0.047756"],
      ["831012.48", "0.03749"],
      ["829990.51", "0.019462"],
      ["400000.0", "0.0001"]
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

# Lists

Lists of supported Source of funds and occupations

### Source of funds

Please see the array of supported source of funds to the right

<pre><code>SOURCE_OF_FUNDS = [
'Angel Investments',
'Business Income',
'Client funds',
'Crowdfunding',
'Dividends',
'Foreign Investments',
'Gifts',
'Government Benefits',
'Grants and Scholarships',
'Inheritance',
'Insurance Payouts',
'Investment Income',
'Legal Settlements',
'Loans',
'Partnership or Shareholder Distributions',
'Personal Savings',
'Retirement Funds',
'Royalties',
'Salary or Wages',
'Sale of Assets',
'Shareholder funds',
'Venture Capital',
]</code></pre>

### Occupations

Please see the array of supported occupations to the right

<pre><code>OCCUPATIONS = [
'Account Collector',
'Accountant',
'Accounting Clerk',
'Actor',
'Actuary',
'Acupuncturist',
'Administrative Assistant',
'Administrative Service and Facility Manager',
'Adult Literacy Teacher',
'Advertising Manager',
'Advertising Sale Agent',
'Aerospace Engineering and Operation Technician',
'Aerospace Engineer',
'Agent and Business Manager of Artist, Performer, and Athlete',
'Agricultural Engineer',
'Agricultural Inspector',
'Agricultural Manager',
'Agricultural Scientist',
'Agricultural Technician',
'Agricultural Worker',
'Air Conditioning and Heating Mechanic and Installer',
'Air Traffic Controller',
'Aircraft Cargo Handling Supervisor',
'Aircraft Equipment Mechanic and Technician',
'Aircraft Service Attendant',
'Airfield Operation Specialist',
'Airline and Commercial Pilot',
'Ambulance Dispatcher',
'Ambulance Driver and Attendant (Except Emergency Medical Technician)',
'Amusement and Recreation Attendant',
'Animal Care and Service Worker',
'Animal Control Worker',
'Announcer',
'Anthropologist',
'Appraiser and Assessor of Property',
'Arbitrator',
'Archeologist',
'Architect',
'Architectural Manager',
'Archivist',
'Art Director',
'Assembler',
'Astronomer',
'Athlete and Sport Competitor',
'Athletic Trainer',
'Atmospheric Scientist',
'Attorney',
'Avionic Equipment Mechanic and Technician',
'Audio Technician',
'Audiologist',
'Audiovisual Equipment Installer and Repairer',
'Auditing Clerk',
'Auditor',
'Author',
'Automated Teller Machine (ATM) Repairer',
'Automotive and Watercraft Service Attendant',
'Automotive Body and Glass Repairer',
'Automotive Service Technician and Mechanic',
'Baggage Porter and Bellhop',
'Bailiff',
'Baker',
'Barber',
'Bartender',
'Behavioral Disorder Counselor',
'Benefit Manager',
'Bicycle Repairer',
'Bill Collector',
'Biochemist and Biophysicist',
'Bioengineer',
'Biological Scientist (all other)',
'Biological Technician',
'Biomedical Engineer',
'Blockmason',
'Boiler Operator',
'Boilermaker',
'Bookkeeping Clerk',
'Brazer',
'Brickmason',
'Bridge and Lock Tender',
'Broadcast Technician',
'Budget Analyst',
'Building Cleaner',
'Building Cleaning Worker (all other)',
'Building Inspector',
'Bu Driver',
'Business Operation Specialist (all other)',
'Butcher',
'Buyer',
'Camera and Photographic Equipment Repairer',
'Camera Operator',
'Captioner',
'Cardiovascular Technologist and Technician',
'Career Counselor and Advisor',
'Career and Technical Education Teacher',
'Cargo and Freight Agent',
'Carpenter',
'Carpet Installer',
'Cartographer',
'Cashier',
'Ceiling Tile Installer',
'Cement Mason',
'Chauffeur',
'Chef',
'Chemical Engineer',
'Chemical Plant and System Operator',
'Chemical Technician',
'Chemist',
'Childcare Center Director',
'Childcare Worker',
'Chiropractor',
'Choreographer',
'Civil Engineering Technician',
'Civil Engineer',
'Claim Adjuster, Appraiser, Examiner, and Investigator',
'Clergy',
'Clerk',
'Bookkeeping, Accounting, and Auditing Clerk',
'Counter and Rental Clerk',
'Financial Clerk',
'General Office Clerk',
'Information Clerk',
'Mail Clerk and Mail Machine Operator (Except Postal Service)',
'Material Recording Clerk',
'Clinical Laboratory Technologist and Technician',
'Coach and Scout',
'Coating Worker',
'Commercial Designer',
'Commercial Diver',
'Commodity Sale Agent',
'Communication Equipment Operator (all other)',
'Community and Social Service Specialist (all other)',
'Community Association Manager',
'Community Health Worker',
'Community Service Manager',
'Compensation, Benefit, and Job Analysi Specialist',
'Compensation and Benefit Manager',
'Compliance Officer',
'Composer - Music',
'Computer Control Programmer and Operator',
'Computer Hardware Engineer',
'Computer Manager',
'Computer Network Architect',
'Computer Occupation (all other)',
'Computer Programmer',
'Computer Repairer',
'Computer Scientist',
'Computer Software Engineer',
'Computer Support Specialist',
'Computer System Analyst',
'Computer System Administrator',
'Concierge',
'Conciliator',
'Conservation Scientist',
'Conservation Technician',
'Construction and Related Worker (all other)',
'Construction Equipment Operator',
'Construction Inspector',
'Construction Laborer and Helper',
'Construction Manager',
'Continuou Mining Machine Operator',
'Control and Valve Installer and Repairer (Except Mechanical Door)',
'Cook',
'Correctional Officer',
'Correctional Treatment Specialist',
'Cosmetologist',
'Cost Estimator',
'Costume Attendant',
'Counselor (all other)',
'Counter and Rental Clerk',
'Courier and Messenger',
'Court Reporter',
'Craft Artist',
'Credit Analyst',
'Credit Counselor',
'Crematory Operator',
'Crossing Guard',
'Curator',
'Customer Service Representative',
'Cutter and Trimmer (Hand)',
'Dancer',
'Data Entry Keyer',
'Data Scientist',
'Database Administrator',
'Delivery Truck Driver and Driver/Sale Worker',
'Demonstrator and Product Promoter',
'Dental Assistant',
'Dental Hygienist',
'Dental Laboratory Technician',
'Dentist',
'Derrick Operator (Oil and Ga)',
'Designer (all other)',
'Desktop Publisher',
'Development Manager',
'Diagnostic Medical Sonographer',
'Diesel Service Technician and Mechanic',
'Dietitian',
'Digital Designer',
'Director - Music',
'Director - Film, Theater',
'Disc Jockey / DJ',
'Dishwasher',
'Dispatcher (Police, Fire, and Ambulance)',
'Dispatcher (Except Police, Fire, and Ambulance)',
'Doctor',
'Door-to-Door Sale Worker, New and Street Vendor, and Related Worker',
'Drafter',
'Drywall Installer',
'Earth Driller (Except Oil and Ga)',
'Economist',
'Editor',
'Education Administrator - Postsecondary',
'Education Administrator - Elementary, Middle, and High School',
'Education Administrator (all other)',
'Education, Training, and Library Worker (all other)',
'Electrical and Electronic Engineering Technician',
'Electrical and Electronic Engineer',
'Electrical and Electronic Installer and Repairer',
'Electrician',
'Electro-mechanical Technician',
'Elementary School Principal',
'Elementary School Teacher',
'Elevator Installer and Repairer',
'Embalmer',
'Emergency Management Director',
'Emergency Medical Technician (EMT)',
'Emergency Response Dispatcher',
'Engineering Manager',
'Engineering Technician, except Drafter (all other)',
'Aerospace Engineer',
'Agricultural Engineer',
'Biomedical Engineer',
'Chemical Engineer',
'Civil Engineer',
'Computer Hardware Engineer',
'Computer Software Engineer',
'Electrical and Electronic Engineer',
'Environmental Engineer',
'Flight Engineer',
'Geological Engineer',
'Health and Safety Engineer',
'Industrial Engineer',
'Manufacturing Engineer',
'Marine Engineer',
'Material Engineer',
'Mechanical Engineer',
'Mining Engineer',
'Mining Safety Engineer',
'Nuclear Engineer',
'Petroleum Engineer',
'Sale Engineer',
'Stationary Engineer',
'All other Engineer',
'English a a Second Language (ESL) Teacher',
'Entertainer and Performer, Sport and Related Worker (all other)',
'Entertainment Attendant and Related Worker (all other)',
'Environmental Engineering Technologist and Technician',
'Environmental Science and Protection Technician',
'Environmental Scientist and Specialist',
'Epidemiologist',
'Escalator Installer and Repairer',
'Etcher and Engraver',
'Exercise Physiologist',
'Explosive Worker, Ordnance Handling Expert, and Blaster',
'Extraction Worker Helper',
'Extraction Worker (all other)',
'Fabric and Apparel Patternmaker',
'Fabricator',
'Family Therapist',
'Farm Labor Contractor',
'Farmer',
'Fashion Designer',
'Fence Erector',
'Film Editor',
'Financial Analyst',
'Financial Clerk',
'Financial Clerk (all other)',
'Financial Examiner',
'Financial Manager',
'Financial Service Sale Agent',
'Fine Artist',
'Fire Inspector and Investigator',
'Firefighter',
'First-Line Supervisor of Construction Trade and Extraction Worker',
'First-Line Supervisor of Correctional Officer',
'First-Line Supervisor of Farming, Fishing, and Forestry Worker',
'First-Line Supervisor of Fire Fighting and Prevention Worker',
'First-Line Supervisor of Food Preparation and Serving Worker',
'First-Line Supervisor of Housekeeping and Janitorial Worker',
'First-Line Supervisor of Landscaping, Lawn Service, and Groundskeeping Worker',
'First-Line Supervisor of Mechanic, Installer, and Repairer',
'First-Line Supervisor of Nonretail Sale Worker',
'First-Line Supervisor of Office and Administrative Support Worker',
'First-Line Supervisor of Personal Service Worker',
'First-Line Supervisor of Police Officer and Detective',
'First-Line Supervisor of Production and Operating Worker',
'First-Line Supervisor of Protective Service Worker (all other)',
'First-Line Supervisor of Retail Sale Worker',
'First-Line Supervisor of Transportation and Material-Moving Machine and Vehicle Operator',
'Fishing Worker',
'Fitness Trainer and Instructor',
'Flight Attendant',
'Floor Layer, Sander and Finisher',
'Flooring Installer',
'Floral Designer',
'Food and Beverage Serving and Related Worker',
'Food Preparation Worker',
'Food Processing Equipment Worker',
'Food Science Technician',
'Food Scientist',
'Food Preparation and Serving Related Worker (all over)',
'Food Service Manager',
'Forensic Science Technician',
'Forest and Conservation Technician',
'Forest and Conservation Worker',
'Forest Fire Inspector and Prevention Specialist',
'Forester',
'Fundraiser',
'Fundraising Manager',
'Funeral Attendant',
'Funeral Service Worker',
'G.E.D. Teacher',
'Gaming Change Person and Booth Cashier',
'Gambing Service Worker',
'Gambing Surveillance Officer',
'Ga Compressor and Ga Pumping Station Operator',
'Ga Plant Operator',
'General Maintenance and Repair Worker',
'Genetic Counselor',
'Geographer',
'Geological Technician',
'Geoscientist',
'Glazier',
'Grader and Sorter (Agricultural Product)',
'Graduate Teaching Assistant',
'Graphic Designer',
'Grinding and Polishing Worker (Hand)',
'Ground Maintenance Worker',
'Hairstylist',
'Hand Laborer',
'Hazardou Material Removal Worker',
'Head Cook',
'Health Diagnosing and Treating Practitioner (all other)',
'Health Educator and Community Health Worker',
'Health Information Technologist',
'Health Service Manager',
'Healthcare Support Worker (all other)',
'Hearing Officer',
'Heating and Air Conditioning Mechanic and Installer',
'Heavy and Tractor-trailer Truck Driver',
'Heavy Vehicle Service Technician',
'High School Equivalency Diploma Teacher',
'High School Principal',
'High School Teacher',
'Highway Maintenance Worker',
'Historian',
'Home Appliance Repairer',
'Home Health Aide',
'Hotel Manager',
'Housekeeping Cleaner',
'Human Resource Manager',
'Human Resource Specialist',
'Human Service Assistant',
'Hunting Worker',
'Hydrologic Technician',
'Hydrologist',
'Industrial Designer',
'Industrial Engineering Technologist and Technician',
'Industrial Machinery Mechanic and Maintenance Worker',
'Industrial Production Manager',
'Information Research Scientist',
'Information Security Analyst',
'Information System Manager',
'Inspector, Tester, Sorter, Sampler, and Weigher',
'Installation, Maintenance, and Repair Worker Helper',
'Installation, Maintenance, and Repair Worker (all other)',
'Instructional Coordinator',
'Insulation Worker',
'Insurance Sale Agent',
'Insurance Underwriter',
'Interior Designer',
'Interpreter',
'Iron Worker',
'Janitor',
'Jeweler',
'Job Analysi Specialist',
'Journalist',
'Judge',
'Kindergarten Teacher',
'Labor Relation Specialist',
'Laboratory Animal Caretaker',
'Landscape Architect',
'Lawyer',
'Layout Worker (Metal and Plastic)',
'Legal Assistant',
'Legal Support Worker (all other)',
'Legislator',
'Librarian and Library Media Specialist',
'Library Technician and Assistant',
'Licensed Practical and Licensed Vocational Nurse',
'Life Scientist (all other)',
'Life, Physical, and Social Science Technician (all other)',
'Lifeguard and Other Recreational Protective Service Worker',
'Lighting Technician',
'Line Installer and Repairer',
'Loan Officer',
'Locker Room, Coatroom, and Dressing Room Attendant',
'Locksmith and Safe Repairer',
'Lodging Manager',
'Logging Worker',
'Logistician',
'Adhesive Bonding Machine Operator and Tender',
'Chemical Equipment Operator and Tender',
'Cleaning, Washing, and Metal Pickling Equipment Operator and Tender',
'Cooling and Freezing Equipment Operator and Tender',
'Crushing, Grinding, and Polishing Machine Setter, Operator, and Tender',
'Cutting and Slicing Machine Setter, Operator, and Tender',
'Extruding and Forming Machine Setter, Operator, and Tender (Synthetic and Glass Fiber)',
'Extruding, Forming, Pressing, and Compacting Machine Setter, Operator, and Tender',
'Furnace, Kiln, Oven, Drier, and Kettle Operator and Tender',
'Mixing and Blending Machine Setter, Operator, and Tender',
'Packaging and Filling Machine Operator and Tender',
'Paper Good Machine Setter, Operator, and Tender',
'Photographic Process Worker and Processing Machine Operator',
'Separating, Filtering, Clarifying, Precipitating, and Still Machine Setter, Operator, and Tender',
'Sewing Machine Operator',
'Shoe Machine Operator and Tender',
'Textile Bleaching and Dyeing Machine Operator and Tender',
'Textile Cutting Machine Setter, Operator, and Tender',
'Textile Knitting and Weaving Machine Setter, Operator, and Tender',
'Textile Winding, Twisting, and Drawing Out Machine Setter, Operator, and Tender',
'Machinist',
'Maid',
'Maintenance and Repair Worker, General',
'Makeup Artist (Theatrical and Performance)',
'Management Analyst',
'Manager (all other)',
'Manicurist',
'Manufactured Building and Mobile Home Installer',
'Manufacturing Sale Representative',
'Mapping Technician',
'Marble Setter',
'Marine Mechanic',
'Market Research Analyst',
'Marketing Manager',
'Marriage and Family Therapist',
'Mason: Brick, Block, Stone, and Cement',
'Massage Therapist',
'Material Mover',
'Material Moving Worker (all other)',
'Material Moving Machine Operator',
'Mathematical Science Occupation (all other)',
'Material Scientist',
'Mathematician',
'Meat, Poultry, and Fish Cutter and Trimmer',
'Meat Packer',
'Mechanical Door Repairer',
'Mechanical Engineering Technologist and Technician',
'Mechanic - Automotive',
'Mechanic - Diesel',
'Mechanic - Heating, Air Conditioning, and Refrigeration',
'Mechanic - Industrial Machinery',
'Mechanic - Small Engine',
'Mediator',
'Medical Appliance Technician',
'Medical Assistant',
'Medical Billing and Coding',
'Medical Doctor',
'Medical Equipment Repairer',
'Medical Laboratory Technologist and Technician',
'Medical Record Technician',
'Medical Registrar Technician',
'Medical Scientist',
'Medical Service Manager',
'Medical Transcriptionist',
'Meeting, Convention, and Event Planner',
'Mental Health Counselor',
'Merchandise Displayer and Window Trimmer',
'Metal Machine Worker',
'Metal Worker',
'Metal Worker and Plastic Worker (all other)',
'Meteorologist',
'Meter Reader (Utility)',
'Microbiologist',
'Middle School Principal',
'Middle School Teacher',
'Millwright',
'Mine Cutting and Channeling Machine Operator',
'Mining Machine Operator (all other)',
'Mobile Equipment Service Technician',
'Model',
'Model Maker (Wood)',
'Molder, Shaper, and Caster (Except Metal and Plastic)',
'Mortician',
'Motel Manager',
'Motion Picture Projectionist',
'Motor Vehicle Operator (all other)',
'Motorcycle Mechanic',
'MRI Technologist',
'Multimedia Artist and Animator',
'Museum Technician',
'Music Director and Composer',
'Musical Instrument Repairer and Tuner',
'Musician',
'Nail Technician',
'Natural Science Manager',
'Naval Architect',
'Network Architect',
'Network System Administrator',
'New Analyst',
'Nuclear Medicine Technologist',
'Nuclear Technician',
'Nurse',
'Nurse Anesthetist, Nurse Midwive, and Nurse Practitioner',
'Nursing Assistant',
'Nutritionist',
'Office and Administrative Support Worker (all other)',
'Office Clerk',
'Office Machine Operator (Except Computer)',
'Occupational Health and Safety Specialist',
'Occupational Health and Safety Technician',
'Occupational Therapist',
'Occupational Therapy Assistant and Aide',
'Operation Research Analyst',
'Ophthalmic Medical Technician',
'Optician, Dispensing',
'Optometrist',
'Orderly',
'Orthotist',
'Painter (Construction and Maintenance)',
'Painting Worker',
'Paperhanger',
'Paralegal',
'Paramedic',
'Parking Enforcement Worker',
'Parking Attendant',
'Passenger Attendant',
'Passenger Vehicle Driver',
'Patternmaker (Wood)',
'Payroll Clerk',
'Pedicurist',
'Personal Care Aide',
'Personal Care and Service Worker (all other)',
'Personal Financial Advisor',
'Pest Control Worker',
'Petroleum Pump System Operator, Refinery Operator, and Gauger',
'Petroleum Technician',
'Pharmacist',
'Pharmacy Technician',
'Phlebotomist',
'Photogrammetrist',
'Photographer',
'Physical Scientist (all other)',
'Physical Therapist Assistant and Aide',
'Physical Therapist',
'Physician Assistant',
'Physician',
'Physicist',
'Pipefitter',
'Pipelayer',
'Plant and System Operator (all other)',
'Plasterer and Stucco Mason',
'Plastic Machine Worker',
'Plumber',
'Podiatrist',
'Police Officer and Detective',
'Police Dispatcher',
'Political Scientist',
'Postal Service Worker',
'Postmaster and Mail Superintendent',
'Postsecondary Education Administrator',
'Postsecondary Teacher',
'Postsecondary Teacher (all other)',
'Power Plant Operator, Distributor, and Dispatcher',
'Practical Nurse, Licensed',
'Preciou Stone and Metal Worker',
'Precision Instrument and Equipment Repairer (all other)',
'Prepress Technician and Worker',
'Preschool Director',
'Preschool Teacher',
'Presser (Textile, Garment, and Related Material)',
'Print Binding and Finishing Worker',
'Printing Press Operator',
'Private Detective and Investigator',
'Probation Officer',
'Producer - Film, Theater',
'Production Manager',
'Production Worker Helper',
'Production Worker (all other)',
'Promotion Manager',
'Proofreader and Copy Marker',
'Property Appraiser and Assessor',
'Property Manager',
'Prosthetist',
'Protective Service Worker (all other)',
'Psychiatric Technician and Aide',
'Psychologist',
'Public Safety Telecommunicator',
'Public Relation Manager',
'Public Relation Specialist',
'Pump Operator',
'Purchasing Agent and Manager',
'Quality Assurance Analyst',
'Quality Control Inspector',
'Quarry Rock Splitter',
'Radiation Therapist',
'Radio, Cellular, and Tower Equipment Installer and Repairer',
'Radiologic Technologist',
'Rail Transportation Worker (all other)',
'Rail-Track Laying and Maintenance Equipment Operator',
'Railroad Occupation',
'Rancher',
'Real Estate Appraiser and Assessor',
'Real Estate Broker and Sale Agent',
'Real Estate Manager',
'Receptionist',
'Recreation Worker',
'Recreational Therapist',
'Recreational Vehicle Service Technician',
'Referee, Umpire, and Other Sport Official',
'Refractory Material Repairer (Except Brickmason)',
'Refrigeration Mechanic and Installer',
'Registered Nurse',
'Rehabilitation Counselor',
'Reinforcing Iron and Rebar Worker',
'Religiou Activity and Education Director',
'Religiou Worker (all other)',
'Repair and Maintenance Worker, General',
'Reporter',
'Retail Sale Worker',
'Rigger',
'Roof Bolter (Mining)',
'Roofer',
'Rotary Drill Operator (Oil and Ga)',
'Roustabout (Oil and Ga)',
'Sale and Related Worker (all other)',
'Sale Manager',
'Sale Representative, Service (all other)',
'School Counselor and Advisor',
'School Principal - Elementary, Middle, and High',
'Science Technician',
'Secretary',
'Security Sale Agent',
'Security and Fire Alarm System Installer',
'Security Guard',
'Segmental Paver',
'Semi Truck Driver',
'Septic Tank Servicer and Sewer Pipe Cleaner',
'Service Unit Operator (Oil, Ga, and Mining)',
'Set and Exhibit Designer',
'Sewer (Hand)',
'Shampooer',
'Sheet Metal Worker',
'Shipping, Receiving, and Traffic Clerk',
'Shoe and Leather Worker and Repairer',
'Signal and Track Switch Repairer',
'Singer',
'Ski Patrol and Other Recreational Protective Service Worker',
'Skincare Specialist',
'Slaughterer',
'Small Engine Mechanic',
'Social Science Research Assistant',
'Social Scientist and Related Worker (all other)',
'Social Service Assistant',
'Social Service Manager',
'Social Worker',
'Sociologist',
'Software Developer',
'Software Engineer',
'Solar Photovoltaic Installer',
'Solderer',
'Sound Engineering Technician',
'Special Education Teacher',
'Special Effect Artist and Animator',
'Speech-Language Pathologist',
'Statistical Assistant',
'Statistician',
'Steamfitter',
'Steel Worker',
'Stone Setter',
'Stonemason',
'Substance Abuse Counselor',
'Subway and Streetcar Operator',
'Surgeon',
'Surgical Assistant and Surgical Technologist',
'Survey Researcher',
'Surveying Technician',
'Surveyor',
'Switchboard Operator (Including Answering Service)',
'System Analyst',
'Tailor, Dressmaker, and Custom Sewer',
'Tank Car, Truck, and Ship Loader',
'Taper (Drywall)',
'Tax Examiner and Collector, and Revenue Agent',
'Tax Preparer',
'Taxi Driver',
'Teacher Assistant',
'Adult Basic and Secondary Education and ESL Teacher',
'High School Teacher',
'Kindergarten and Elementary School Teacher',
'Middle School Teacher',
'Substitute Teacher',
'All Other Teacher and Instructor',
'Technical Writer',
'Telecommunication Equipment Installer and Repairer (Except Line Installer)',
'Telemarketer',
'Telephone Operator',
'Television, Video, and Motion Picture Camera Operator and Editor',
'Teller',
'Terrazzo Worker',
'Textile Career',
'Textile, Apparel, and Furnishing Worker (all other)',
'Therapist (all other)',
'Tile Setter',
'Timekeeping Clerk',
'Tire Builder',
'Tire Repairer and Changer',
'Tobacco Processing Worker',
'Tool and Die Maker',
'Tool Grinder, Filer, and Sharpener',
'Top Executive',
'Tour Guide and Escort',
'Traffic Technician',
'Training Manager',
'Training and Development Specialist',
'Translator',
'Transportation Attendant',
'Transportation Inspector',
'Transportation Security Screener',
'Transportation Worker (all other)',
'Transportation, Storage, and Distribution Manager',
'Travel Agent',
'Travel Clerk',
'Truck Driver - Delivery and Sale Worker',
'Truck Driver - Heavy and Tractor-trailer',
'Ultrasound Technician',
'Umpire, Referee, and Other Sport Official',
'Upholsterer',
'Urban and Regional Planner',
'Usher, Lobby Attendant, and Ticket Taker',
'Vascular Technologist',
'Vending, Coin, and Amusement Machine Servicer and Repairer',
'Veterinarian',
'Veterinary Assistant',
'Veterinary Technologist and Technician',
'Video Editor',
'Video Technician',
'Vocational Nurse, Licensed',
'Waiter or Waitress',
'Watch Repairer',
'Water and Wastewater Treatment Plant and System Operator',
'Water Transportation Occupation',
'Web Developer',
'Weigher, Measurer, Checker, and Sampler, Recordkeeping',
'Welder and Cutter',
'Wellhead Pumper',
'Wholesale Sale Representative',
'Wildlife Biologist',
'Wind Turbine Technician',
'Word Processor and Typist',
'Woodworker',
'Woodworker (all other)',
'Writer',
'X-Ray Technician',
'Yard Conductor',
'Yoga Instructor/Teacher',
'Youth Program Director',
'Youth Worker',
'Zookeeper',
'Zoologist',
]</code></pre>
