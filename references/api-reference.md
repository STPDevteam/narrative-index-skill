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
> **Custody**: Owner keys are held in isolated infrastructure and are never
> exposed to API clients or agents. Fund-moving actions require your EIP-712
> signature.
>
> **Geo-restriction**: New-position endpoints return HTTP 403 `GEO_RESTRICTED`
> in blocked or close-only regions. Close-only regions may still redeem and
> withdraw. Read-only endpoints are unaffected.
>
> **Rate limiting**: Global limits of 10 requests/second and 100 requests/minute
> per IP. Exceeding returns HTTP 429.

## Contents

- [Authentication](#authentication) — challenge, connect, session token, logout, Twitter
- [Wallet Management](#wallet-management) — balance, locked funds, deposit address, withdraw
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
- [Custom Indexes](#custom-indexes) — compose, save/fork, revisions, paper performance, lifecycle, catalog, Agent init, ERC-8004, x402 deposits
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
| sessionToken | 7-day bearer for Campaign, custom-index, locked-funds reads, x402 deposits |

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
| lockedBalance | Application-level locks (World Cup auto-roll, pending BUY, Agent cash/fees). Itemize with `GET /wallets/me/locked-funds`. |
| withdrawableBalance | Spendable amount after locks, computed per token by the same rule (`computeSpendable`) that gates withdraw and invest, so it is never more optimistic than the server. Use it for Max withdraw/new invest — not `totalBalance − lockedBalance` |
| withdrawBlockedReason | Optional. `agent_fee_reserve_unbacked` (Agent fee reserve exceeds the wallet's native USDC) or `agent_funding_unbacked` (initial funding lacks backing). Present only when the whole wallet is treated as 0 spendable |
| withdrawBlockedDetail | Optional user-facing English explanation for `withdrawBlockedReason`. The 400 `Insufficient withdrawable balance` message appends the same text (match on the prefix) |

### GET /wallets/me/locked-funds

Session only (`Authorization: Bearer <sessionToken>`). `Cache-Control: private, no-store`.
`summary.totalLocked` matches `lockedBalance`.

Items have `id`, `kind`, `amount`, `status`, `releasable`, `releaseAction`,
`blockedReason`, `context`, `updatedAt`.

`kind`: `round_gap` | `pending_buy` | `series_cash` | `fee_reserve` |
`pending_fee` | `automation_funding` | `unresolved_settlement`.

Agent rows include `context.agents[]` (`seriesId`, `displayName`, `status`)
and `managementScope: series | shared`. Those amounts are **not** released
by this API — use automation disable or custom-index redeem.

### POST /wallets/me/locked-funds/release

Session + geo (close-only allowed) + EIP-712 `ReleaseLockedFunds`.

Normalize `itemIds` (trim, drop empty, dedupe, Unicode sort), then
`itemIdsHash = keccak256(utf8(JSON.stringify(ids)))`. Sign
`action="releaseLockedFunds"`, `userId`, `itemIdsHash`,
`acknowledgeRisk=true`, `fundsChainId`, `nonce`.

Per-item `result`: `released` | `reconciled` | `rejected` | `unchanged`.
`reconciled` books a proven BUY as `FILLED` (spent, not refunded).
`pending_buy` force-unlock: owner manual BUY, still locked, `NEEDS_REVIEW`,
age ≥ 7 days, no positive fill/position evidence.

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
asset-direction Indexes and managed products.

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
| Payout | Rewards are flagged monthly; the actual transfer is confirmed manually |

Invalid `inviteCode` does **not** block registration — user simply won't be bound.

---

## Deposit Integrations

Backend proxies for third-party deposit channels. A channel that is not
enabled returns 503 `NOT_CONFIGURED`.

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
| ReleaseLockedFunds | `action: "releaseLockedFunds"`, `userId`, `itemIdsHash: bytes32`, `acknowledgeRisk: bool`, `fundsChainId`, `nonce` |
| TwitterLink | `action: "twitterLink"`, `userId`, `nonce` |
| TwitterUnlink | `action: "twitterUnlink"`, `userId`, `nonce` |

Use `ProductInvestConfigured` whenever `POST /products/:productKey/invest`
contains `overrides.worldCup`; the hash must come from preview or definition
and match the normalized configuration. Sign `autoCompound: false`
explicitly when the checkbox is unchecked. Custom-index **self-managed**
invest uses the same type. Custom-index **Agent initialization** does not —
sign `IndexAutomationInitialization` from the quote payload
(domain `Polyvaults Index Automation` v2).

Withdraw: sign the **final** `token` and `chain` values that will be sent in
the request body (defaults: `USDC.e` / `polygon`).

### Session Token (non-fund writes)

Campaign write endpoints, custom-index compose/instances/automation, locked-funds
reads, and x402 deposits accept `Authorization: Bearer <sessionToken>`. Token is
issued by `POST /auth/connect` (7-day TTL). Revoke via `POST /auth/logout`.
Fund operations (invest, redeem, withdraw, locked-funds **release**) still
require EIP-712 every time.

### CLOB signing notes

- CLOB API key derivation (`ClobAuth`) must be signed by the owner EOA. Do not
  set CLOB L1 `POLY_ADDRESS` to a Deposit Wallet or Safe contract address; it
  causes `Invalid L1 Request headers` / malformed API credentials.
- Deposit Wallet order signing uses `POLY_1271`; Safe users use
  `POLY_GNOSIS_SAFE`.
- Withdraw signatures bind destination, amount, token and chain.

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

## Custom Indexes

All browser and external-agent traffic uses the public API under
`/custom-index/*` with `Authorization: Bearer <sessionToken>`. Use only the
routes documented here.
`GET /products` does not list CUSTOM instances.

### Compose

`POST /custom-index/compose/sessions` then `POST .../sessions/:id/messages`.
Send `Idempotency-Key` on session create.

When x402 payment is required, the first call returns **HTTP 402**
with `PAYMENT-REQUIRED` (x402 v2, `exact`, Polygon USDC). Sign
`TransferWithAuthorization` (EOA) or use
`POST /custom-index/payments/attempts/:attemptId/safe-payload` after a
`CustomIndexX402Payment` EIP-712 authorization (Safe, native USDC, no
Polymarket relayer). Retry with `PAYMENT-SIGNATURE`. When mode is `disabled`,
sessions create without 402.

A settled attempt covers one **delivered** inference. Interrupted runs stay
reusable. `POST .../messages` 402s again for the next completed turn.
`/reconfigure` is not billed. Rotate `Idempotency-Key` after a successful turn.
`userId` is taken from the session; do not rely on a body `userId`.

x402 signing errors (compose fee and hop deposit share them). The server
recovers the EIP-3009 signer locally before calling the facilitator:

| Code | Meaning |
|------|---------|
| `X402_ATTEMPT_EXPIRED` (400) | The attempt's consent deadline passed; fetch a new challenge. The EIP-3009 `validBefore` is stamped separately when the payload is built |
| `X402_SIGNATURE_SIGNER_MISMATCH` (400) | EOA signer ≠ `from`; response includes `payer`, `recoveredSigner` and the EIP-712 `domain` actually used. Non-65-byte signatures (e.g. OKX v1's 103-byte nested format) are passed through unverified |
| `X402_CONTRACT_SIGNATURE_REJECTED` (400) | Contract payer's ERC-1271 `isValidSignature` reverted or returned a non-magic value. Safe compose payers are graded `contract_payer`, not mismatched |
| `X402_EIP7702_DELEGATOR_UNSUPPORTED` (400) | EOA delegated (`0xef0100‖impl`) to an implementation not on the reviewed list (e.g. MetaMask Stateless7702 and OKX smart-wallet implementations are supported) |
| `X402_EIP7702_SIGNATURE_UNSUPPORTED` (400) | Known delegator but uninitialized or its ERC-1271 probe refused this digest |
| `X402_EXTRA_MISMATCH` (400) | Client-echoed `accepted.extra` (token `name`/`version`) differs from the requirements |
| `X402_FACILITATOR_REJECTED` (400) / `X402_FACILITATOR_UNAVAILABLE` (503) | Facilitator 4xx (our request; has `facilitatorReason`) vs connection failure/5xx |
| `X402_NETWORK_NOT_SUPPORTED` | The challenge network must be CAIP-2 (`eip155:137`) |

Allow at least ~600s for the signing window. Safe owner signatures carry `v`
31/32 (`eth_sign`) — do not "normalize" them to 27/28.

#### Compose routes and response semantics

| Route | Purpose |
|-------|---------|
| `POST /custom-index/compose/sessions` | Create session (402-gated when required). Returns `sessionId`, `expiresAt`, `limits`, `paymentAttemptId` when paid |
| `GET .../sessions/:sessionId` | Owner snapshot (turns + reducer state) and `operationInProgress`. When true the proposal is only the last completed version — do not hydrate/unlock save, retry later |
| `POST .../sessions/:sessionId/messages` | Billed inference. JSON, or SSE with `Accept: text/event-stream` (`stage`, `token`, `recommendations`, `proposal`, `done`, `error`; 15s heartbeat comments; client disconnect cancels upstream work) |
| `POST .../sessions/:sessionId/reconfigure` | Deterministic, no-LLM rebuild after an OI change. Body `minOiUsd` / `maxOiUsd` (top-level or in `preferences`) or `resetToLiquidityRecommendation: true` |
| `PUT .../sessions/:sessionId/proposal` | Import a public catalog proposal (re-validated live) for "fork then edit"; returns `proposal`, `proposalHash`, `resolvedThesis`, `candidates` |
| `PATCH .../sessions/:sessionId/proposal` | Manual edit of `items[]` (`marketRef`, `buyDirection`, `targetWeight`); live-validated; may return `guidance` |

`composeStatus`:

- `ok` — new `proposal` + `proposalHash`. The only state that may replace or
  save a proposal.
- `guidance` — not enough valid candidates / validation failed. Carries
  `candidateCount`, `diagnostics` (incl. `guidanceReason`: `selection_review` |
  `liquidity_filter` | `coverage` | `live_validation`), `candidates`,
  `resolvedThesis`. A previously accepted proposal may be returned for
  comparison with `proposalStaleForPreferences: true` — it must **not** be saved
  or invested. Under `selection_review` the OI counts are deliberately omitted;
  do not suggest lowering OI.
- `conversation` — answer/clarification only; keep the existing proposal.

Every response includes the effective `preferences` (OI range). `candidates[]`
is the full discovery pool ordered by `relevanceRank` with `relevanceTier`
(`direct` | `related`), `relevanceScore`, `eligibilityReasons[]` and
`liquidityNotices`. Zero-OI or confirmed-closed markets are never returned.
Spread tiers: `normal`; `wide` stays selectable with non-blocking
`WIDE_SPREAD`; `unusable` (above ~0.35) is blocking `SPREAD_TOO_WIDE`. `liquidityRecommendation`
(`recommendedPreferences.minOiUsd`, `verifiedAnchors`, `reasonCodes`,
`oiStopsUsd`) is an after-the-fact slider readout of the basket, not a gate.

Selection guards: markets whose resolution window elapsed are dropped even if
they still show as open; mutually exclusive sibling legs (numeric buckets,
date ladders, single-winner contests, rank-exclusive or deadline-implied
cross-market pairs) collapse to one; a per-leg direction review keeps only
legs that pay on the thesis direction, and a hallucinated `suggestedName` is
replaced by a deterministic one. The final basket may shrink after that review.

Errors: 429 `COMPOSE_QUOTA_*` (`retryAfterSec`), 409 session busy, 409
`RECONFIGURE_SNAPSHOT_REQUIRED` / `RECONFIGURE_THESIS_REQUIRED` with
`recovery=start_new_session` (create a new session with the retained thesis),
503 LLM unavailable (explicit guidance; never a no-AI proposal).

### Save / fork

`POST /custom-index/instances` is not billed — it binds the compose payment.
In `required` mode send `paymentAttemptId` or `composeSessionId`; otherwise
400 `X402_PAYMENT_REQUIRED`. One payment binds to at most one series. Fork
(`POST /custom-index/catalog/:slug/fork`, active sources only) needs its own
payment. Body: `proposalHash` + `proposal` (items with `marketRef`,
`buyDirection`, `targetWeight`), optional `displayName` (≤ 80 chars), `slug`,
`status` (`draft|active`, default `active`), `visibility` (`private` default |
`public`; only active can be public), `constraints` (`minOiUsd`, `maxOiUsd`
or `null`). Response: `id`, `productKey`, `routeSlug`, `strategyVersion`,
`strategyHash`, `status`.

Save does not re-check spread/OI/price; those are checked at preview/invest.
409 `CUSTOM_INDEX_SLUG_TAKEN` / `CUSTOM_INDEX_PUBLIC_SLUG_TAKEN` (message
contains `slug already exists`; retry `-2`, `-3`).

`PATCH /custom-index/instances/:id` changes `displayName`, `status`
(`archived` | `active`) and `visibility` only — never the basket. Archiving is
refused (400) while FILLED positions are unredeemed. `expired`/`superseded`
cannot become `active`, and `expired` cannot be archived.

### Revisions (update an index)

Series identity: `seriesId` = first revision's `id`; each update is an
immutable revision (`revisionNumber`, `isCurrentRevision`, `supersedesId`).
r1 keeps `custom-index:<userId>:<slug>`; r≥2 is `...:r<N>`. Slug is reused.
Owner detail/performance/health/preview/invest accept any revision id and
resolve to the current one. Updates never trade or rebalance existing
positions; they stay on their revision's `productKey`.

1. `POST /custom-index/instances/:id/update-session` (session + geo; consumes
   a compose session-create quota). Server builds the seed from the DB:
   terminal legs dropped, weights renormalized, OI constraints inherited.
   Returns `sessionId`, `baseRevisionId`, `seriesId`, `revisionNumber`,
   `thesis`, `updateContext`, `preferences`, `seedProposal`,
   `seedProposalHash`, `keptCount`, `dropped[]`, `seedBelowMinimum`, `seeded`.
   Reuse `preferences` verbatim (an unbounded range is `maxOiUsd: null`).
   An all-ended basket still seeds (empty); the thesis is pinned, so a
   different thesis needs a new index. A lifecycle-check failure fails open
   (seed the whole basket); an `archived` row without `expiredAt` only opens an
   update when every leg is *affirmatively* ended, else restore first.
2. Compose/refine in that session (`/messages`, `/reconfigure`, PATCH).
3. `POST /custom-index/instances/:id/revisions` — body `composeSessionId`,
   `baseRevisionId`, `proposalHash`, optional `displayName`. `proposal` /
   `thesis` / `constraints` in the body are ignored; the server reads them from
   the session. Not billed, no new mint. Draft stays draft; `expired` returns
   to `active`; public stays public under the same slug.
   - 409 `COMPOSE_SESSION_BUSY` — agent still refreshing.
   - 409 `CUSTOM_INDEX_REVISION_CONFLICT` — `baseRevisionId` is no longer
     current, or another update/archive/publish/rename landed first. No merge.
4. `GET /custom-index/instances/:id/revisions[?limit=&before=]` — newest first:
   `revisionNumber`, `productKey`, `status`, `isCurrentRevision`,
   `supersedesId`, `itemCount`, `createdAt`, `expiredAt`, own `paperPerformance`.

### Owner reads

`GET /custom-index/instances` (`?hasPosition=true` optional). Each row:
`automationSummary` (`requested`, `status` `enabled|paused|disabling|disabled`,
`pauseReason`, `seriesCash`, `nextRunAt`, `initialization`), `hasPosition`,
`positionAction` (`residualOnly`, `hiddenByDefault`, `hasPendingActivity`).
`hasPosition` is open FILLED size / deployed principal / position value —
Agent cash alone does not qualify. `hiddenByDefault` is presentation only.

Optional list query: `status`, `includeHealth=true`. List returns only the
current revision of each series but aggregates real positions across **all**
its revision keys (`hasDeposits`, `investedAmount`, `currentValue`, `pnl`).

`GET /custom-index/instances/:id` canonicalizes to the current revision.
Authenticated responses (list, detail, portfolio, performance, revisions,
health): `Cache-Control: private, no-store`.

Lifecycle fields on list/detail: `expired`, `expiredAt`, `settlementStatus`
(`pending` | `settled`), `lifecycle { checkedAt, fresh, terminalCount,
openCount, unknownCount }`, `needsUpdate` (some, not all, legs ended),
`canInvest`, `canUpdate`, and `components[]` with per-leg `lifecycle`
(`open` | `ended_pending_settlement` | `resolved` | `unknown`),
`configuredWeight` (never rewritten) and `effectiveWeight` (renormalized over
live legs). A stale/never-evaluated snapshot is history only: `lifecycle` is
`null` and `effectiveWeight` falls back to the configured weight.

#### Lifecycle and expiry

The lifecycle is re-evaluated every ~15 minutes over the whole saved basket of
current draft/active/expired/archived revisions. A leg is `resolved` when the
market is closed and settled, `ended_pending_settlement` when closed with
resolution still pending, `unknown` when no data came back for that exact
market. Only when **every** leg is affirmatively closed and none is
`unknown` does the series become `expired` (never reversible; `settlementStatus`
advances `pending` → `settled`). Fetch failures write nothing (fail-open).
Expired: preview/invest rejected, removed from the public list, but the slug
page, curve, valuation, settlement and redeem keep working; fork requires
`active`.

#### GET /custom-index/instances/:id/performance?range=24h|7d|30d|all

Simulated since-creation curve — **not** user P&L:

```json
{
  "mode": "paper_since_creation",
  "unit": "fraction",
  "baselineCoverage": 1,
  "excludedLegs": [],
  "series": [{ "timestamp": "2026-07-27T09:00:00.000Z", "indexValue": 103.42, "totalReturn": 0.0342 }]
}
```

`totalReturn` is a fraction (`0.0342` = +3.42%). Entry prices freeze at the
first trusted quote and use the **ask**; marks use the **bid** (a one-sided
book's midpoint is not tradable). Legs `closed` at baseline
(`closed_pending_resolution` / `closed_resolved`) or with spread > 0.35
(`unusable_spread`) are excluded and the rest renormalized; `baselineCoverage`
< 1 means legs were dropped. A not-yet-closed leg without a quote keeps
`awaiting_initial_quote` (retried hourly). Below 60% surviving weight, or after
48h of waiting, the state becomes terminal `baseline_unavailable` — show "no
simulated return". After a baseline exists, missing quotes make the summary
`stale` (render the last full mark; never fake 0%). `series[].timestamp` is the
mark's observation time. Hourly points are kept for 31 days; daily points and
the 0% baseline are kept permanently. `paperPerformance`
(`status`, `startedAt`, `latestAt`, `indexValue`, `totalReturn`) is the summary
on list/detail; if absent, do not render "preparing first mark" as data.
Public equivalent: `GET /custom-index/catalog/:slug/performance`.

#### GET /custom-index/instances/:id/portfolio?range=24h|7d|30d|all

Authoritative **real** position read for a series (invalid `range` → 400).

```json
{
  "seriesId": "ci_r1",
  "currentRevisionId": "ci_r2",
  "currentRevisionNumber": 2,
  "hasDeposits": true,
  "timeRange": "all",
  "portfolio": {
    "nav": 12.0, "deployedPrincipal": 10.0, "positionValue": 12.0,
    "unrealizedPnl": 2.0, "realizedPnl": 0, "pnl": 2.0, "totalReturn": 0.2,
    "totalClobFees": 0, "returnChart": [],
    "historyCoverage": { "complete": true, "startsAt": "2026-07-01", "reason": null }
  },
  "revisions": [{ "id": "ci_r2", "revisionNumber": 2, "productKey": "custom-index:usr_abc:macro-v1:r2",
    "isCurrentRevision": true, "hasDeposits": false, "portfolio": {}, "paperPerformance": {} }]
}
```

Top-level `portfolio` is the whole series (principal and P&L re-reduced, never
summed/averaged percentages); `revisions[].portfolio` is each exact productKey;
`revisions[].paperPerformance` is that revision's own simulated curve.
`hasDeposits` means a COMPLETED/PARTIAL deposit ever existed — independent of
current value. `historyCoverage.complete=false` (reason
`series_history_started_after_multiple_revision_activity`) means the curve is
truncated: do not present it as the full All-Time history. The series NAV is
persisted hourly under a synthetic `custom-index-series:<seriesId>` key.

Self-managed invest: `POST .../instances/:id/invest` with
`ProductInvestConfigured`. Preview/health expose per-leg `spread`; preview uses
**mid** prices while FAK orders fill at the **ask**, so a wide spread must
be disclosed before signing (`WIDE_SPREAD`). Redeem: `POST .../redeem` — body
`productKey` may be any revision of the series (older positions live on older
revisions); stops the **series** Agent first; HTTP 409
`AUTOMATION_DISABLE_DRAINING` means wait (no sale yet).

### Public catalog

No auth, owner-safe (no owner, `productKey` or internal ids).

| Endpoint | Notes |
|----------|-------|
| `GET /custom-index/catalog?cursor=&limit=&search=` | Public + active + not expired, ranked by paper return (no-baseline last). `search` = single fuzzy page, `nextCursor` null, uncached. Cached 60s otherwise (`Cache-Control: public, max-age=60, stale-while-revalidate=300`) |
| `GET /custom-index/catalog/:slug` | Components (`marketRef`, slug, direction, weight, optional event `question`). Also serves `expired` indexes (read-only) |
| `GET /custom-index/catalog/:slug/performance?range=` | Aggregate paper curve + methodology only |
| `GET /custom-index/recipes?limit=&offset=` | Anonymous rotating hot-market recipe cards (`label`, `prompt`, `theme`); response has `poolSize` and the effective `offset` for Shuffle. Near-settled price-target strikes are excluded |
| `POST /custom-index/catalog/:slug/fork` | Session + geo; own payment when x402 required |

Discovery filters fail open: an index never evaluated, or without a paper
portfolio, stays visible; only a fresh all-terminal lifecycle snapshot or a
confirmed zero open legs hides it from the list (deep links still work).

### Agent automatic management

Pricing (unauthenticated): `GET /custom-index/automation/public-conf`
(use exactly this path; a `.../config` path is not served).

First buy (do **not** call `/invest`):

1. `POST /custom-index/instances/:id/automation/initialization/quote`
   body `{ "amount": "10", "slippageBps": 200 }`
2. Sign returned typed data: domain
   `{ name: "Polyvaults Index Automation", version: "2", chainId: 137 }`,
   primaryType `IndexAutomationInitialization`.
3. `POST .../automation/initialization` with quote fields + signature.
4. Poll `GET .../automation` (`initialization.state`, `funding.pendingReason`,
   receipts). Do not auto-retry ambiguous submits.

$10 input floor is hard; full-basket coverage is a recommendation. Surviving
legs are reweighted over the affordable subset. Authenticated FAK zero-fill
ends init and releases unspent series cash.

Later: `POST .../automation` `{ "action": "enable"|"pause"|"resume"|"disable" }`.
Disable keeps positions; cash releases after drain. Shared fee reserve
releases only when every mandate is disabled. `GET .../automation/runs` for
receipts.

### ERC-8004

After save, the platform mints an Identity NFT and transfers it to the user EOA.
Read `agentRegistration` or `GET /custom-index/agents/:seriesId/registration.json`.
Retry: `POST /custom-index/instances/:id/agent/retry`. Failure does not
roll back the index. `registration.json` advertises the `Polyvaults Custom
Index` service as `{app base}/custom-index/{seriesId}` (the front-end page, not
the compose API); private/draft series expose no thesis, basket or PII.
Revisions reuse the original agent identity (no new mint).

### Agent initialization quote failures

Quote errors occur **before** any wallet prompt. 400
`AUTOMATION_INITIAL_MARKET_UNAVAILABLE` names the market whose CLOB `/book` is
missing or mismatched (it may still be listed as active). 503
`AUTOMATION_INITIAL_BALANCE_UNAVAILABLE` / `..._MARKET_DATA_UNAVAILABLE` /
`..._FUNDING_PLAN_FAILED` / `..._ORDER_PLAN_FAILED` are retryable read/plan
failures. Initial BUYs are market-buys with a 2-decimal USDC amount; a fill
better than the limit returns more shares for the same cash (cash-bounded, not
share-bounded). Status exposes `initialization.funding.pendingReason` (+
`pendingDetail`) when funding evidence is awaited.

### POST /x402/deposits

Optional hop deposit (default off). Session + geo. Body `amount` (USDC
decimal). HTTP 402 until EIP-3009 settlement; the platform then forwards
native USDC to the user Safe in a second step.
List: `GET /x402/deposits`. Status `forward_review` = payment received,
credit needs manual review — show Needs review + attempt id; do not
re-charge; keep polling. Paying from the credited Safe returns
`X402_DEPOSIT_SELF_TRANSFER`.

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
| IndexAllocationStatus | PENDING, PLACED, FILLED, FAILED, SOLD, REDEEMED, NEEDS_REVIEW |
| TradeStatus | PENDING, MATCHED, MINED, CONFIRMED, FAILED |
