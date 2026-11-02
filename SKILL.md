---
name: narrative-index
description: >-
  Operates a custodial multi-asset directional index investment platform on
  Polymarket. Supports asset-direction Indexes, NarrativeBasket products such
  as TACO, World Cup 2026 bracket products, and Custom hybrid indexes (AI
  compose, immutable revisions, paper-performance curves, public catalog/fork,
  lifecycle/expiry) with optional Agent automatic management. Provides wallet
  management, locked-funds
  explain/self-release, portfolio monitoring, performance returns, chart data,
  withdrawals, referral rewards, World Cup auto-roll, remote signing policy,
  x402 compose/deposit, and multi-chain EIP-712 V2 signatures. Use when the
  user mentions investing, prediction markets, Polymarket, custom index, index
  revision/update, paper performance, marketplace catalog/fork, Agent
  management, locked funds, withdraw blocked, x402, portfolio performance, USDC/pUSD balance,
  deposits, withdrawals, productKey, TACO, World Cup brackets, CLOB auth,
  wallet signing, or strike price allocation.
---

# Narrative Index — Agent Skill

## Platform Overview

Polyvaults is a custodial Polymarket index platform. It supports legacy
asset-direction Indexes (`BTC/OIL/ETH` Bullish/Bearish), managed baskets such as
TACO, World Cup 2026 bracket products, and **Custom hybrid indexes**
(`CUSTOM`, lazy-resolved; not in `GET /products`). Each user gets a segregated
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
- **Session token**: `POST /auth/connect` returns a 7-day `sessionToken`. Use
  it for Campaign writes, custom-index compose/instances/automation, locked
  funds, and x402 deposits (`Authorization: Bearer`). Fund-moving invest,
  redeem, withdraw, and locked-funds **release** still require EIP-712.
- **Custody**: Owner keys are held in isolated infrastructure and are never
  exposed to API clients or agents; fund-moving actions need the user's
  EIP-712 signature.
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
| 5. Invest | EIP-712 sign + execute | `invest_product` / `invest_index`, or custom-index Agent init |
| 6. Monitor | Dashboard NAV/PnL/holdings | `get_portfolio`, `get_product_portfolios`, `list_custom_index_instances`, `get_custom_index_portfolio`, `get_custom_index_performance` |
| 6b. Maintain (custom) | Refresh a basket whose legs ended | `create_custom_index_update_session`, `create_custom_index_revision` |
| 7. Exit / cash out | Early redeem, unlock, or withdraw | `redeem_product` / `redeem_custom_index`, `release_locked_funds`, `withdraw` |

**Deposits** are on-chain transfers to `depositAddress` on Polygon (USDC or
USDC.e). There is no backend "deposit" write API — poll `get_wallet_balance`
until funds arrive. Cross-chain deposits use Fun.xyz / Relay.link proxies
(documented in api-reference).

**Custom indexes** use the public BFF under `/custom-index/*` with
`sessionToken` and only the documented public routes. When x402 is required,
compose sessions return HTTP 402 until a Polygon USDC x402 payment settles;
when it is not required, sessions create without 402. A completed inference
consumes the payment even if you do not save; interrupted runs reuse the
settled attempt. Saving binds that payment — it is not a new charge.
Optional **Agent** management is a separate V2 mandate
(`IndexAutomationInitialization`), not `ProductInvestConfigured`. See Custom
Index tools below.

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
referral at first registration only. Store `sessionToken` for Campaign writes, custom-index compose/instances,
automation, locked-funds reads, and x402 deposits
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
`withdrawableBalance`. Use `withdrawableBalance` for max withdraw/new invest —
never `totalBalance − lockedBalance` or `availableBalance`. It follows the same
per-token rule (`computeSpendable`) as every withdraw/invest check, so it is
never more optimistic than the server.

