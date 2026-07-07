---
name: narrative-index
description: >-
  Operates a custodial multi-asset directional index investment platform on
  Polymarket. Supports asset-direction indices, NarrativeBasket products such
  as TACO, World Cup 2026 bracket products. Provides wallet management,
  portfolio monitoring, performance returns, chart data, withdrawals, referral
  rewards, World Cup auto-roll chain management, remote signing policy, and
  multi-chain wallet EIP-712 V2 signatures. Use when the user mentions
  investing, prediction markets, Polymarket, portfolio performance, USDC/pUSD
  balance, deposits, withdrawals, productKey, TACO, World Cup brackets, CLOB
  auth, signing service, or strike price allocation.
---

# Narrative Index — Agent Skill

## Platform Overview

Polyvaults is a custodial Polymarket index platform. It supports legacy
asset-direction indices (`BTC/OIL/ETH` Bullish/Bearish), managed baskets such as
TACO, and World Cup 2026 bracket products. Each user gets a segregated
Polymarket Deposit Wallet or legacy Safe managed by the platform. Trades are
executed as gasless FAK market orders via Polymarket CLOB V2 with builder
attribution.

- **Base URL**: `https://api.polyvaults.ai`
- **Network**: Polygon
- **Trading collateral**: pUSD. Deposits can arrive as native USDC or USDC.e;
  investment preparation wraps/converts them into pUSD when needed.
- **Auth model**: All requests identify the user by `userId` (UUID), obtained
  through `connect_wallet` after an EIP-191 challenge signature.
- **User signature auth**: Fund-moving write endpoints require EIP-712 V2
  typed data (`version: "2"`, `domain.chainId` = wallet chain, `fundsChainId: 137`).
  `autoCompound` is part of the invest signature. World Cup `stop-rolling` and
  `retry-roll` also require signatures scoped by `rootDepositId`.
- **Session token**: `POST /auth/connect` returns a 7-day `sessionToken` for
  Campaign write endpoints (non-fund actions). Fund ops still sign every time.
- **Remote signing**: The main API never holds KMS decrypt permission or
  plaintext owner keys. A separate signing-service signs EIP-191/EIP-712
  payloads over an internal authenticated channel, with optional mTLS, rate
  limits, and destination/signature policy enforcement.
- **Geo-restriction**: New-position endpoints are blocked in restricted
  countries/regions. Close-only regions may redeem/withdraw but cannot open new
  positions. Read-only endpoints are unaffected.
- **Rate limiting**: Global rate limits apply — 10 requests/second and
  100 requests/minute per IP.
- **CLOB auth**: CLOB API keys are derived with the owner EOA. For Deposit
  Wallets, the CLOB order itself uses `POLY_1271`; contract wallets are not
  used as the L1 `POLY_ADDRESS` for `/auth/api-key`.

### Supported Assets

| Symbol | Name | Category | Status | Price Source |
|--------|------|----------|--------|-------------|
| BTC | Bitcoin | CRYPTO | active | Binance |
| ETH | Ethereum | CRYPTO | active | Binance |
| SOL | Solana | CRYPTO | coming_soon | Binance |
| OIL | Crude Oil | ENERGY | active | Yahoo CL=F, MEXC fallback |
| GOLD | Gold | METALS | coming_soon | Pyth Network |
| SILVER | Silver | METALS | coming_soon | Pyth Network |

Assets with `active` status have live Polymarket markets and can be traded.
Assets with `coming_soon` are registered but not yet available for investment.

For full endpoint schemas see [references/api-reference.md](references/api-reference.md).
For strategy mechanics see [references/strategy-guide.md](references/strategy-guide.md).

---

## Agent User Journey (end-to-end)

| Phase | Goal | Primary tools |
|-------|------|---------------|
| 1. Onboard | Create user + custodial wallet | `connect_wallet` |
| 2. Fund | Get deposit address, confirm balance | `get_deposit_address`, `get_wallet_balance` |
| 3. Discover | Pick asset/product, check tradability | `get_assets`, `get_market_status`, `list_products`, `get_product_health` |
| 4. Preview | Show allocation before signing | `preview_product` or `preview_index` |
| 5. Invest | EIP-712 sign + execute | `invest_product` or `invest_index` |
| 6. Monitor | Dashboard NAV/PnL/holdings | `get_portfolio`, `get_product_portfolios`, `get_product_portfolio` |
| 7. Exit / cash out | Early redeem or withdraw | `redeem_product` / `early_redeem`, `withdraw` |

