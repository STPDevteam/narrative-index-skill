---
name: narrative-index
description: >-
  Operates a custodial multi-asset directional index investment platform on
  Polymarket. Supports BTC, ETH, SOL (crypto), Oil (energy), Gold, Silver
  (metals). Provides wallet management, USDC deposits, Bullish/Bearish index
  preview and execution, portfolio monitoring, performance returns, chart data,
  and withdrawals. Use when the user mentions crypto/commodity index investing,
  prediction markets, Polymarket, portfolio performance, USDC balance, deposits,
  withdrawals, or strike price allocation.
---

# Narrative Index — Agent Skill

## Platform Overview

Narrative Index Vault is a custodial multi-asset directional index product built
on Polymarket prediction markets. Each user gets a segregated Safe wallet managed
by the platform. Trades are executed as gasless FAK market orders via the
Polymarket Builder Program.

- **Base URL**: `https://api.polyvaults.ai`
- **Network**: Polygon
- **Collateral**: USDC.e (bridged) and USDC (native) — auto-converted
- **Auth model**: All requests identify the user by `userId` (UUID), obtained
  through `connect_wallet`.
- **Signature auth**: Write endpoints (`invest`, `withdraw`, `redeem`) require
  EIP-712 typed data signatures from the user's connected wallet.
- **Geo-restriction**: Write endpoints return HTTP 403 (`GEO_RESTRICTED`) for
  requests from US IP addresses (detected via Cloudflare `cf-ipcountry` header).
  Read-only endpoints are unaffected.
- **Rate limiting**: Global rate limits apply — 10 requests/second and
  100 requests/minute per IP.
- **Key encryption**: Wallet private keys are encrypted at rest using AWS KMS
  (AES-256 symmetric encryption via AWS KMS API).

### Supported Assets

| Symbol | Name | Category | Status | Price Source |
|--------|------|----------|--------|-------------|
| BTC | Bitcoin | CRYPTO | active | Binance |
| ETH | Ethereum | CRYPTO | coming_soon | Binance |
| SOL | Solana | CRYPTO | coming_soon | Binance |
| OIL | Crude Oil | ENERGY | active | Pyth Network (WTI rolling) |
| GOLD | Gold | METALS | coming_soon | Pyth Network |
| SILVER | Silver | METALS | coming_soon | Pyth Network |

Assets with `active` status have live Polymarket markets and can be traded.
Assets with `coming_soon` are registered but not yet available for investment.

For full endpoint schemas see [references/api-reference.md](references/api-reference.md).
For strategy mechanics see [references/strategy-guide.md](references/strategy-guide.md).

---

## Available Tools

### 1. connect_wallet

Register or log in a user. First-time calls auto-create a user record and a
deployed Safe wallet.

```
POST /auth/connect
Body: { "walletAddress": "0x..." }
```

Returns `userId`, `safeAddress`, `depositAddress`, `isNewUser`.

```bash
curl -X POST https://api.polyvaults.ai/auth/connect \
  -H 'Content-Type: application/json' \
  -d '{"walletAddress":"0x1234...abcd"}'
```

---

### 2. get_wallet_balance

Query the user's Safe wallet balance, including both USDC.e (bridged) and
native USDC.

```
GET /wallets/:userId/balance
```

Returns `formattedBalance` (USDC.e), `formattedNativeBalance` (native USDC),
and `totalBalance` (sum of both).

```bash
curl https://api.polyvaults.ai/wallets/{userId}/balance
```

---

### 3. get_deposit_address

Get the Safe wallet address where the user should send USDC or USDC.e.
Both are accepted; native USDC is automatically converted to USDC.e when
investing.

```
GET /wallets/:userId/deposit-address
```

Returns `address`, `network` ("Polygon"), `token` ("USDC / USDC.e").

```bash
curl https://api.polyvaults.ai/wallets/{userId}/deposit-address
```

---

### 4. preview_index

Preview how funds would be allocated across strikes before investing.
Only strikes meeting the $1 minimum are returned.

```
POST /index/preview
Body: { "indexType": "BULLISH"|"BEARISH", "amount": 100, "userId": "...", "asset": "BTC" }
```

Optional fields:
- `asset` — asset symbol (default: `BTC`). Available: BTC, OIL, etc.
- `eventSlug` — override the default current-month event
- `userId` — when provided, the backend checks if a USDC→USDC.e swap is
  needed and calculates the swap fee (0.1%). Allocations are computed using
  the amount after fee deduction.

Returns `asset`, `allocations[]`, `droppedStrikes`, `resolvedStrikes`,
`minimumDepositRequired`, `effectiveAmount`, `swapFee`.

```bash
curl -X POST https://api.polyvaults.ai/index/preview \
  -H 'Content-Type: application/json' \
  -d '{"indexType":"BULLISH","amount":100,"userId":"abc-123","asset":"OIL"}'
```