When an Agent reserve lost its backing (shared fee reserve above the wallet's
native USDC, or unbacked initial funding) the whole wallet is treated as `0`
spendable. The balance response then adds `withdrawBlockedReason`
(`agent_fee_reserve_unbacked` | `agent_funding_unbacked`) and a user-facing
English `withdrawBlockedDetail`. Show that detail instead of a bare "$0.00",
and point the user to `get_locked_funds` / Agent disable. The 400
`Insufficient withdrawable balance` message appends the same detail; keep
matching on its prefix.

```bash
curl https://api.polyvaults.ai/wallets/{userId}/balance
```

---

### 2b. get_locked_funds

Itemized lock sources for the **session user** (not a path `userId`). Same
`totalLocked` as `lockedBalance`. Agent reservations are listed with
`context.agents` (`seriesId`, `displayName`, `status`) and
`managementScope: series | shared`. Do not treat Agent cash as user-releasable.

```
GET /wallets/me/locked-funds
Headers: Authorization: Bearer <sessionToken>
```

`kind`: `round_gap`, `pending_buy`, `series_cash`, `fee_reserve`,
`pending_fee`, `automation_funding`, `unresolved_settlement`. Pausing an
Agent does not release cash; disabling waits for drain. Shared fee reserve
releases only after every mandate is disabled.

---

### 2c. release_locked_funds

Batch-release **eligible** items. Session + EIP-712 `ReleaseLockedFunds` +
geo (close-only allowed). Normalize `itemIds` (trim, drop empty, dedupe,
Unicode sort) then
`itemIdsHash = keccak256(toUtf8Bytes(JSON.stringify(ids)))`.

```
POST /wallets/me/locked-funds/release
Body: { "userId": "...", "itemIds": ["..."], "itemIdsHash": "0x...",
  "acknowledgeRisk": true, "signature": "0x...", "nonce": ...,
  "signatureChainId": 137, "fundsChainId": 137 }
```

Per-item `result`: `released` | `reconciled` | `rejected` | `unchanged`.
`reconciled` means a pending BUY was **proven filled** and booked `FILLED`
(cash spent, lock cleared) — do not describe this as a refund.
`pending_buy` force-unlock is only for the owner's manual BUY that is
`NEEDS_REVIEW`, still locked, and ≥7 days old. Positive fill/position
evidence blocks unlock. Agent-managed kinds are never released here;
disable via `POST /custom-index/instances/:id/automation` `action=disable`.

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

List platform-fixed products (INDEX and MANAGED, including World Cup).
Custom indexes are **not** registered here — use `list_custom_index_instances`.

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
   `asset + direction`; custom indexes use **redeem_custom_index** (stops series
   Agent first — 409 `AUTOMATION_DISABLE_DRAINING` is wait-and-retry)
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

### Workflow 9 — Custom Index (compose, optional Agent)

1. **connect_wallet** — keep `sessionToken`
2. Pricing: `GET /custom-index/automation/public-conf` (use exactly this path;
   a `.../config` path is not served and a 403 there does not mean the
   feature is missing)
3. Compose: `POST /custom-index/compose/sessions` then `/messages`. If HTTP
   402, settle x402 (EOA EIP-3009 or Safe `CustomIndexX402Payment` +
   `.../payments/attempts/:id/safe-payload`). Use only the documented public routes.
   Branch on `composeStatus`: only `ok` carries a new `proposal`/`proposalHash`;
   `guidance` and `conversation` must **not** replace or save a proposal (see
   "Compose response semantics"). Optionally tune liquidity with
   `POST .../reconfigure` (no LLM, not billed). Edit weights/legs with
   `PATCH .../sessions/:id/proposal`.
4. Save: `POST /custom-index/instances` with `proposal` / `proposalHash` and
   `paymentAttemptId` or `composeSessionId` when x402 is required. Optional
   `status` (`draft|active`), `visibility` (`private` default | `public`),
   `slug`, `constraints`. Fork is a new series and a new payment. Revisions
   are not billed. A new index immediately gets a **paper-performance** curve
   (`GET .../performance`) that is separate from real positions.
5. **Self-managed invest**: check `canInvest`, preview (read per-leg `spread` /
   `WIDE_SPREAD` and tell the user previews are mid-market while FAK fills at
   the ask), then `POST .../instances/:id/invest` with
   `ProductInvestConfigured` and pUSD collateral.