**Deposits** are on-chain transfers to `depositAddress` on Polygon (USDC or
USDC.e). There is no backend "deposit" write API — poll `get_wallet_balance`
until funds arrive. Cross-chain deposits use Fun.xyz / Relay.link proxies
(documented in api-reference).

**First connect** may return `isDeployed: false` or `isApproved: false` while
the platform wallet is still provisioning. Retry `connect_wallet` after a few
seconds, or call `get_wallet_info` until both are true before investing.

---

## Available Tools

### 1. connect_wallet

Register or log in a user. First call `GET /auth/challenge?address=0x...`,
have the wallet `personal_sign` the returned `challenge`, then submit the
signature. First-time connect auto-creates a user record and wallet.

```
GET /auth/challenge?address=0x...
POST /auth/connect
Body: { "walletAddress": "0x...", "signature": "0x...", "challenge": "...", "inviteCode": "optional", "chainId": 8453 }
```

Optional `chainId` — wallet's current EVM chain (e.g. Base `8453`). Recommended
for Smart Wallet users to speed up signature verification.

Returns `userId`, `safeAddress`, `depositAddress`, `isDeployed`, `isApproved`,
`isNewUser`, `twitterHandle`, `sessionToken`. Optional `inviteCode` binds
referral at first registration only. Store `sessionToken` for Campaign writes
(`Authorization: Bearer <sessionToken>`).

If `isDeployed` or `isApproved` is false, wallet provisioning is still in
progress — retry connect or poll `get_wallet_info` before invest.

```bash
curl 'https://api.polyvaults.ai/auth/challenge?address=0x1234...abcd'
curl -X POST https://api.polyvaults.ai/auth/connect \
  -H 'Content-Type: application/json' \
  -d '{"walletAddress":"0x1234...abcd","signature":"0x...","challenge":"Sign this message..."}'
```

---

### 2a. get_wallet_info

Full wallet metadata including deployment/approval status and wallet type.

```
GET /wallets/:userId
```

Returns `ownerAddress`, `safeAddress`, `walletType` (`DEPOSIT_WALLET` or
`SAFE`), `isDeployed`, `isApproved`. Use after first connect to confirm the
wallet is ready for trading.

---

### 2. get_wallet_balance

Query the user's wallet balance, including USDC.e, native USDC, pUSD, and any
application-level locked amount reserved for World Cup auto-roll.

```
GET /wallets/:userId/balance
```

Returns `formattedBalance` (USDC.e), `formattedNativeBalance` (native USDC),
`formattedPusdBalance`, `totalBalance`, `lockedBalance`, and
`withdrawableBalance`. Use `withdrawableBalance` for max withdraw/new invest.

```bash
curl https://api.polyvaults.ai/wallets/{userId}/balance
```

---

### 3. get_deposit_address

Get the wallet address where the user should send USDC or USDC.e on Polygon.
Both are accepted; native USDC and USDC.e are prepared into pUSD when investing.

```
GET /wallets/:userId/deposit-address
```

Returns `address`, `network` ("Polygon"), `token` ("USDC / USDC.e").

```bash
curl https://api.polyvaults.ai/wallets/{userId}/deposit-address
```

---

### 4. preview_index

Legacy asset-direction preview. Preview how funds would be allocated across
strikes before investing. Only strikes meeting the $1 minimum are returned.
New product surfaces should prefer `preview_product`.

```
POST /index/preview
Body: { "indexType": "BULLISH"|"BEARISH", "amount": 100, "userId": "...", "asset": "BTC" }
```

Optional fields:
- `asset` — asset symbol (default: `BTC`). Active: BTC, ETH, OIL.
- `eventSlug` — override the default current-month event
- `userId` — when provided, the backend estimates any collateral preparation
  fee and computes allocations using the effective amount.

Returns `asset`, `allocations[]`, `droppedStrikes`, `resolvedStrikes`,
`minimumDepositRequired`, `effectiveAmount`, `swapFee`.

```bash
curl -X POST https://api.polyvaults.ai/index/preview \
  -H 'Content-Type: application/json' \
  -d '{"indexType":"BULLISH","amount":100,"userId":"abc-123","asset":"OIL"}'
```

