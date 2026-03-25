---
name: narrative-index
description: >-
  Operates a custodial crypto index investment platform on Polymarket.
  Supports wallet management, USDC deposits, BTC Bullish/Bearish index
  preview and execution, portfolio monitoring, performance returns,
  and withdrawals. Use when the user mentions crypto index investing,
  BTC prediction markets, Polymarket, portfolio performance, USDC
  balance, deposits, withdrawals, or strike price allocation.
---

# Narrative Index — Agent Skill

## Platform Overview

Narrative Index Vault is a custodial BTC directional index product built on
Polymarket prediction markets. Each user gets a segregated Safe wallet managed
by the platform. Trades are executed as gasless FAK market orders via the
Polymarket Builder Program.

- **Base URL**: `https://api.polyvaults.ai`
- **Network**: Polygon
- **Collateral**: USDC.e (bridged) and USDC (native) — auto-converted
- **Auth model**: All requests identify the user by `userId` (UUID), obtained
  through `connect_wallet`.

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
Body: { "indexType": "BULLISH"|"BEARISH", "amount": 100, "userId": "..." }
```

Optional fields:
- `eventSlug` — override the default current-month event
- `userId` — when provided, the backend checks if a USDC→USDC.e swap is
  needed and calculates the swap fee (0.1%). Allocations are computed using
  the amount after fee deduction.

Returns `allocations[]`, `droppedStrikes`, `resolvedStrikes`,
`minimumDepositRequired`, `effectiveAmount`, `swapFee`.

```bash
curl -X POST https://api.polyvaults.ai/index/preview \
  -H 'Content-Type: application/json' \
  -d '{"indexType":"BULLISH","amount":100,"userId":"abc-123"}'
```

---

### 5. invest_index

Execute the index investment. Places FAK (Fill-and-Kill) market orders for
each qualified strike from the user's Safe balance. Orders fill immediately
against available liquidity; any unfilled portion is cancelled.

If the user's USDC.e balance is insufficient but they hold native USDC, the
platform automatically swaps the needed amount via Uniswap V3 before placing
orders.

```
POST /index/invest
Body: { "userId": "...", "indexType": "BULLISH"|"BEARISH", "amount": 100 }
```

Returns `depositId`, `allocations[]` (with `orderId`, `orderStatus`),
`overallStatus` (SUCCESS / PARTIAL / FAILED).

```bash
curl -X POST https://api.polyvaults.ai/index/invest \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","indexType":"BULLISH","amount":100}'
```

---

### 6. get_positions

List all index deposits and their per-strike allocations for a user.

```
GET /index/positions/:userId
```

Returns an array of deposits, each with `allocations[]`, `status`
(PENDING / EXECUTING / COMPLETED / PARTIAL / FAILED).

```bash
curl https://api.polyvaults.ai/index/positions/{userId}
```

---

### 7. get_portfolio

Dashboard metrics: NAV, deployed principal, available balance, PnL, total
return, and a return chart series.

```
GET /portfolio?userId=...&indexFilter=ALL&timeRange=24h
```

- `indexFilter`: ALL | BULLISH | BEARISH
- `timeRange`: 24h | 7d | 30d | all

Returns `nav`, `deployedPrincipal`, `availableBalance`, `pnl`,
`totalReturn`, `returnChart[]`.

```bash
curl 'https://api.polyvaults.ai/portfolio?userId=abc-123&timeRange=7d'
```

---

### 8. get_returns

Monthly daily-return data for BTC spot, Bullish Index, Bearish Index, and
outperformance.

```
GET /performance/returns?month=YYYY-MM
```

Limited to the last 6 months.

Returns an array of `{ date, btcPrice, btcReturn, bullishReturn,
bearishReturn, bullishOutperformance, bearishOutperformance }`.

```bash
curl 'https://api.polyvaults.ai/performance/returns?month=2026-03'
```

---

### 9. withdraw

Withdraw from the user's Safe wallet. Supports Polygon local transfers and
cross-chain withdrawals via the Polymarket Bridge (Ethereum, Arbitrum, Base,
Optimism, BSC, Solana).

```
POST /wallets/withdraw
Body: { "userId": "...", "toAddress": "0x...", "amount": 100, "chain": "ethereum" }
```

- `token` (optional): `"USDC"` or `"USDC.e"`. Only applies to Polygon withdrawals. Defaults to `"USDC.e"`.
- `chain` (optional): Target chain. Defaults to `"polygon"`. Supported: `polygon`, `ethereum`, `arbitrum`, `base`, `optimism`, `bsc`, `solana`.

Returns `transactionHash`, `status` ("SUBMITTED" for Polygon, "BRIDGING" for cross-chain).
Cross-chain responses also include `chain` and `bridgeDepositAddress` for status tracking.

```bash
# Polygon withdrawal
curl -X POST https://api.polyvaults.ai/wallets/withdraw \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","toAddress":"0xdead...","amount":100,"token":"USDC"}'