6. **Agent-managed first buy**: do **not** call `/invest`. Quote
   `POST .../instances/:id/automation/initialization/quote` (`amount`,
   `slippageBps`), sign `IndexAutomationInitialization` (domain
   `Polyvaults Index Automation` v2, `chainId: 137`), submit
   `POST .../initialization`, then poll `GET .../automation`. $10 floor is
   hard; full-basket coverage is a recommendation — unaffordable legs may be
   dropped and the rest reweighted. Max should leave ~30 evaluation fees plus
   conversion buffer. Do not auto-retry an ambiguous init. Authenticated FAK
   zero-fill ends init and releases unspent series cash.
7. List: `GET /custom-index/instances` (`?hasPosition=true` optional). Read
   `automationSummary`, `hasPosition`, `positionAction`, plus lifecycle fields
   (`needsUpdate`, `canInvest`, `canUpdate`, `expired`, `settlementStatus`).
   Do not hide holdings from NAV because `hiddenByDefault` is true. Real
   P&L comes from `GET .../portfolio`; `paperPerformance` is simulated.
8. Exit: `POST .../instances/:id/redeem` stops the **series** Agent first.
   HTTP 409 `AUTOMATION_DISABLE_DRAINING` means wait and retry — no sale yet.
   Disable-keep-positions uses automation `action=disable` (no redeem).

### Workflow 10 — Locked funds

1. **get_locked_funds** — explain each `kind` and Agent `context`
2. Round-gap / eligible pending BUY: **release_locked_funds** with risk ack
3. Agent cash: navigate to that series' automation; disable (keep positions)
   or redeem (sell one revision, stop series Agent). Do not invent a
   force-unlock for `series_cash` / `fee_reserve` / `automation_funding`.

### Workflow 11 — Referral

1. **connect_wallet** with optional `inviteCode` at first registration
2. **get_referral** — show permanent `referralCode` and `referralUrl`
3. Explain fee split: referred users' 5% profit fee shares 2.5% with referrer
4. `rewards.pending` / `processing` are not yet paid — payout is manual monthly

### Workflow 12 — Update a Custom Index (new revision)

Use when `needsUpdate=true` (some legs ended but not all) or when the owner
wants a different basket for the **same thesis**. A different thesis is a new
index, not a revision.

1. `GET /custom-index/instances/:id` — read `components[]` (`lifecycle`,
   `configuredWeight`, `effectiveWeight`) and `canUpdate`.
2. `POST /custom-index/instances/:id/update-session` — server seeds an agent
   session from the current basket (terminal legs dropped, weights
   renormalized). Keep `sessionId`, `baseRevisionId`, `preferences`
   (inherited OI range; pass them back unchanged) and `seedBelowMinimum`.
   An all-ended basket still seeds (empty); ask the agent to refresh it.
3. Iterate via compose `/messages`, `/reconfigure`, `PATCH .../proposal`
   (billing rules are the same as normal compose). Only `composeStatus=ok`
   yields a new `proposalHash`; the browser never supplies the basket.
4. `POST /custom-index/instances/:id/revisions` with `composeSessionId`,
   `baseRevisionId`, `proposalHash`, optional `displayName`. The server reads
   the proposal/thesis/OI from the update session. Not billed, no new mint.
5. Errors: 409 `CUSTOM_INDEX_REVISION_CONFLICT` (someone changed the series —
   reload and restart from step 1, never auto-merge), 409
   `COMPOSE_SESSION_BUSY` (agent still refreshing — wait and retry).
6. Explain the result: existing positions are **not** traded or rebalanced.
   They stay on the old revision's `productKey`; only new investments land on
   the new revision. Each revision keeps its own paper curve (curves are never
   spliced). Redeeming any revision stops the whole series Agent.
7. `GET /custom-index/instances/:id/revisions` for history;
   `GET .../portfolio` for series totals with per-revision breakdown.

