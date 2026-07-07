# Narrative Index — API Reference

> Base URL: `https://api.polyvaults.ai`
>
> All requests use JSON. Identify users by `userId` (UUID from `POST /auth/connect`).
>
> **Multi-asset**: Most endpoints accept an optional `asset` query/body parameter
> (default: `BTC`). Supported assets: BTC, ETH, SOL, OIL, GOLD, SILVER.
>
> **Product abstraction**: New surfaces should prefer `/products/:productKey/*`.
> Legacy `/index/*` endpoints remain for asset-direction compatibility.
>
> **Remote signing**: The main API does not hold KMS decrypt permission or
> plaintext owner keys. It calls an isolated signing-service over an internal
> authenticated channel. The signer can enforce mTLS, throttling, raw-tx deny,
> CLOB auth/order checks, and withdraw destination policy.
>
> **Geo-restriction**: New-position endpoints return HTTP 403 `GEO_RESTRICTED`
> in blocked or close-only regions. Close-only regions may still redeem and
> withdraw. Read-only endpoints are unaffected.
>
> **Rate limiting**: Global limits of 10 requests/second and 100 requests/minute
> per IP. Exceeding returns HTTP 429.

## Contents

- [Authentication](#authentication) — challenge, connect, session token, logout, Twitter
- [Wallet Management](#wallet-management) — balance, deposit address, withdraw-fee, withdraw
- [Market Status & Assets](#market-status--assets) — asset registry, market availability
- [Products](#products) — productKey-based INDEX/MANAGED products
- [World Cup Positions](#world-cup-positions) — auto-roll chains, locks, retry/stop
- [Index Investment](#index-investment) — preview, invest, positions, redeem
- [Performance](#performance) — monthly daily returns
- [Chart](#chart) — asset price + strike lines
- [Portfolio Dashboard](#portfolio-dashboard) — NAV, PnL, totalReturn, breakdown
- [Referral](#referral) — permanent referral code and rewards
- [Deposit Integrations](#deposit-integrations) — Fun.xyz / Relay.link proxies
- [Accounting](#accounting) — no public read endpoints; use portfolio APIs
- [Signature Authentication](#signature-authentication) — EIP-712 V2 signing
- [Enum Reference](#enum-reference) — all enum values

---

## Authentication

### GET /auth/challenge

Issue a one-time EIP-191 `personal_sign` challenge for wallet ownership proof.

**Query params:**

| Param | Required | Description |
|-------|----------|-------------|
| address | Yes | EVM wallet address (`0x...`) |

**Response:**

```json
{
  "challenge": "Sign this message to verify your wallet ownership.\n\nAddress: 0x...\nNonce: ...",
  "expiresAt": 1780314666123
}
```

The challenge is valid for 5 minutes and is consumed by `POST /auth/connect`.

### POST /auth/connect

Register or log in after signing the challenge.

**Request:**

```json
{
  "walletAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "signature": "0x...",
  "challenge": "Sign this message to verify your wallet ownership.\n\nAddress: 0x...\nNonce: ...",
  "inviteCode": "OPTIONAL",
  "chainId": 8453
}
```

| Field | Required | Description |
|-------|----------|-------------|
| walletAddress | Yes | EVM address that signed the challenge |
| signature | Yes | EIP-191 `personal_sign` of `challenge` |
| challenge | Yes | From `GET /auth/challenge` |
| inviteCode | No | Referrer's permanent code; binding only at first registration |
| chainId | No | Wallet's current chain (e.g. Base `8453`); speeds Smart Wallet verify |

**Response:**

```json
{
  "userId": "uuid",
  "walletAddress": "0x...",
  "safeAddress": "0x...",
  "depositAddress": "0x...",
  "isDeployed": true,
  "isApproved": true,
  "isNewUser": false,
  "twitterHandle": null,
  "sessionToken": "64-char-hex",
  "createdAt": "2026-03-12T08:00:00.000Z"
}
```

| Field | Description |
|-------|-------------|
| userId | Unique user ID for all subsequent calls |
| safeAddress | Platform-managed wallet address |
| depositAddress | Same as safeAddress; send USDC or USDC.e here |
| isNewUser | true on first connect |
| isDeployed / isApproved | Wallet provisioning status; retry connect if false |
| twitterHandle | Linked Twitter handle if available |
| sessionToken | 7-day bearer token for Campaign writes |

### POST /auth/logout

Invalidate the caller's session token. Idempotent.

**Headers:** `Authorization: Bearer <sessionToken>`

**Response:** `{ "ok": true }`

### POST /auth/twitter/link

Link a Twitter account via OAuth 2.0 PKCE. Requires EIP-712 `TwitterLink` signature.

**Request:**

```json
{
  "userId": "uuid",
  "code": "oauth-code",
  "redirectUri": "https://app.polyvaults.ai/auth/x/callback",
  "codeVerifier": "pkce-verifier",
  "signature": "0x...",
  "nonce": 1740643200000
}
```

### DELETE /auth/twitter/link

Unlink Twitter. Requires EIP-712 `TwitterUnlink` signature.

**Request:** `{ "userId": "uuid", "signature": "0x...", "nonce": 1740643200000 }`

### GET /auth/twitter/:userId

Check Twitter linking status. Returns `{ linked: true, twitterId, twitterHandle, ... }` or `{ linked: false }`.

---

## Wallet Management

### GET /wallets/:userId

Full wallet info (`ownerAddress`, `safeAddress`, `walletType`, `isDeployed`,
`isApproved`). `walletType` can be `DEPOSIT_WALLET` for Polymarket Deposit
Wallet users or `SAFE` for legacy Safe users.

### GET /wallets/:userId/balance

```json
{
  "usdcBalance": "1500000000",
  "formattedBalance": "1500.00",
  "nativeUsdcBalance": "500000000",
  "formattedNativeBalance": "500.00",
  "pusdBalance": "250000000",
  "formattedPusdBalance": "250.000000",
  "totalBalance": "2250.000000",
  "lockedBalance": "50.000000",
  "withdrawableBalance": "2200.000000"
}
```

| Field | Description |
|-------|-------------|
| usdcBalance / formattedBalance | USDC.e (bridged) balance |
| nativeUsdcBalance / formattedNativeBalance | Native USDC balance |
| pusdBalance / formattedPusdBalance | pUSD trading collateral balance |
| totalBalance | Sum of USDC.e + native USDC + pUSD |
| lockedBalance | Application-level cash lock for auto-roll chains |
| withdrawableBalance | max(0, totalBalance - lockedBalance); use this for max withdraw/new invest |

### GET /wallets/:userId/deposit-address

```json
{ "address": "0x...", "network": "Polygon", "token": "USDC / USDC.e" }
```

Both USDC and USDC.e deposits are accepted. Trading uses pUSD; investment
preparation wraps/converts collateral as needed.

### GET /wallets/withdraw-fee

```json
{ "feeRate": 0.01, "feeToken": "USDC", "network": "Polygon", "minWithdrawal": 0.01 }
```

### POST /wallets/withdraw

Supports Polygon local transfer and cross-chain withdrawal via Polymarket Bridge.

> Requires EIP-712 signature (`Withdraw` type). See [Signature Authentication](#signature-authentication).

**Request:**

```json
{
  "userId": "uuid",
  "toAddress": "0x...",
  "amount": 100,
  "token": "pUSD",
  "chain": "ethereum",
  "previewEstimatedOutput": 99.9,
  "slippage": 0.02,
  "signature": "0x...",
  "nonce": 1740643200000
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| userId | string | Yes | User ID |
| toAddress | string | Yes | Destination address (EVM `0x...` or Solana base58) |
| amount | number | Yes | Maximum authorized withdrawal amount in dollars, min $0.01. The actual transfer may be lower because of available balance/precision, but must never exceed this value. |
| token | string | No | Polygon only: `"USDC"`, `"USDC.e"`, or `"pUSD"` |
| chain | string | No | Target chain (default `"polygon"`). Options: `polygon`, `ethereum`, `arbitrum`, `base`, `optimism`, `bsc`, `solana` |
| previewEstimatedOutput | number | No | Cross-chain quote baseline from `withdraw-quote` |
| slippage | number | No | Cross-chain max slippage, 0.001–0.1, default 0.02 |
| signature | string | Yes | EIP-712 signature |
| nonce | number | Yes | `Date.now()` millisecond timestamp (single-use) |

For Polygon: transfers the selected token. For cross-chain: consolidates via
the bridge flow and validates current quote when preview fields are provided.

**Response — Polygon:**

```json
{
  "transactionHash": "0x...", "from": "0x...", "to": "0x...",
  "amount": "100.000000", "status": "SUBMITTED"
}
```

**Response — Cross-chain:**

```json
{
  "transactionHash": "0x...", "from": "0x...", "to": "0x...",
  "amount": "100.000000", "status": "BRIDGING",
  "chain": "Ethereum", "bridgeDepositAddress": "0x..."
}
```

### POST /wallets/withdraw-quote

Preview cross-chain fees and estimated arrival time.

```json
{ "userId": "uuid", "amount": 100, "chain": "ethereum", "recipientAddress": "0x..." }
```

**Response:**

```json
{
  "chain": "Ethereum", "inputAmount": 100, "estimatedOutput": 99.99,
  "estimatedOutputBaseUnit": "99990000",
  "fees": { "gasUsd": 0.003, "totalImpactUsd": 0 },
  "estimatedTimeMs": 25000, "minWithdrawal": 7,
  "quoteId": "quote-id",
  "priceDisclaimer": "The estimated output may fluctuate slightly between preview and execution due to market and routing changes. This is not a final locked value."
}
```

### GET /wallets/withdraw-status/:address

Track cross-chain withdrawal using `bridgeDepositAddress`.

**Response:**

```json
{
  "transactions": [{
    "fromChainId": "137", "toChainId": "1",
    "fromAmountBaseUnit": "100000000",
    "status": "COMPLETED", "txHash": "0x..."
  }]
}
```

Statuses: `DEPOSIT_DETECTED` → `PROCESSING` → `ORIGIN_TX_CONFIRMED` → `SUBMITTED` → `COMPLETED` | `FAILED`

### GET /wallets/supported-chains

Lists supported withdrawal chains.

```json
[
  { "id": "polygon", "name": "Polygon", "chainId": "137", "minWithdrawal": 2, "addressType": "evm" },
  { "id": "ethereum", "name": "Ethereum", "chainId": "1", "minWithdrawal": 7, "addressType": "evm" },
  { "id": "solana", "name": "Solana", "chainId": "1151111081099710", "minWithdrawal": 2, "addressType": "svm" }
]
```

---

## Market Status & Assets

### GET /market/status

Check prediction market availability. Supports single asset or all assets.

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| asset | string | No | Asset symbol (e.g. `BTC`, `OIL`). When omitted, returns all assets. |

**Response (single asset):**

```json
{
  "available": true,
  "asset": "BTC",
  "status": "active",
  "month": "April",
  "year": 2026,
  "slug": "what-price-will-bitcoin-hit-in-april-2026",
  "title": "What price will Bitcoin hit in April?",
  "marketsCount": 17,
  "totalOpenInterest": 250000
}
```

**Response (all assets — no `asset` param):** Array of the above objects.

| Field | Description |
|-------|-------------|
| available | Whether the current month's market exists |
| status | `active` / `pending_liquidity` / `coming_soon` |
| totalOpenInterest | Sum of open interest across all markets for this asset |

When unavailable: `available: false`, `slug: null`, `title: null`, `marketsCount: 0`.

**Errors:**

| Code | When |
|------|------|
| 400  | `The [Month Year] [asset] prediction market is not yet available on Polymarket.` — also applies to `POST /index/preview`, `POST /index/invest`, and `GET /chart/strikes` |

### GET /assets

List all registered assets with their current status.

**Response:**

```json
{
  "assets": [
    { "symbol": "BTC", "name": "Bitcoin", "category": "CRYPTO", "status": "active" },
    { "symbol": "ETH", "name": "Ethereum", "category": "CRYPTO", "status": "active" },
    { "symbol": "OIL", "name": "Crude Oil", "category": "ENERGY", "status": "active" },
    { "symbol": "GOLD", "name": "Gold", "category": "METALS", "status": "coming_soon" }
  ]
}
```

| Field | Description |
|-------|-------------|
| status | `active` = live market, can invest. `pending_liquidity` = market exists but low OI. `coming_soon` = registered, no market yet. |
| category | `CRYPTO`, `ENERGY`, or `METALS` |

---

## Products

The current product abstraction uses `productKey` to address both legacy
asset-direction indices and managed products.

### GET /products

Returns all registered INDEX and MANAGED products.

```
GET /products
```

```json
[
  { "productKey": "btc-bullish", "productKind": "INDEX", "displayName": "BTC Bullish", "asset": "BTC", "indexDirection": "BULLISH" },
  { "productKey": "narrative-basket:taco-v1", "productKind": "MANAGED", "displayName": "TACO Index" }
]
```

When a `productKey` contains `:`, URL-encode it in the path
(`narrative-basket%3Ataco-v1`) but keep the raw value in body/signature fields.

### GET /products/:productKey/definition

Returns product metadata. `INDEX` products return `asset` and `indexDirection`;
managed basket products may also return a `basket` definition.

### GET /products/:productKey/health

Returns tradability.

- `INDEX`: `{ productKey, productKind: "INDEX", tradable, reason? }`
- `MANAGED`: per-strike health snapshot with `clusters[]`, `eligibleStrikes`,
  OI, buy prices, and `dropReason`.

### POST /products/:productKey/health/preview

Same shape as `GET /health`, but accepts product-specific `overrides`. Use this
for World Cup custom baskets with `overrides.worldCup.teamRefs`.

### POST /products/:productKey/preview

Preview a product investment.

```json
{
  "amount": 100,
  "userId": "uuid",
  "overrides": {
    "worldCup": {
      "stageKey": "r48-32",
      "teamRefs": ["BRAZIL", "ARGENTINA"],
      "exitAfterStageKey": "final"
    }
  }
}
```

Response fields include `productKey`, `productKind`, `totalDeposit`,
`effectiveAmount`, `swapFee`, `totalAllocated`, `items[]`, `droppedStrikes[]`,
`minimumDepositRequired`, `eventSlug`, and `eventTitle`.

World Cup custom/preset override responses can also include
`normalizedWorldCupConfig` and `strategyHash`. Keep the exact `strategyHash` for
the subsequent `invest` signature.

### POST /products/:productKey/invest

Execute product investment. Requires EIP-712 signature.

```json
{
  "userId": "uuid",
  "productKey": "worldcup-2026:custom:r48-32",
  "amount": 100,
  "slippage": 0.02,
  "autoCompound": true,
  "strategyHash": "0x...",
  "overrides": { "worldCup": { "teamRefs": ["BRAZIL"] } },
  "signature": "0x...",
  "nonce": 1780314666123
}
```

| Field | Required | Description |
|-------|----------|-------------|
| productKey | Yes | Must match the URL path and signed message |
| amount | Yes | USD amount; business minimum is $10 |
| slippage | No | 0.001–0.1, default 0.02 |
| autoCompound | No | Enables TACO reinvest or World Cup auto-roll when supported; **must be signed** in EIP-712 |
| strategyHash | Conditional | Required for `overrides.worldCup` invests |
| overrides | No | Product-specific options such as World Cup teams/exit stage |
| signatureChainId | No | Wallet signing chain (e.g. Base `8453`); defaults to Polygon `137` |
| fundsChainId | No | Funds execution chain; currently always `137` (Polygon) |

Use `ProductInvest` signing for ordinary INDEX/MANAGED products. Use
`ProductInvestConfigured` when the request contains `overrides.worldCup`.
`autoCompound` is part of the typed-data message (sign `false` when disabled).
There is no per-user or platform active-position cap; auto-compounding can roll
growing balances forward as long as the product remains eligible and collateral
is available.

Response fields mirror preview and add `depositId`, `hasPlacedOrders`,
`overallStatus`, `createdAt`, and per-item order execution fields such as
`orderId`, `orderStatus`, `filledAmount`, `filledShares`, `closeStatus`,
`closedShares`, `remainingShares`, and `failReason`.

Early-exit all FILLED positions for a product. Requires EIP-712
`ProductRedeem`.

If the product has no `FILLED` positions but the user has an active World Cup
auto-roll chain with locked cash, the backend may **fallback** to stop-rolling
semantics (close `autoCompound` + release lock) instead of returning 400.

```json
{ "userId": "uuid", "productKey": "narrative-basket:taco-v1", "slippage": 0.02, "signature": "0x...", "nonce": 1780314719456, "signatureChainId": 137, "fundsChainId": 137 }
```

Response: `sold`, `totalReceived`, `totalCost`, `profit`, `fee`,
`closeStatus`, `results[]`. Fee is 5% of positive profit (2.5% platform +
2.5% referrer when the user has a referrer).

### POST /products/:productKey/stop-rolling

Stops a **specific** World Cup auto-roll chain and releases application-level
locked cash. Requires EIP-712 `ProductStopRolling` signature. Use when
`rollRetry.canStopRolling=true` on the positions endpoint — typically when
funds are stuck in `round_gap` waiting for the next stage or retry.

```json
{
  "userId": "uuid",
  "productKey": "worldcup-2026:custom:r48-32",
  "rootDepositId": "dep-root-uuid",
  "signature": "0x...",
  "nonce": 1780314719456,
  "signatureChainId": 137,
  "fundsChainId": 137
}
```

`rootDepositId` scopes the action to one chain; without it sibling chains at
the same `productKey` could be affected.

### POST /products/:productKey/retry-roll

Manually retry a failed World Cup auto-roll for a specific chain. Requires
EIP-712 `ProductRetryRoll` signature. Use when `rollRetry.canRetry=true`.

```json
{
  "userId": "uuid",
  "productKey": "worldcup-2026:host-nations:r48-32",
  "rootDepositId": "dep-root-uuid",
  "signature": "0x...",
  "nonce": 1780314719456,
  "signatureChainId": 137,
  "fundsChainId": 137
}
```

### POST /products/:productKey/potential-return

World Cup only: multi-round idealized return ladder. Runs the same allocation
as preview, then simulates auto-roll through the exit stage using current
Polymarket prices. Geo-restricted like preview.

```json
{
  "amount": 100,
  "userId": "uuid",
  "overrides": { "worldCup": { "teamRefs": ["BRAZIL"], "exitAfterStageKey": "final" } }
}
```

### GET /products/worldcup-2026/positions

Read-only dashboard view of all World Cup chains for a user: current stage,
exit stage, locks, roll retry state, and per-round deposits.

```
GET /products/worldcup-2026/positions?userId=uuid
```

Key response fields per chain: `rootDepositId`, `chainStatus`, `exitAfterStageKey`,
`willAutoRoll`, `lock`, `rollRetry.canRetry`, `rollRetry.canStopRolling`.

### GET /products/worldcup-2026/catalog

Returns World Cup entry presets, custom basket options, teams, exit stages, and
the schedule gate. Use it before World Cup preview/invest UI.

### GET /portfolio/products

Product-level holdings list.

```
GET /portfolio/products?userId=uuid&family=worldcup-2026
```

### GET /portfolio/products/:productKey

Single product portfolio. Returns standard metrics plus `clusters[]` for
managed products and `teams[]` for World Cup products.

### GET /portfolio/worldcup-2026/dashboard-groups

Variant-level World Cup dashboard (Custom, European Teams, etc.) with
aggregated P&L, total return, and `returnChart`.

```
GET /portfolio/worldcup-2026/dashboard-groups?userId=uuid&timeRange=all
```

---

## World Cup Positions

See `GET /products/worldcup-2026/positions` above. Use it to drive dashboard
UI for auto-roll chains:

| Field | Description |
|-------|-------------|
| chainStatus | `ACTIVE` / `PENDING_ROLL` / `TERMINAL` / `RELEASED` |
| exitAfterStageKey | User-selected exit point (e.g. `final`, `semifinals`) |
| lock.status | `ACTIVE` when cash is earmarked in `round_gap` |
| rollRetry.canRetry | Show manual retry-roll button |
| rollRetry.canStopRolling | Show stop-rolling button to release lock |

Preset entry products include `worldcup-2026:europe:r48-32`,
`worldcup-2026:south-am:r48-32`, `worldcup-2026:host-nations:r48-32`,
`worldcup-2026:top-seeded:r48-32`, and `worldcup-2026:custom:r48-32`.
Custom baskets pass `overrides.worldCup.teamRefs` and optional
`exitAfterStageKey`.

---

## Index Investment

Legacy `/index/*` endpoints remain for asset-direction products. New product
pages should prefer `/products/:productKey/*`.

### POST /index/preview

Preview strike allocation before investing.

**Request:**

```json
{ "indexType": "BULLISH", "amount": 100, "userId": "abc-123", "asset": "BTC" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| indexType | enum | Yes | BULLISH or BEARISH |
| amount | number | Yes | Preview amount ($), DTO minimum 1 |
| asset | string | No | Asset symbol (default: BTC) |
| eventSlug | string | No | Override default current-month event |
| userId | string | No | When provided, estimates collateral preparation fees from the user's balances |

**Response:**

```json
{
  "asset": "BTC",
  "indexType": "BULLISH",
  "totalDeposit": 100,
  "effectiveAmount": 99.95,
  "swapFee": 0.05,
  "totalAllocated": 99.95,
  "allocations": [
    {
      "strikePrice": 90000,
      "direction": "UP",
      "groupItemTitle": "↑ 90,000",
      "buyDirection": "YES",
      "tokenId": "12345...",
      "weight": 0.35,
      "allocation": 34.98
    }
  ],
  "droppedStrikes": ["↑ 120,000"],
  "resolvedStrikes": [
    { "strikePrice": 75000, "direction": "UP", "groupItemTitle": "↑ 75,000" }
  ],
  "minimumDepositRequired": 5,
  "eventTitle": "What price will Bitcoin hit in March?"
}
```

| Field | Description |
|-------|-------------|
| asset | Asset symbol for this allocation |
| totalDeposit | Requested investment amount |
| effectiveAmount | Amount after deducting swap fee (= totalDeposit if no swap needed) |
| swapFee | Swap fee (0.1% of USDC amount needing conversion; 0 if no swap) |
| allocations | Strikes passing iterative pruning (each >= $1) |
| weight | Proportion of total weight (0–1) |
| allocation | Dollar amount assigned (based on effectiveAmount) |
| buyDirection | YES or NO |
| droppedStrikes | Active strikes pruned due to insufficient allocation |
| resolvedStrikes | Already-settled strikes (all directions) |
| minimumDepositRequired | Minimum deposit to keep all strikes |

### POST /index/invest

Execute index investment. Uses FAK (Fill-and-Kill) market orders that fill
immediately against available liquidity. If pUSD balance is insufficient, the
backend prepares collateral by wrapping USDC.e and, if needed, swapping native
USDC to USDC.e first.

> Requires EIP-712 signature (`Invest` type). See [Signature Authentication](#signature-authentication).

**Request:**

```json
{ "userId": "uuid", "indexType": "BULLISH", "amount": 100, "asset": "BTC", "slippage": 0.02, "signature": "0x...", "nonce": 1740643200000 }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| userId | string | Yes | User ID |
| indexType | enum | Yes | BULLISH or BEARISH |
| amount | number | Yes | Investment amount ($), min 10 |
| asset | string | No | Asset symbol (default: BTC) |
| eventSlug | string | No | Override default current-month event |
| slippage | number | No | 0.001–0.1, default 0.02 |
| signature | string | Yes | EIP-712 signature |
| nonce | number | Yes | `Date.now()` millisecond timestamp (single-use) |

**Response:**

```json
{
  "depositId": "uuid",
  "userId": "user-123",
  "asset": "BTC",
  "indexType": "BULLISH",
  "totalDeposit": 100,
  "totalAllocated": 98.50,
  "hasPlacedOrders": false,
  "allocations": [
    {
      "strikePrice": 90000,
      "direction": "UP",
      "groupItemTitle": "↑ 90,000",
      "buyDirection": "YES",
      "tokenId": "12345...",
      "weight": 0.35,
      "allocation": 35.00,
      "orderId": "order-abc",
      "orderStatus": "FILLED",
      "filledAmount": 35.00,
      "filledShares": 50.0,
      "closeStatus": null,
      "closedShares": 0,
      "remainingShares": 50.0,
      "failReason": null
    }
  ],
  "droppedStrikes": ["↑ 120,000"],
  "overallStatus": "SUCCESS",
  "eventTitle": "What price will Bitcoin hit in March?",
  "createdAt": "2026-03-12T08:00:00.000Z"
}
```

| Field | Description |
|-------|-------------|
| overallStatus | SUCCESS (no failures) / PARTIAL (some failed) / FAILED (all failed) |

**Errors:**

| HTTP | Message |
|------|---------|
| 400 | Insufficient balance (USDC.e + native USDC combined) |
| 400 | Minimum investment is $10 |
| 400 | No active markets available |

### GET /index/positions/:userId

All index deposits for the user.

**Response:**

```json
[
  {
    "id": "uuid",
    "userId": "user-123",
    "asset": "BTC",
    "indexType": "BULLISH",
    "depositAmount": 100,
    "eventSlug": "what-price-will-bitcoin-hit-in-march-2026",
    "eventTitle": "What price will Bitcoin hit in March?",
    "status": "COMPLETED",
    "allocations": [ ... ],
    "createdAt": "2026-03-12T08:00:00.000Z"
  }
]
```

**status enum:** PENDING | EXECUTING | COMPLETED | PARTIAL | FAILED

### POST /index/redeem

Early redeem all active positions for a direction. Market-sells via CLOB FAK
orders. A 5% fee is charged on positive profit.

> Requires EIP-712 signature (`Redeem` type). See [Signature Authentication](#signature-authentication).

**Request:**

```json
{ "userId": "uuid", "direction": "BULLISH", "asset": "BTC", "slippage": 0.02, "signature": "0x...", "nonce": 1740643200000 }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| userId | string | Yes | User ID |
| direction | enum | Yes | BULLISH or BEARISH |
| asset | string | Yes | Asset symbol; required by signature validation |
| slippage | number | No | 0.001–0.1, default 0.02 |
| signature | string | Yes | EIP-712 signature |
| nonce | number | Yes | `Date.now()` millisecond timestamp (single-use) |

**Response:**

```json
{
  "sold": 4,
  "totalReceived": 35.12,
  "totalCost": 30.00,
  "profit": 5.12,
  "fee": 0.26,
  "closeStatus": "COMPLETED",
  "results": [
    {
      "tokenId": "12345...",
      "title": "What price will Bitcoin hit in March?",
      "groupItemTitle": "↑ 90,000",
      "shares": 50.5,
      "requestedShares": 50.5,
      "soldShares": 50.5,
      "closedShares": 50.5,
      "remainingShares": 0,
      "sellPrice": 0.65,
      "received": 32.83,
      "status": "SOLD",
      "closeStatus": "COMPLETED",
      "nextRetryAt": null
    }
  ]
}
```

| Field | Description |
|-------|-------------|
| sold | Number of allocations successfully sold |
| totalReceived | Actual USDC.e received (on-chain balance delta) |
| profit | totalReceived - totalCost |
| fee | 5% of positive profit |
| closeStatus | COMPLETED / RETRYING / NEEDS_REVIEW / PENDING / NO_ACTION |
| results[].status | SOLD / PLACED / REJECTED / NO_BIDS / SKIPPED / ERROR |

**Auto-redemption**: Resolved markets are automatically scanned every 15
minutes by a cron job. This endpoint is only for pre-settlement exits.

---

## Performance

### GET /performance/returns

Daily return data. Limited to the last 6 months.

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| month | string | Yes | Format: `YYYY-MM` |
| asset | string | No | Asset symbol (default: BTC) |
| live | boolean | No | Calculate live values where supported |

**Response:**

```json
[
  {
    "date": "2026-03-01",
    "asset": "BTC",
    "btcPrice": 87500.00,
    "btcReturn": 0.0,
    "bullishReturn": 0.0,
    "bearishReturn": 0.0,
    "bullishOutperformance": 0.0,
    "bearishOutperformance": 0.0
  }
]
```

| Field | Description |
|-------|-------------|
| asset | Asset symbol |
| btcPrice | Asset spot price (named `btcPrice` for backward compatibility) |
| btcReturn | Asset cumulative return from month start (0.008 = 0.8%) |
| bullishReturn | Bullish Index cumulative return |
| bearishReturn | Bearish Index cumulative return |
| *Outperformance | indexReturn - assetReturn |

---

## Chart

### GET /chart/strikes

Hourly price data + strike price lines for any supported asset.

Legacy alias: `GET /chart/btc-strikes` (same behavior, defaults to BTC).

**Query params:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| indexType | enum | Yes | — | BULLISH or BEARISH |
| asset | string | No | BTC | Asset symbol |
| eventSlug | string | No | current month | Event slug |
| from | string | No | month start | ISO 8601 start time |
| to | string | No | now | ISO 8601 end time |

**Response:**

```json
{
  "asset": "BTC",
  "priceData": [
    { "timestamp": "2026-03-01T00:00:00.000Z", "close": 87500.00 }
  ],
  "strikePrices": [
    {
      "strikePrice": 90000,
      "direction": "UP",
      "groupItemTitle": "↑ 90,000",
      "resolved": false,
      "hitDate": null
    }
  ],
  "nextStrike": "↑ 90,000",
  "resolved": "1/3",
  "eventTitle": "What price will Bitcoin hit in March?"
}
```

strikePrices includes both UP and DOWN directions regardless of indexType.

---

## Portfolio Dashboard

### GET /portfolio

**Query params:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| userId | string | Yes | — | User ID |
| asset | string | No | — | Filter to specific asset. When provided, includes per-direction breakdown. |
| timeRange | enum | No | all | 24h / 7d / 30d / all |

**Response (no asset filter):**

```json
{
  "nav": 1520.50,
  "deployedPrincipal": 1000.00,
  "positionValue": 1020.50,
  "availableBalance": 500.00,
  "unrealizedPnl": 15.50,
  "realizedPnl": 5.00,
  "pnl": 20.50,
  "untrackedPnl": 0,
  "totalReturn": 0.0205,
  "returnChart": [
    { "timestamp": "2026-03-12T01:00:00.000Z", "totalReturn": 0.018 }
  ],
  "timeRange": "24h"
}
```

**Response (with `asset` filter):** Same fields plus `asset`, `bullish`, and `bearish` direction breakdowns.

| Field | Description |
|-------|-------------|
| nav | Net asset value = position market value + available balance |
| deployedPrincipal | Active + redeemed positions' cost |
| positionValue | Current market value of active positions |
| availableBalance | Available wallet collateral/cash balance in USD terms |
| unrealizedPnl | Active position value - active position cost |
| realizedPnl | Sum of redeemed amounts - redeemed position cost |
| pnl | unrealizedPnl + realizedPnl |
| untrackedPnl | Polymarket Data API PnL for wallet positions/closed positions not tracked by DB allocations |
| totalReturn | pnl / deployedPrincipal (0.0205 = 2.05%) |

returnChart granularity: 24h → hourly, 7d/30d/all → daily.

### GET /portfolio/breakdown

Per-direction (BULLISH / BEARISH) investment metrics.

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| userId | string | Yes | User ID |
| asset | string | No | Filter to specific asset. When omitted, returns overall + per-asset `assets[]`. |

**Response (with `asset`):**

```json
{
  "bullish": {
    "deployedPrincipal": 500.00,
    "positionValue": 520.50,
    "unrealizedPnl": 20.50,
    "totalReturn": 0.041
  },
  "bearish": {
    "deployedPrincipal": 300.00,
    "positionValue": 285.00,
    "unrealizedPnl": -15.00,
    "totalReturn": -0.0333
  }
}
```

**Response (without `asset`):** Same structure plus `assets[]` array containing per-asset breakdowns.

---

## Referral

### GET /referral/:userId

Permanent referral code and dashboard data.

```json
{
  "referralCode": "A3X7K2",
  "referralUrl": "https://polyvaults.ai?ref=A3X7K2",
  "totalReferrals": 1,
  "monthlyDeposits": {
    "currentMonth": 1500.00,
    "history": [{ "month": "2026-04", "volume": 1500.00, "count": 3 }]
  },
  "rewards": {
    "totalAccrued": 125.50,
    "totalPaid": 75.00,
    "pending": 50.50,
    "processing": 30.00
  }
}
```

| Rule | Value |
|------|-------|
| Referral binding | At registration only via `inviteCode` in `POST /auth/connect` |
| Referral fee | 2.5% of referred user's profit (split from the 5% redemption fee) |
| Payout | Monthly cron flags due rewards; actual transfer is manual operator-confirmed |

Invalid `inviteCode` does **not** block registration — user simply won't be bound.

---

## Deposit Integrations

Backend proxies for third-party deposit channels. Require server-side proxy
tokens (`POLYVAULTS_FUN_PROXY_TOKEN` / `POLYVAULTS_RELAY_PROXY_TOKEN`); return
503 `NOT_CONFIGURED` when unset.

### Fun.xyz — `/api/integrations/fun/*`

| Endpoint | Description |
|----------|-------------|
| `GET /api/integrations/fun/assets/allow` | Allowed deposit assets |
| `GET /api/integrations/fun/assets/supported` | Supported assets list |
| `POST /api/integrations/fun/eoa` | Resolve Fun EOA for deposit |
| `POST /api/integrations/fun/fops` | Create Fun funding operation |
| `GET /api/integrations/fun/asset/erc20/price/:chainId/:token` | Token price |

### Relay.link — `/api/integrations/relay/*`

| Endpoint | Description |
|----------|-------------|
| `GET /api/integrations/relay/chains` | Supported chains |
| `POST /api/integrations/relay/quote` | Deposit quote |
| `POST /api/integrations/relay/routable-options` | Routable asset options |
| `GET /api/integrations/relay/requests` | Track deposit requests |
| `POST /api/integrations/relay/deposit-address/reindex` | Reindex deposit address |

---

## Accounting

The current backend does not expose public accounting read endpoints. Use:

- `GET /portfolio` for aggregate NAV/PnL
- `GET /portfolio/breakdown` for legacy BULLISH/BEARISH metrics
- `GET /portfolio/products` and `GET /portfolio/products/:productKey` for product-level views

---

## Signature Authentication

Write endpoints for fund movement (`invest`, `withdraw`, `redeem`,
`stop-rolling`, `retry-roll`) require EIP-712 typed data signatures from the
user's connected wallet. The backend recovers the signer address and compares
it to the registered user wallet or wallet owner address. Smart wallets
(Coinbase Smart Wallet, Safe) on Base/Ethereum/Polygon are supported via
ERC-1271 verification.

### EIP-712 V2 Domain (recommended)

```json
{ "name": "Polyvaults", "version": "2", "chainId": <signatureChainId> }
```

- `domain.chainId` = wallet signing chain (e.g. Base `8453`, Polygon `137`)
- `message.fundsChainId` = funds execution chain; currently always `137` (Polygon)
- Request body should include `signatureChainId` and `fundsChainId: 137`

Legacy V1 (`version: "1"`, `chainId: 137`, no `fundsChainId`) is still accepted
during rollout but unsuitable for Base Smart Wallet users.

**Types (V2 — all fund actions include `fundsChainId: uint256` before `nonce`):**

| Action | Fields |
|--------|--------|
| Invest | `action: "invest"`, `userId`, `indexType`, `amount: uint256` (6 decimals), `fundsChainId`, `nonce` |
| Withdraw | `action: "withdraw"`, `userId`, `toAddress`, `amount: uint256` (max authorized), `token`, `chain`, `fundsChainId`, `nonce` |
| Redeem | `action: "redeem"`, `userId`, `direction`, `asset`, `fundsChainId`, `nonce` |
| ProductInvest | `action: "productInvest"`, `userId`, `productKey`, `amount: uint256`, `autoCompound: bool`, `fundsChainId`, `nonce` |
| ProductInvestConfigured | `action: "productInvestConfigured"`, `userId`, `productKey`, `amount: uint256`, `strategyHash: bytes32`, `autoCompound: bool`, `fundsChainId`, `nonce` |
| ProductRedeem | `action: "productRedeem"`, `userId`, `productKey`, `fundsChainId`, `nonce` |
| ProductStopRolling | `action: "productStopRolling"`, `userId`, `productKey`, `rootDepositId`, `fundsChainId`, `nonce` |
| ProductRetryRoll | `action: "productRetryRoll"`, `userId`, `productKey`, `rootDepositId`, `fundsChainId`, `nonce` |
| TwitterLink | `action: "twitterLink"`, `userId`, `nonce` |
| TwitterUnlink | `action: "twitterUnlink"`, `userId`, `nonce` |

Use `ProductInvestConfigured` whenever `POST /products/:productKey/invest`
contains `overrides.worldCup`; the hash must come from preview or definition
and match the normalized configuration. Sign `autoCompound: false`
explicitly when the checkbox is unchecked.

Withdraw: sign the **final** `token` and `chain` values that will be sent in
the request body (defaults: `USDC.e` / `polygon`).

### Session Token (non-fund writes)

Campaign write endpoints accept `Authorization: Bearer <sessionToken>` instead
of per-request EIP-712. Token is issued by `POST /auth/connect` (7-day TTL).
Revoke via `POST /auth/logout`. Fund operations still require EIP-712 every time.

### Internal Signing Service Notes

These are operator/debugging notes for CLOB and relayed transaction failures:

- Main API signs through `RemoteSigner`; plaintext owner EOA keys never appear
  in the main app process.
- CLOB API key derivation (`ClobAuth`) must be signed by the owner EOA. Do not
  set CLOB L1 `POLY_ADDRESS` to a Deposit Wallet or Safe contract address; it
  causes `Invalid L1 Request headers` / malformed API credentials.
- Deposit Wallet order signing uses `POLY_1271`; Safe users use
  `POLY_GNOSIS_SAFE`.
- Signing-service policy mode is `off | audit | enforce`. In `audit`, denied
  policy decisions are logged but not blocked; in `enforce`, disallowed
  destinations/selectors/CLOB domains are rejected.
- Raw transaction signing is disabled by default (`SIGNER_ALLOW_RAW_TX=false`).
- Withdraw signatures are re-checked by the signing-service for destination,
  maximum amount, token, and chain binding before external transfers are signed.

**Nonce**: Use `Date.now()` (millisecond timestamp). Valid from
`now - 5 minutes` through a small future skew (default 2 seconds). **Each
`(userId, nonce)` is single-use** — reusing a nonce returns 400
"Nonce already used".

**Errors:**

| HTTP | Message |
|------|---------|
| 400 | Missing signature/userId/nonce |
| 400 | Signature expired or nonce too far in the future |
| 400 | Nonce already used (replay protection) |
| 401 | Signature does not match user wallet address |

---

## Enum Reference

| Enum | Values |
|------|--------|
| IndexType | BULLISH, BEARISH |
| AssetCategory | CRYPTO, ENERGY, METALS |
| AssetStatus | active, pending_liquidity, coming_soon |
| PortfolioTimeRange | 24h, 7d, 30d, all |
| OrderSide | BUY, SELL |
| OrderType | GTC, GTD, FOK, FAK |
| IndexDepositStatus | PENDING, EXECUTING, COMPLETED, PARTIAL, FAILED |
| IndexAllocationStatus | PENDING, PLACED, FILLED, FAILED, SOLD, REDEEMED |
| TradeStatus | PENDING, MATCHED, MINED, CONFIRMED, FAILED |