---

### 5. invest_index

Legacy asset-direction investment. Places FAK (Fill-and-Kill) market orders for
each qualified strike from the user's wallet balance. Orders fill immediately
against available liquidity; any unfilled portion is cancelled.

If the user's pUSD balance is insufficient, the platform prepares collateral by
wrapping USDC.e and, if needed, swapping native USDC to USDC.e before wrapping.

> Requires EIP-712 `Invest` signature. Include `signatureChainId` and
> `fundsChainId: 137`. See
> [api-reference.md](references/api-reference.md#signature-authentication).

```
POST /index/invest
Body: { "userId": "...", "indexType": "BULLISH"|"BEARISH", "amount": 100, "asset": "BTC", "slippage": 0.02, "signature": "0x...", "nonce": 1740643200000 }
```

Optional: `asset` (default: BTC), `eventSlug`, `slippage` (0.001–0.1, default 0.02).

Returns `depositId`, `asset`, `allocations[]` (with `orderId`, `orderStatus`),
`hasPlacedOrders`, `overallStatus` (SUCCESS / PARTIAL / FAILED).

```bash
curl -X POST https://api.polyvaults.ai/index/invest \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","indexType":"BULLISH","amount":100,"asset":"BTC","signature":"0x...","nonce":1740643200000}'
```

---

### 6. get_positions

List all index deposits and their per-strike allocations for a user.

```
GET /index/positions/:userId
```

Returns an array of deposits, each with `asset`, `allocations[]`, `status`
(PENDING / EXECUTING / COMPLETED / PARTIAL / FAILED).

```bash
curl https://api.polyvaults.ai/index/positions/{userId}
```

---

### 7. get_portfolio

Dashboard metrics: NAV, deployed principal, available balance, PnL, total
return, and a return chart series. Supports filtering by asset.

```
GET /portfolio?userId=...&timeRange=24h&asset=BTC
```

- `asset` (optional): filter to a specific asset. When provided, also returns
  per-direction breakdown (`bullish`, `bearish`).
- `timeRange`: 24h | 7d | 30d | all

Returns `nav`, `deployedPrincipal`, `positionValue`, `availableBalance`,
`unrealizedPnl`, `realizedPnl`, `pnl`, `untrackedPnl`, `totalReturn`,
`returnChart[]`.

```bash
curl 'https://api.polyvaults.ai/portfolio?userId=abc-123&timeRange=7d'
curl 'https://api.polyvaults.ai/portfolio?userId=abc-123&asset=OIL'
```

---

### 8. get_returns

Monthly daily-return data for an asset's spot price, Bullish Index, Bearish
Index, and outperformance.

```
GET /performance/returns?month=YYYY-MM&asset=BTC
```

- `asset` (optional): defaults to BTC. Available: BTC, OIL, GOLD, etc.
- `live` (optional boolean): when true, calculate live values where supported.
- Limited to the last 6 months.

Returns an array of `{ date, asset, btcPrice, btcReturn, bullishReturn,
bearishReturn, bullishOutperformance, bearishOutperformance }`.

```bash
curl 'https://api.polyvaults.ai/performance/returns?month=2026-03&asset=OIL'
```

---

### 9. withdraw

Withdraw from the user's platform wallet. Supports Polygon local transfers and
cross-chain withdrawals via the Polymarket Bridge (Ethereum, Arbitrum, Base,
Optimism, BSC, Solana).

> Requires EIP-712 `Withdraw` signature. Sign `token` and `chain` in the typed
> data. Include `signatureChainId` and `fundsChainId: 137`. See
> [api-reference.md](references/api-reference.md#signature-authentication).

```
POST /wallets/withdraw
Body: { "userId": "...", "toAddress": "0x...", "amount": 100, "chain": "ethereum", "token": "pUSD", "slippage": 0.02, "previewEstimatedOutput": 99.9, "signature": "0x...", "nonce": 1740643200000 }
```

- `token` (optional): `"USDC"`, `"USDC.e"`, or `"pUSD"`. Polygon withdrawals only. Defaults depend on available balance.
- `chain` (optional): Target chain. Defaults to `"polygon"`. Supported: `polygon`, `ethereum`, `arbitrum`, `base`, `optimism`, `bsc`, `solana`.
- `slippage` and `previewEstimatedOutput` are cross-chain quote validation fields.

Returns `transactionHash`, `status` ("SUBMITTED" for Polygon, "BRIDGING" for cross-chain).
Cross-chain responses also include `chain` and `bridgeDepositAddress` for status tracking.

```bash
# Polygon withdrawal
curl -X POST https://api.polyvaults.ai/wallets/withdraw \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","toAddress":"0xdead...","amount":100,"token":"USDC","signature":"0x...","nonce":1740643200000}'

# Cross-chain withdrawal to Ethereum
curl -X POST https://api.polyvaults.ai/wallets/withdraw \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","toAddress":"0xdead...","amount":100,"chain":"ethereum","signature":"0x...","nonce":1740643200000}'
```

### 9a. withdraw_quote

Preview cross-chain withdrawal fees and estimated time.

```
POST /wallets/withdraw-quote
Body: { "userId": "...", "amount": 100, "chain": "ethereum", "recipientAddress": "0x..." }
```

Returns `quoteId`, `estimatedOutput`, `fees`, `estimatedTimeMs`,
`minWithdrawal`, and a price disclaimer.

### 9b. withdraw_status

Track cross-chain withdrawal progress using the `bridgeDepositAddress`.

```
GET /wallets/withdraw-status/:address
```

Status flow: `DEPOSIT_DETECTED` → `PROCESSING` → `ORIGIN_TX_CONFIRMED` → `SUBMITTED` → `COMPLETED`.

### 9c. supported_chains

List all supported withdrawal chains with minimum amounts.

```
GET /wallets/supported-chains
```

Query withdrawal fee beforehand with `GET /wallets/withdraw-fee`.

---

### 10. get_chart

Hourly price data with strike price lines for any supported asset.

```
GET /chart/strikes?indexType=BULLISH&asset=BTC
```

Optional query params: `asset` (default: BTC), `eventSlug`, `from`, `to` (ISO 8601).

Legacy alias: `GET /chart/btc-strikes` (same behavior, defaults to BTC).

Returns `asset`, `priceData[]`, `strikePrices[]`, `nextStrike`, `resolved`,
`eventTitle`.

```bash
curl 'https://api.polyvaults.ai/chart/strikes?indexType=BULLISH&asset=OIL'
```

---

### 11. early_redeem

Market-sell all active positions for a given direction (BULLISH or BEARISH).
A 5% fee is charged on positive profit (2.5% platform + 2.5% referrer when
the user has a referrer). Partial closes can return `RETRYING` and be retried
by the backend.

Resolved positions are automatically redeemed by a cron job every 15 minutes —
this endpoint is only for **early** pre-settlement exits.

> Requires EIP-712 `Redeem` signature. Include `signatureChainId` and
> `fundsChainId: 137`. See
> [api-reference.md](references/api-reference.md#signature-authentication).

```
POST /index/redeem
Body: { "userId": "...", "direction": "BULLISH"|"BEARISH", "asset": "BTC", "slippage": 0.02, "signature": "0x...", "nonce": 1740643200000 }
```

`asset` is required by signature validation. `slippage` is optional.

Returns `sold`, `totalReceived`, `totalCost`, `profit`, `fee`,
`closeStatus`, `results[]`.

```bash
curl -X POST https://api.polyvaults.ai/index/redeem \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","direction":"BULLISH","asset":"BTC","signature":"0x...","nonce":1740643200000}'
```

---

### 12. get_portfolio_breakdown

Get separate metrics for BULLISH and BEARISH directions. Supports per-asset
filtering.

```
GET /portfolio/breakdown?userId=...&asset=BTC
```

- `asset` (optional): when provided, returns breakdown for that asset only.
  When omitted, returns overall breakdown plus `assets[]` array with per-asset
  details.

Returns `bullish` and `bearish`, each with `deployedPrincipal`,
`positionValue`, `unrealizedPnl`, `realizedPnl`.

```bash
curl 'https://api.polyvaults.ai/portfolio/breakdown?userId=abc-123&asset=OIL'
```

---

### 13. get_market_status

Check the current month's prediction market availability for one or all assets.

```
GET /market/status?asset=BTC
```

- `asset` (optional): when provided, returns status for that asset only.
  When omitted, returns an array of all registered assets.

Returns `available`, `asset`, `status` (`active` / `pending_liquidity` /
`coming_soon`), `month`, `year`, `slug`, `title`, `marketsCount`,
`totalOpenInterest`.

```bash
curl 'https://api.polyvaults.ai/market/status'
curl 'https://api.polyvaults.ai/market/status?asset=OIL'
```

---

### 14. get_assets

List all registered assets with their current status.

```
GET /assets
```

Returns `{ assets: [{ symbol, name, category, status }] }`.

```bash
curl https://api.polyvaults.ai/assets
```

---

### 15. list_products

List all registered investment products (INDEX and MANAGED families).

```
GET /products
```

Current families include `btc-bullish`, `narrative-basket:taco-v1`,
`worldcup-2026:*`, etc.

```bash
curl https://api.polyvaults.ai/products
```

---

### 16. get_product_definition

Read product metadata. MANAGED products may include full basket definitions.
For World Cup products, prefer `get_worldcup_catalog` for teams and presets.

```
GET /products/:productKey/definition
```

```bash
curl 'https://api.polyvaults.ai/products/narrative-basket%3Ataco-v1/definition'
```

---

### 17. get_product_health

Check whether a product is currently tradable. INDEX products return a simple
`tradable` flag; MANAGED products return per-strike health, eligible counts,
OI, buy prices, and `dropReason`.

```
GET /products/:productKey/health
POST /products/:productKey/health/preview
```

Use `POST /health/preview` with `overrides.worldCup.teamRefs` for custom World
Cup baskets.

---

### 18. preview_product

Current recommended preview endpoint for INDEX and MANAGED products.

```
POST /products/:productKey/preview
Body: { "amount": 100, "userId": "...", "overrides": { ... } }
```

For World Cup custom baskets, pass `overrides.worldCup.teamRefs`. The response
can include `normalizedWorldCupConfig` and `strategyHash`; keep these for
`invest_product`.

**Preview fields agents should surface:**

| Field | Use |
|-------|-----|
| `items[]` / `allocations[]` | Per-market/strike allocation, weight, buy price |
| `droppedStrikes[]` | Markets pruned (OI, min order, price bounds) |
| `effectiveAmount` / `swapFee` | Net investable amount after collateral prep |
| `minimumDepositRequired` | Min amount to keep all eligible legs |
| `strategyHash` | Required for World Cup custom basket invest |
| `eventTitle` / `eventSlug` | Context for INDEX products |

Call `get_product_health` first when you need tradability / drop reasons before
preview.

---

### 19. invest_product

Current recommended investment endpoint. Creates a deposit, prepares pUSD, and
places FAK orders. There is no per-user or platform active-position cap; the
business minimum remains $10.

```
POST /products/:productKey/invest
Body: { "userId": "...", "productKey": "...", "amount": 100, "slippage": 0.02, "autoCompound": false, "strategyHash": "0x...", "overrides": { ... }, "signature": "0x...", "nonce": 1740643200000 }
```

Requires EIP-712 `ProductInvest` signature. Use `ProductInvestConfigured` when
the request contains `overrides.worldCup` and include the preview `strategyHash`.
`autoCompound` **must be signed** (`false` when disabled). Include
`signatureChainId` and `fundsChainId: 137`.

---

### 20. redeem_product

Market-sell all FILLED positions for a productKey. This is the preferred early
exit endpoint for TACO, World Cup, and new INDEX product pages.

```
POST /products/:productKey/redeem
Body: { "userId": "...", "productKey": "...", "slippage": 0.02, "signature": "0x...", "nonce": 1740643200000 }
```

Requires EIP-712 `ProductRedeem`. Returns the same `closeStatus` and per-token
result structure as `early_redeem`.

---

### 21. get_product_portfolios

List product-level holdings for a user. Use `family=worldcup-2026` for the
World Cup portfolio list.

```
GET /portfolio/products?userId=...&family=worldcup-2026
```

---

### 22. get_product_portfolio

Get a single product's NAV, PnL, chart, and product-specific grouping. MANAGED
products return `clusters[]`; World Cup products also return `teams[]`.

```
GET /portfolio/products/:productKey?userId=...&timeRange=all
```

---

### 23. stop_rolling

Stop a **specific** World Cup auto-roll chain and release application-level
locked cash. Requires EIP-712 `ProductStopRolling` with `rootDepositId`.
Only use when `rollRetry.canStopRolling=true` from `get_worldcup_positions`.

```
POST /products/:productKey/stop-rolling
Body: { "userId": "...", "productKey": "...", "rootDepositId": "...", "signature": "0x...", "nonce": ..., "signatureChainId": 137, "fundsChainId": 137 }
```

---

### 24. retry_roll

Manually retry a failed World Cup auto-roll. Requires EIP-712 `ProductRetryRoll`.
Use when `rollRetry.canRetry=true` from `get_worldcup_positions`.

```
POST /products/:productKey/retry-roll
Body: { "userId": "...", "productKey": "...", "rootDepositId": "...", "signature": "0x...", "nonce": ..., "signatureChainId": 137, "fundsChainId": 137 }
```

---

### 25. get_worldcup_positions

Read-only dashboard of all World Cup chains: current stage, exit stage, locks,
roll retry state, and per-round deposits.

```
GET /products/worldcup-2026/positions?userId=...
```

---

### 26. get_worldcup_dashboard_groups

Variant-level World Cup portfolio (Custom, European Teams, etc.) with aggregated
P&L and return chart.

```
GET /portfolio/worldcup-2026/dashboard-groups?userId=...&timeRange=all
```

---

### 27. get_worldcup_potential_return

World Cup only: simulate multi-round idealized returns through exit stage.

```
POST /products/:productKey/potential-return
Body: { "amount": 100, "userId": "...", "overrides": { "worldCup": { ... } } }
```

---

### 28. get_referral

Get permanent referral code, share URL, and reward dashboard.

```
GET /referral/:userId
```

---

### 29. get_worldcup_catalog

Fetch World Cup 2026 stages, teams, entry presets, exit stages, and the current
schedule gate. Use this before rendering World Cup preset/custom investment UI.

```
GET /products/worldcup-2026/catalog
```

---

## Common Workflows

### Workflow 1 — New User Deposit & Invest

1. **connect_wallet** — challenge → sign → obtain `userId`, `depositAddress`,
   `sessionToken`; retry if `isDeployed`/`isApproved` are false
2. **get_deposit_address** (or use `depositAddress` from connect) — instruct user
   to send USDC or USDC.e on Polygon; no backend deposit API exists
3. **get_wallet_balance** — poll until `withdrawableBalance` reflects the deposit
4. **get_assets** + **get_market_status** — confirm asset is `active`
5. **list_products** or **get_assets** — choose a product or active asset
6. **get_product_health** (optional) — check tradability / drop reasons
7. **preview_product** (preferred) or **preview_index** — show `items[]`,
   `droppedStrikes[]`, `minimumDepositRequired`, `effectiveAmount`
8. Sign EIP-712 (`ProductInvest`, `ProductInvestConfigured`, or legacy `Invest`)
   then **invest_product** or **invest_index**
   - If `PARTIAL`, inspect per-item `orderStatus` / `failReason`
   - If `hasPlacedOrders`, fills are still syncing (cron every 5 min)

### Workflow 2 — Check Investment Performance (Dashboard)

1. **get_portfolio** — aggregate NAV, PnL, `totalReturn`, `returnChart`
   (optionally `?asset=BTC` for per-direction breakdown)
2. **get_portfolio_breakdown** — legacy BULLISH/BEARISH split per asset
3. **get_product_portfolios** — all product holdings (`?family=worldcup-2026`
   for World Cup list)
4. **get_product_portfolio** — single product detail (clusters, teams, chart)
5. **get_worldcup_positions** — World Cup auto-roll chains, locks, retry state
6. **get_returns** — public benchmark daily returns vs asset spot (not user PnL)
7. **get_positions** — legacy per-deposit strike detail if needed
8. **get_chart** — price + strike context for asset-direction products

### Workflow 3 — Withdraw Funds (Polygon)

1. **get_wallet_balance** — confirm `withdrawableBalance`
2. Ask the user which token to withdraw: pUSD, USDC.e, or native USDC
3. Inform the user about the 1% withdrawal fee
   (`GET /wallets/withdraw-fee` for exact rate)
4. **withdraw** — execute with `token` param; return `transactionHash` for
   on-chain tracking

### Workflow 4 — Cross-Chain Withdraw

1. **get_wallet_balance** — confirm available balance
2. **supported_chains** — show the user available chains and minimums
3. **withdraw_quote** — preview fees and estimated arrival time
4. **withdraw** — execute with `chain` param (e.g. `"ethereum"`, `"solana"`)
5. **withdraw_status** — use returned `bridgeDepositAddress` to track progress
   Status flow: DEPOSIT_DETECTED → PROCESSING → COMPLETED

### Workflow 5 — Early Redeem (Pre-Settlement Exit)

1. Use **get_product_portfolio** for product pages, or **get_portfolio_breakdown**
   for legacy asset-direction pages
2. Prefer **redeem_product** with `productKey`; use **early_redeem** for legacy
   `asset + direction`
3. Inform user of `profit`, `fee` (5% of profit if positive), `totalReceived`,
   and `closeStatus`
4. Funds return to the wallet as pUSD/available collateral

### Workflow 6 — Explore Available Assets

1. **get_assets** — see all registered assets and their status
2. **get_market_status** — check which assets have live Polymarket markets
3. **get_chart** — view price data and strike lines for any asset
4. **get_returns** — compare historical performance across assets

### Workflow 7 — TACO / NarrativeBasket Product

1. **list_products** — find `narrative-basket:taco-v1`
2. **get_product_definition** and **get_product_health** — show clusters,
   strike eligibility, OI, prices, and drop reasons
3. **preview_product** — show `items[]`, structured `droppedStrikes[]`, and
   `minimumDepositRequired`
4. Sign `ProductInvest` and call **invest_product**; optional `autoCompound`
   reinvests profitable settlements back into the same TACO product
5. Use **get_product_portfolio** and **redeem_product** for monitoring and exit

### Workflow 8 — World Cup 2026 Bracket Product

1. **get_worldcup_catalog** — fetch entry presets, custom team catalog, exit
   stages, and schedule gate
2. For presets, use `entryPresets[].entryProductKey`; for custom baskets, use
   `custom.entryProductKey` and pass `overrides.worldCup.teamRefs` plus optional
   `exitAfterStageKey`
3. **preview_product** — keep `normalizedWorldCupConfig` and `strategyHash`
4. Sign `ProductInvestConfigured` (include `autoCompound` bool) if
   `overrides.worldCup` is present, then call **invest_product**
5. **get_worldcup_positions** — monitor chains, locks, and `rollRetry` state
6. When `rollRetry.canRetry=true`, offer **retry_roll**; when
   `rollRetry.canStopRolling=true`, offer **stop_rolling** (not redeem)
7. For active FILLED positions, use **redeem_product** for early exit

### Workflow 9 — Referral

1. **connect_wallet** with optional `inviteCode` at first registration
2. **get_referral** — show permanent `referralCode` and `referralUrl`
3. Explain fee split: referred users' 5% profit fee shares 2.5% with referrer
4. `rewards.pending` / `processing` are not yet paid — payout is manual monthly

---

## Key Concepts

- **Multi-asset support**: The platform supports multiple underlying assets
  across crypto (BTC, ETH, SOL), energy (Oil), and metals (Gold, Silver).
  Each asset has independent Polymarket events, strike prices, and price feeds.
- **Asset status**: `active` = live markets, can invest. `pending_liquidity` =
  market exists but insufficient open interest. `coming_soon` = registered but
  no market yet.
- **ProductKey**: Current product APIs identify strategies by `productKey`.
  Use URL-encoded keys in paths (`narrative-basket%3Ataco-v1`) but keep the raw
  key in request bodies and EIP-712 messages.
- **Product kinds**: `INDEX` products are asset + direction indices.
  `MANAGED` products include TACO and World Cup bracket baskets.
- **Deposits**: On-chain only — send USDC/USDC.e to `depositAddress` on Polygon.
  Balance updates via `get_wallet_balance`; no custodial deposit API.
- **Preview before invest**: Always call preview to show allocation, dropped
  markets, and `minimumDepositRequired` before asking the user to sign.
- **IndexType**: `BULLISH` buys YES on "Will [asset] hit $X?" (upside).
  `BEARISH` buys YES on "Will [asset] drop below $X?" (downside).
- **Minimum investment**: Product preview DTOs allow $1, but real invest
  enforces the business minimum, currently $10. Individual strike allocations
  must pass Polymarket minimum order constraints.
- **Order type**: FAK (Fill-and-Kill) market orders with slippage-bounded
  `worstPrice`, executed gaslessly. Unfilled remainder is cancelled.
- **Settlement**: Monthly. Polymarket uses Eastern Time (ET) for market
  creation and settlement; other products settle when their underlying
  Polymarket events resolve.
- **Weight formula**: INDEX products weight by market liquidity and price.
  TACO weights by eligible event OI. World Cup weights by sub-market OI times
  buy price.
- **Wallet**: New users normally use a Polymarket Deposit Wallet
  (`POLY_1271`); legacy users may still use a Safe (`POLY_GNOSIS_SAFE`). Users
  never hold the operational owner key; all owner EOA signing is delegated to
  the isolated signing-service.
- **Signing service policy**: The signer can run in `off`, `audit`, or
  `enforce` policy mode. It allows CLOB auth/order signing only when the
  `policyContext` proves the owner EOA, maker wallet, chain, domain, and
  destination are expected. Raw transaction signing is disabled by default.
- **pUSD collateral**: Trading uses pUSD. The platform accepts USDC.e and
  native USDC deposits and prepares pUSD through wrapping/swap flows. Wallet
  balance includes pUSD, USDC.e, native USDC, locked balance, and withdrawable
  balance.
- **Auto-redemption**: A cron job runs every 15 minutes to scan resolved
  markets. Winning CTF tokens are redeemed to pUSD, and a 5% profit fee is
  collected. Users do not need to manually claim settled positions.
- **EIP-712 V2**: Domain `{ name: "Polyvaults", version: "2", chainId: signatureChainId }`.
  All fund messages include `fundsChainId: 137`. Withdraw binds `token` + `chain`.
  Legacy V1 (`version: "1"`) still accepted during rollout.
- **Auto-roll / autoCompound**: TACO can reinvest profitable settlements back
  into TACO. World Cup products roll into the next stage product when enabled;
  waiting cash is `lockedBalance`. Use **stop_rolling** (with `rootDepositId`)
  to release locks; use **retry_roll** after failed rolls.
- **World Cup redeem fallback**: If no FILLED positions exist but locked cash
  remains in an auto-roll chain, `redeem_product` may fallback to stop-rolling.
- **Referral**: One permanent code per user; 2.5% of referred user profit goes
  to referrer (from the 5% redemption fee). Binding only at first connect.
- **Realized PnL tracking**: Both manual early redemption and auto-settlement
  profits/losses are tracked via `realizedPnl` in portfolio metrics.
- **Price sources**: Crypto assets use Binance spot prices. Oil uses Yahoo
  `CL=F` with MEXC futures fallback. Gold and Silver use Pyth spot feeds.

---

## Error Handling

| HTTP | Message | Action |
|------|---------|--------|
| 400 | "Insufficient balance" | Ask user to deposit more USDC/USDC.e or free locked balance |
| 400 | "Minimum investment is $10" | Use at least $10 for invest |
| 400 | "No active markets" | Current month event not yet live; try later |
| 400 | "Unknown productKey" | Refresh `GET /products` or verify URL encoding |
| 400 | "productKey in URL must match productKey in request body" | Use the same raw productKey in path/body/signature |
| 400 | "Missing or invalid required field: strategyHash" | Preview World Cup custom config first and sign `ProductInvestConfigured` |
| 400 | "Missing or invalid required field: rootDepositId" | Pass `rootDepositId` from `get_worldcup_positions` for stop-rolling/retry-roll |
| 400 | "Only the last 6 months are available" | Adjust `month` param |
| 400 | "Signature expired" | Regenerate nonce (use `Date.now()`) and re-sign |
| 400 | "Nonce already used" | Generate a fresh nonce — each nonce is single-use |
| 401 | "Signature does not match" | User must sign with the wallet used at connect |
| 403 | "GEO_RESTRICTED" | Region restricted; close-only users may redeem/withdraw only |
| 429 | Too Many Requests | Rate limited; wait and retry |
| 500 | Server error | Retry once; if persistent, report to user |

When an invest call returns `overallStatus: "PARTIAL"`, inspect individual
`items[].orderStatus` or legacy `allocations[].orderStatus` and `failReason`.
Do not automatically retry failed strikes unless the user requests it.