### Workflow 13 — Discover, evaluate and fork a public index

1. `GET /custom-index/catalog?search=&limit=&cursor=` (no auth) — ranked by
   paper return; `search` returns a single fuzzy-matched page.
2. `GET /custom-index/catalog/:slug` and `.../performance?range=` — owner-safe
   components and aggregate paper curve. Expired indexes are still readable.
3. Fork: `POST /custom-index/catalog/:slug/fork` (session, geo; needs its own
   payment when x402 is required, active sources only). To edit first,
   `PUT /custom-index/compose/sessions/:id/proposal` imports the public
   proposal into a compose session, then edit with PATCH and save.
4. Anonymous onboarding cards: `GET /custom-index/recipes?limit=&offset=`.

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
- **Product kinds**: `INDEX` = asset + direction. `MANAGED` = TACO and World
  Cup. `CUSTOM` = owner hybrid indexes under `/custom-index/*` (lazy-resolved,
  absent from `GET /products`).
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
  never hold the operational owner key.
- **pUSD collateral**: Trading uses pUSD. The platform accepts USDC.e and
  native USDC deposits and prepares pUSD through wrapping/swap flows. Wallet
  balance includes pUSD, USDC.e, native USDC, locked balance, and withdrawable
  balance.
- **Auto-redemption**: A cron job runs every 15 minutes to scan resolved
  markets. Winning CTF tokens are redeemed to pUSD, and a 5% profit fee is
  collected. Users do not need to manually claim settled positions.
- **EIP-712 V2**: Domain `{ name: "Polyvaults", version: "2", chainId: signatureChainId }`.
  All fund messages include `fundsChainId: 137`. Withdraw binds `token` + `chain`.
  Locked-funds release binds `itemIdsHash` + `acknowledgeRisk`.
  Legacy V1 (`version: "1"`) still accepted during rollout.
  Agent init uses a **different** domain: `{ name: "Polyvaults Index Automation",
  version: "2", chainId: 137 }`, primaryType `IndexAutomationInitialization`.
  Recurring-only mandates use `IndexAutomationMandate` (domain version `1`).
- **Custom Index Agent**: Optional automatic management. First buy is the
  initialization mandate, not `/invest`. Status `enabled | paused | disabling
  | disabled`. `disabled` means self-managed even if `requested` was historically
  true. Series cash is not position value.
- **Custom index series & revisions**: A series is one thesis over time.
  `seriesId` equals the first revision's `id`, so old URLs keep working. Every
  update creates an **immutable** revision (`revisionNumber`,
  `isCurrentRevision`); r1 keeps `custom-index:<userId>:<slug>`, r≥2 uses
  `...:r<N>`. Owner reads accept any revision id but canonicalize to the
  current one; exact old revisions appear only in `/revisions` and the series
  portfolio's `revisions[]`. Updates never trade or rebalance.