# Cross-chain withdrawal to Ethereum
curl -X POST https://api.polyvaults.ai/wallets/withdraw \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","toAddress":"0xdead...","amount":100,"chain":"ethereum"}'
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

### 10. get_btc_chart

Hourly BTC price data with strike price lines for visualization.

```
GET /chart/btc-strikes?indexType=BULLISH
```

Optional query params: `eventSlug`, `from`, `to` (ISO 8601).

Returns `priceData[]`, `strikePrices[]`, `nextStrike`, `resolved`,
`eventTitle`.

```bash
curl 'https://api.polyvaults.ai/chart/btc-strikes?indexType=BULLISH'
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

## Common Workflows

### Workflow 1 — New User Deposit & Invest

1. **connect_wallet** — obtain `userId` and `depositAddress`
2. Instruct the user to transfer USDC or USDC.e to `depositAddress` on Polygon
   (both accepted; native USDC is auto-converted when investing)
3. **get_wallet_balance** — confirm deposit arrived; show `totalBalance`
4. **preview_index** — pass `userId` to see swap fees if applicable; let user
   choose BULLISH or BEARISH and confirm the amount
5. **invest_index** — execute; auto-swap happens if needed; check `overallStatus`
   - If `PARTIAL`, inform the user which strikes failed
   - If `FAILED`, check balance and retry

### Workflow 2 — Check Investment Performance

1. **get_portfolio** — show NAV, PnL, totalReturn
2. **get_returns** — show daily index returns vs BTC for the relevant month
3. **get_positions** — show per-strike breakdown if the user wants details

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

---

## Key Concepts

- **IndexType**: `BULLISH` buys YES on "Will BTC hit $X?" (upside).
  `BEARISH` buys YES on "Will BTC drop below $X?" (downside).
- **Minimum investment**: $10 total. Individual strike allocations must be
  >= $1 and >= 5 shares.
- **Order type**: FAK (Fill-and-Kill) market orders, executed gaslessly.
  Orders fill immediately against available liquidity; unfilled remainder
  is cancelled. No orders stay pending on the order book.
- **Settlement**: Monthly, on the last day at UTC 23:59.
- **Weight formula**: Proprietary algorithm based on market liquidity,
  ensuring diversified allocation across strikes.
- **Safe wallet**: Platform-managed Gnosis Safe on Polygon. Users never hold
  private keys; the platform signs via encrypted EOA owner keys.
- **Dual USDC support**: The platform accepts both **USDC.e** (Bridged USDC,
  `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`) and **native USDC**
  (`0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359`) on Polygon. When investing,
  native USDC is automatically converted to USDC.e via Uniswap V3 (0.1% swap
  fee). Withdrawals support choosing either token. Users can deposit either
  token without manual conversion.

---

## Error Handling

| HTTP | Message | Action |
|------|---------|--------|
| 400 | "Insufficient balance" | Ask user to deposit more USDC or USDC.e |
| 400 | "Minimum investment is $10" | Use at least $10 for invest |
| 400 | "No active markets" | Current month event not yet live; try later |
| 400 | "Only the last 6 months are available" | Adjust `month` param |
| 500 | Server error | Retry once; if persistent, report to user |

When an `invest_index` call returns `overallStatus: "PARTIAL"`, inspect
individual `allocations[].orderStatus` to identify which strikes failed and
report them to the user. Do not automatically retry failed strikes unless the
user requests it.