---

### 5. invest_index

Execute the index investment. Places FAK (Fill-and-Kill) market orders for
each qualified strike from the user's Safe balance. Orders fill immediately
against available liquidity; any unfilled portion is cancelled.

If the user's USDC.e balance is insufficient but they hold native USDC, the
platform automatically swaps the needed amount via Uniswap V3 before placing
orders.

> Requires EIP-712 signature. See [api-reference.md](references/api-reference.md#signature-authentication) for signing details.

```
POST /index/invest
Body: { "userId": "...", "indexType": "BULLISH"|"BEARISH", "amount": 100, "asset": "BTC", "signature": "0x...", "nonce": 1740643200000 }
```

Optional: `asset` (default: BTC), `eventSlug`.

Returns `depositId`, `asset`, `allocations[]` (with `orderId`, `orderStatus`),
`overallStatus` (SUCCESS / PARTIAL / FAILED).

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
`unrealizedPnl`, `realizedPnl`, `pnl`, `totalReturn`, `returnChart[]`.

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
- Limited to the last 6 months.

Returns an array of `{ date, asset, btcPrice, btcReturn, bullishReturn,
bearishReturn, bullishOutperformance, bearishOutperformance }`.

```bash
curl 'https://api.polyvaults.ai/performance/returns?month=2026-03&asset=OIL'
```

---

### 9. withdraw

Withdraw from the user's Safe wallet. Supports Polygon local transfers and
cross-chain withdrawals via the Polymarket Bridge (Ethereum, Arbitrum, Base,
Optimism, BSC, Solana).

> Requires EIP-712 signature.

```
POST /wallets/withdraw
Body: { "userId": "...", "toAddress": "0x...", "amount": 100, "chain": "ethereum", "signature": "0x...", "nonce": 1740643200000 }
```

- `token` (optional): `"USDC"` or `"USDC.e"`. Only applies to Polygon withdrawals. Defaults to `"USDC.e"`.
- `chain` (optional): Target chain. Defaults to `"polygon"`. Supported: `polygon`, `ethereum`, `arbitrum`, `base`, `optimism`, `bsc`, `solana`.

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
Body: { "userId": "...", "amount": 100, "chain": "ethereum" }
```

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

### 11. get_accounting_positions

Current positions with unrealized PnL breakdown.

```
GET /accounting/:userId/positions
```

```bash
curl https://api.polyvaults.ai/accounting/{userId}/positions
```

---

### 12. get_trades

Trade history for the user.

```
GET /accounting/:userId/trades?limit=50
```

```bash
curl 'https://api.polyvaults.ai/accounting/{userId}/trades?limit=50'
```

---

### 13. get_pnl

P&L report: totalDeposits, totalWithdrawals, currentBalance,
totalRealizedPnL, totalUnrealizedPnL, returnPercentage.

```
GET /accounting/:userId/pnl
```

```bash
curl https://api.polyvaults.ai/accounting/{userId}/pnl
```

---

### 14. early_redeem

Market-sell all active positions for a given direction (BULLISH or BEARISH).
A 2% fee is charged on any profit and sent to the platform fee address.

Resolved positions are automatically redeemed by a cron job every hour —
this endpoint is only for **early** (pre-settlement) redemption.

> Requires EIP-712 signature.

```
POST /index/redeem
Body: { "userId": "...", "direction": "BULLISH"|"BEARISH", "asset": "BTC", "signature": "0x...", "nonce": 1740643200000 }
```

Optional: `asset` (default: BTC).

Returns `sold`, `totalReceived`, `totalCost`, `profit`, `fee`, `results[]`.

```bash
curl -X POST https://api.polyvaults.ai/index/redeem \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","direction":"BULLISH","signature":"0x...","nonce":1740643200000}'
```

---

### 15. get_portfolio_breakdown

Get separate metrics for BULLISH and BEARISH directions. Supports per-asset
filtering.

```
GET /portfolio/breakdown?userId=...&asset=BTC
```

- `asset` (optional): when provided, returns breakdown for that asset only.
  When omitted, returns overall breakdown plus `assets[]` array with per-asset
  details.

Returns `bullish` and `bearish`, each with `deployedPrincipal`,
`positionValue`, `unrealizedPnl`, `totalReturn`.

```bash
curl 'https://api.polyvaults.ai/portfolio/breakdown?userId=abc-123&asset=OIL'
```

---

### 16. get_market_status

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

### 17. get_assets

List all registered assets with their current status.

```
GET /assets
```

Returns `{ assets: [{ symbol, name, category, status }] }`.

```bash
curl https://api.polyvaults.ai/assets
```

---

## Common Workflows

### Workflow 1 — New User Deposit & Invest

1. **connect_wallet** — obtain `userId` and `depositAddress`
2. Instruct the user to transfer USDC or USDC.e to `depositAddress` on Polygon
   (both accepted; native USDC is auto-converted when investing)
3. **get_wallet_balance** — confirm deposit arrived; show `totalBalance`
4. **get_assets** or **get_market_status** — check which assets are `active`
5. **preview_index** — pass `userId` and `asset` to see swap fees if applicable;
   let user choose BULLISH or BEARISH and confirm the amount
6. **invest_index** — execute; auto-swap happens if needed; check `overallStatus`
   - If `PARTIAL`, inform the user which strikes failed
   - If `FAILED`, check balance and retry

### Workflow 2 — Check Investment Performance

1. **get_portfolio** — show NAV, PnL, totalReturn (optionally filter by `asset`)
2. **get_returns** — show daily index returns vs asset spot for the relevant month
3. **get_positions** — show per-strike breakdown if the user wants details
4. **get_portfolio_breakdown** — compare BULLISH vs BEARISH performance

### Workflow 3 — Withdraw Funds (Polygon)

1. **get_wallet_balance** — confirm available balance (USDC.e + native USDC)
2. Ask the user which token to withdraw: USDC or USDC.e (default USDC.e)
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

1. **get_portfolio_breakdown** — show per-direction PnL to help decide which to redeem
2. **early_redeem** — sell all positions for the chosen direction (BULLISH or BEARISH)
3. Inform user of `profit`, `fee` (2% of profit if positive), and `totalReceived`
4. Funds return to the Safe wallet as USDC.e

### Workflow 6 — Explore Available Assets

1. **get_assets** — see all registered assets and their status
2. **get_market_status** — check which assets have live Polymarket markets
3. **get_chart** — view price data and strike lines for any asset
4. **get_returns** — compare historical performance across assets

---

## Key Concepts

- **Multi-asset support**: The platform supports multiple underlying assets
  across crypto (BTC, ETH, SOL), energy (Oil), and metals (Gold, Silver).
  Each asset has independent Polymarket events, strike prices, and price feeds.
- **Asset status**: `active` = live markets, can invest. `pending_liquidity` =
  market exists but insufficient open interest. `coming_soon` = registered but
  no market yet.
- **IndexType**: `BULLISH` buys YES on "Will [asset] hit $X?" (upside).
  `BEARISH` buys YES on "Will [asset] drop below $X?" (downside).
- **Minimum investment**: $10 total. Individual strike allocations must be
  >= $1 and >= 5 shares.
- **Order type**: FAK (Fill-and-Kill) market orders, executed gaslessly.
  Orders fill immediately against available liquidity; unfilled remainder
  is cancelled. No orders stay pending on the order book.
- **Settlement**: Monthly. Polymarket uses Eastern Time (ET) for market
  creation and settlement.
- **Weight formula**: Proprietary algorithm based on market liquidity,
  ensuring diversified allocation across strikes.
- **Safe wallet**: Platform-managed Gnosis Safe on Polygon. Users never hold
  private keys; the platform signs via encrypted EOA owner keys (AWS KMS).
- **Dual USDC support**: The platform accepts both **USDC.e** (Bridged USDC,
  `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`) and **native USDC**
  (`0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359`) on Polygon. When investing,
  native USDC is automatically converted to USDC.e via Uniswap V3 (0.1% swap
  fee). Withdrawals support choosing either token. Users can deposit either
  token without manual conversion.
- **Auto-redemption**: A cron job runs every hour to scan for resolved markets.
  Winning CTF tokens are automatically redeemed to USDC.e, and a 2% profit fee
  is collected. Users do not need to manually claim settled positions.
- **Realized PnL tracking**: Both manual early redemption and auto-settlement
  profits/losses are tracked via `realizedPnl` in portfolio metrics.
- **Price sources**: Crypto assets use Binance spot prices. Commodities use
  Pyth Network oracle feeds (with automatic WTI futures contract rolling for Oil).

---

## Error Handling

| HTTP | Message | Action |
|------|---------|--------|
| 400 | "Insufficient balance" | Ask user to deposit more USDC or USDC.e |
| 400 | "Minimum investment is $10" | Use at least $10 for invest |
| 400 | "No active markets" | Current month event not yet live; try later |
| 400 | "Only the last 6 months are available" | Adjust `month` param |
| 400 | "Signature expired" | Regenerate nonce (use `Date.now()`) and re-sign |
| 400 | "Nonce already used" | Generate a fresh nonce — each nonce is single-use |
| 401 | "Signature does not match" | User must sign with the wallet used at connect |
| 403 | "GEO_RESTRICTED" | US IP detected; trading/withdrawals blocked by policy |
| 429 | Too Many Requests | Rate limited; wait and retry |
| 500 | Server error | Retry once; if persistent, report to user |

When an `invest_index` call returns `overallStatus: "PARTIAL"`, inspect
individual `allocations[].orderStatus` to identify which strikes failed and
report them to the user. Do not automatically retry failed strikes unless the
user requests it.