- **Paper vs real performance**: `paperPerformance` / `GET .../performance`
  (`mode: paper_since_creation`) is a **simulated** since-inception curve with
  frozen entry prices and virtual shares — not the user's P&L, and it excludes
  fees and real-fill differences. Entry crosses the **ask**, marks use the
  **bid** (never mid). Legs already `closed` or with spread > 0.35 are dropped
  at baseline and weights renormalized (`excludedLegs[]`, `baselineCoverage`);
  `status`: `ready` | `stale` (show the last full mark) | `awaiting_initial_quote`
  | terminal `baseline_unavailable` ("no simulated return", don't wait).
  Real money lives only in `GET .../portfolio`.
- **Series portfolio**: `hasDeposits` is independent of current value (a fully
  redeemed index with 0 P&L still "was invested"). `portfolio.historyCoverage`
  `{ complete, startsAt, reason }` — when `complete=false`
  (`series_history_started_after_multiple_revision_activity`) do not label the
  curve as full All-Time. Series `totalReturn` is recomputed from merged
  principal/P&L, never summed or averaged across revisions.
- **Custom index lifecycle**: a 15-minute sweep classifies each leg
  `open | ended_pending_settlement | resolved | unknown` (`components[]`
  keeps `configuredWeight`, exposes renormalized `effectiveWeight`). Only when
  **every** leg is affirmatively closed does the series become `expired`
  (`settlementStatus`: `pending` → `settled`); fetch failures yield `unknown`
  and never expire (fail-open). Expired indexes cannot be previewed/invested
  and leave the public list, but slug page, curve, valuation and redeem keep
  working. `PATCH` cannot reactivate them; recover with a new revision on live
  markets. `needsUpdate` = some (not all) legs ended.
- **Liquidity & spread disclosure**: relevance outranks liquidity in AI Auto.
  Only an `unusable` spread (above ~0.35) excludes a market (`SPREAD_TOO_WIDE`). A merely `wide` spread stays
  selectable with a non-blocking `WIDE_SPREAD` notice (`liquidityNotices`,
  per-leg `spread` in preview). Always tell the user previews are mid-market
  while FAK orders fill at the ask. `liquidityRecommendation` is a slider
  readout of the final basket, not a gate; a user-set OI range still filters.
- **Resolution-window filter**: compose drops markets whose resolution window
  already elapsed even if the market still shows as open (settling), and drops
  mutually-exclusive sibling legs (numeric buckets, date ladders, single-winner
  contests, rank/deadline conflicts across markets) so a basket never buys
  legs that cannot all pay.
- **Locked funds**: `lockedBalance` is application accounting, not an on-chain
  freeze. Explain via `GET /wallets/me/locked-funds`. Only round-gap and
  7-day `NEEDS_REVIEW` manual BUY are user-releasable.
- **Order status**: Unknown CLOB submit stays `PENDING` (not guessed `FAILED`).
  `NEEDS_REVIEW` is parked evidence, not a normal holding. Do not auto-retry
  ambiguous invest/init.
- **Residuals**: List rows may include `positionAction.hiddenByDefault` for
  leftover size below live CLOB minima. Presentation only — never zero NAV/PnL.
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
| 400 | `X402_PAYMENT_REQUIRED` | Settle compose x402 or pass `paymentAttemptId` / `composeSessionId` |
| 400 | `X402_DEPOSIT_SELF_TRANSFER` | Hop deposit cannot be paid from the same Safe it credits |
| 400 | `AUTOMATION_INITIAL_MARKET_UNAVAILABLE` | A selected outcome has no investable CLOB book; show the market |
| 400 | `Insufficient withdrawable balance` (+ detail) | Agent reserve unbacked; show `withdrawBlockedDetail`, route to locked funds / Agent disable |
| 400 | `X402_ATTEMPT_EXPIRED` | Consent window passed; fetch a fresh 402 challenge and sign again |
| 400 | `X402_SIGNATURE_SIGNER_MISMATCH` | EIP-3009 signer ≠ payer (wrong domain/wallet); response has `recoveredSigner` + `domain` |
| 400 | `X402_CONTRACT_SIGNATURE_REJECTED` | Contract payer's `isValidSignature` refused this digest |
| 400 | `X402_EIP7702_DELEGATOR_UNSUPPORTED` / `X402_EIP7702_SIGNATURE_UNSUPPORTED` | EOA delegated via 7702 to an unreviewed/uninitialized implementation; use another payer |
| 400 | `X402_FACILITATOR_REJECTED` | Our request was wrong (`facilitatorReason`); not an outage |
| 503 | `X402_FACILITATOR_UNAVAILABLE` | Facilitator unreachable/5xx; retry later |
| 503 | `AUTOMATION_INITIAL_BALANCE_UNAVAILABLE`, `..._MARKET_DATA_UNAVAILABLE`, `..._FUNDING_PLAN_FAILED`, `..._ORDER_PLAN_FAILED` | Init quote failed before signing; no wallet prompt is expected — retry the quote |
| 409 | `CUSTOM_INDEX_REVISION_CONFLICT` | Series changed under you; reload, restart the update (no auto-merge) |
| 409 | `COMPOSE_SESSION_BUSY` | Agent still processing this session; wait and retry |
| 409 | `CUSTOM_INDEX_SLUG_TAKEN` / `CUSTOM_INDEX_PUBLIC_SLUG_TAKEN` | Slug exists (message contains `slug already exists`); retry with `-2`, `-3` |
| 409 | `RECONFIGURE_SNAPSHOT_REQUIRED` / `RECONFIGURE_THESIS_REQUIRED` | Old session can't reconfigure; start a new session with the kept thesis (`recovery=start_new_session`) |
| 429 | `COMPOSE_QUOTA_*` | Per-user/IP compose quota hit; honor `retryAfterSec` |
| 401 | "Invalid or expired session" | Re-run `connect_wallet`; do not show the raw 401 as a product error |
| 401 | "Signature does not match" | User must sign with the wallet used at connect |
| 403 | "GEO_RESTRICTED" | Region restricted; close-only users may redeem/withdraw only |
| 409 | `AUTOMATION_DISABLE_DRAINING` | Agent stop committed; wait and retry redeem — do not submit a second sale |
| 429 | Too Many Requests | Rate limited; wait and retry |
| 500 | Server error | Retry once; if persistent, report to user |

When an invest call returns `overallStatus: "PARTIAL"`, inspect individual
`items[].orderStatus` or legacy `allocations[].orderStatus` and `failReason`.
Do not automatically retry failed strikes unless the user requests it.

---

## Custom Index (compose, revisions, Agent, ERC-8004)

Use `sessionToken` and only the public routes below. Compose is available
exclusively through `sessions`, `sessions/:id`, `.../messages`,
`.../reconfigure` and `.../proposal`.

### Compose + x402

1. `POST /custom-index/compose/sessions` with a stable `Idempotency-Key`.
2. If HTTP **402**: settle Polygon USDC `exact` (EOA EIP-3009, or Safe
   `CustomIndexX402Payment` then `POST /custom-index/payments/attempts/:id/safe-payload`
   — native USDC, never Polymarket relayer). Retry with `PAYMENT-SIGNATURE`.
   If no 402, x402 mode is off — continue. Show the signer/domain when
   `X402_SIGNATURE_SIGNER_MISMATCH` returns; a stale challenge returns
   `X402_ATTEMPT_EXPIRED` (re-fetch it). Keep the signing window >= 600s.
3. Messages on that `sessionId`. Each **delivered** inference consumes the
   attempt; interrupted runs stay reusable. `/reconfigure` is not billed.
   Send `Accept: text/event-stream` for SSE (`stage` / `token` /
   `recommendations` / `proposal` / `done` / `error`).
4. Save/fork binds the paid inference. Revisions do not charge or mint again.
5. Poll `agentRegistration` / `GET /custom-index/agents/:seriesId/registration.json`.
   Mint failure does not unwind the index.

Compose x402 is **not** an invest.

### Compose response semantics

| `composeStatus` | Meaning | Agent action |
|-----------------|---------|--------------|
| `ok` | New valid `proposal` + `proposalHash` | Only state that may replace/save a proposal |
| `guidance` | Not enough valid candidates or validation failed; has `diagnostics.guidanceReason` (`selection_review` / `liquidity_filter` / `coverage` / `live_validation`) | Keep prior proposal for comparison only; `proposalStaleForPreferences: true` forbids save/invest. Under `selection_review`, do **not** advise lowering OI — the direction review, not liquidity, emptied the basket |
| `conversation` | Answer/clarification, no proposal | Keep the existing proposal untouched |

- `preferences` in every response is the **effective** OI range — sync UI/state
  to it, not to the value you sent.
- `candidates[]` is the discovery pool, ordered by `relevanceRank`. Each has
  `relevanceTier` (`direct` | `related`), `eligibilityReasons[]` and
  non-blocking `liquidityNotices` (`WIDE_SPREAD`). Blocking `SPREAD_TOO_WIDE`
  markets are shown but not selectable.
- AI Auto targets 6 legs when the pool allows (3–5 if that is all that is
  relevant); a terminal review may shrink it. Wide-spread markets are not
  silently dropped — disclose the cost.
- `GET /custom-index/compose/sessions/:id` restores a session; when
  `operationInProgress: true` do not hydrate/unlock save from the snapshot,
  keep the pointer and retry. Discard a stored session pointer only when the
  session is explicitly not found / closed / expired.
- `POST .../reconfigure` body: `minOiUsd` / `maxOiUsd` (or in `preferences`),
  or `resetToLiquidityRecommendation: true`. Deterministic, no LLM.
- `PUT .../proposal` imports a public catalog proposal (fork-then-edit);
  `PATCH .../proposal` edits `items[]` (`marketRef`, `buyDirection`,
  `targetWeight`) after server live-validation.

### Owner instance APIs (session)

| Action | Endpoint |
|--------|----------|
| Save / fork | `POST /custom-index/instances`, `POST /custom-index/catalog/:slug/fork` |
| List | `GET /custom-index/instances?hasPosition=true&status=&includeHealth=` |
| Detail / health / preview | `GET/POST /custom-index/instances/:id[/health|/preview]` |
| Rename / archive / restore / publish | `PATCH /custom-index/instances/:id` (`displayName`, `status`, `visibility`; never the basket) |
| Paper performance | `GET /custom-index/instances/:id/performance?range=24h\|7d\|30d\|all` |
| Update session | `POST /custom-index/instances/:id/update-session` |
| Save revision | `POST /custom-index/instances/:id/revisions` |
| Revision history | `GET /custom-index/instances/:id/revisions[?limit=&before=]` |
| Self-managed invest | `POST /custom-index/instances/:id/invest` (`ProductInvestConfigured`) |
| Redeem | `POST /custom-index/instances/:id/redeem` (stops series Agent first; any revision's key accepted) |
| Series portfolio | `GET /custom-index/instances/:id/portfolio?range=` |
| Public catalog/recipes | `GET /custom-index/catalog[?search=&cursor=&limit=]`, `GET /custom-index/catalog/:slug[/performance]`, `GET /custom-index/recipes` |
| Automation pricing | `GET /custom-index/automation/public-conf` |
| Init quote / start | `POST .../automation/initialization[/quote]` |
| Status / pause / disable | `GET/POST .../automation` (`enable\|pause\|resume\|disable`) |
| Run receipts | `GET .../automation/runs` |

List/detail fields: `automationSummary` (`requested`, `status`, `pauseReason`,
`seriesCash`, `nextRunAt`, `initialization`), `hasPosition`, `hasDeposits`,
`positionAction`, `paperPerformance`, lifecycle (`expired`, `expiredAt`,
`settlementStatus`, `lifecycle { checkedAt, fresh, terminalCount, openCount,
unknownCount }`, `needsUpdate`, `canInvest`, `canUpdate`) and `components[]`.
Authenticated instance responses (list/detail/portfolio/performance/revisions/
health) are `Cache-Control: private, no-store`.

`PATCH` rules: archiving is refused while the series still holds unredeemed
FILLED positions (400); `expired` / `superseded` cannot return to `active` or
be archived; archived baskets whose legs all ended auto-flip to `expired`
(restore is refused when the basket is fully terminal). `draft` indexes are
always private and cannot be invested until patched to `active`. Only `active`
indexes can be public. API default `visibility` is `private`.

Public catalog: public + active, not expired, ranked by paper return;
`:slug` and `/performance` stay readable for `expired` (read-only page).
Components carry an optional event `question`; never owner/productKey/ids.

### x402 hop deposits (optional, default off)

`POST /x402/deposits` then settle. Wait for `status=forwarded`.
`forward_review` means the payment arrived but the credit needs manual review —
show **Needs review** + attempt id; do not ask the user to pay again; keep
polling. The payer must be an outside wallet, not the Safe being credited
(`X402_DEPOSIT_SELF_TRANSFER` otherwise).

Catalog/read: `GET /custom-index/catalog`, `GET /custom-index/catalog/:slug`,
`GET /custom-index/recipes`.

