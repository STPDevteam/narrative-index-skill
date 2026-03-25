# Narrative Index — API Reference

> Base URL: `https://api.polyvaults.ai`
>
> All requests use JSON. Identify users by `userId` (UUID from `POST /auth/connect`).

## Contents

- [Authentication](#authentication) — `POST /auth/connect`
- [Wallet Management](#wallet-management) — balance, deposit address, withdraw-fee, withdraw
- [Index Investment](#index-investment) — preview, invest, positions
- [Performance](#performance) — monthly daily returns
- [Chart](#chart) — BTC price + strike lines
- [Portfolio Dashboard](#portfolio-dashboard) — NAV, PnL, totalReturn
- [Accounting](#accounting) — positions, trades, pnl report
- [Enum Reference](#enum-reference) — all enum values

---

## Authentication

### POST /auth/connect

Register or log in.

**Request:**

```json
{ "walletAddress": "0x1234567890abcdef1234567890abcdef12345678" }
```

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
  "createdAt": "2026-03-12T08:00:00.000Z"
}
```

| Field | Description |
|-------|-------------|
| userId | Unique user ID for all subsequent calls |
| safeAddress | Platform-managed Safe wallet |
| depositAddress | Same as safeAddress; send USDC or USDC.e here |
| isNewUser | true on first connect |

---

## Wallet Management

### GET /wallets/:userId

Full wallet info (ownerAddress, safeAddress, isDeployed, isApproved).

### GET /wallets/:userId/balance

```json
{
  "usdcBalance": "1500000000",
  "formattedBalance": "1500.00",
  "nativeUsdcBalance": "500000000",
  "formattedNativeBalance": "500.00",
  "totalBalance": "2000.000000"
}
```

| Field | Description |
|-------|-------------|
| usdcBalance / formattedBalance | USDC.e (bridged) balance |
| nativeUsdcBalance / formattedNativeBalance | Native USDC balance |
| totalBalance | Sum of both |

### GET /wallets/:userId/deposit-address

```json
{ "address": "0x...", "network": "Polygon", "token": "USDC / USDC.e" }
```

Both USDC and USDC.e deposits are accepted. Native USDC is auto-converted
to USDC.e when investing.

### GET /wallets/withdraw-fee

```json
{ "feeRate": 0.01, "feeToken": "USDC", "network": "Polygon", "minWithdrawal": 0.01 }
```

### POST /wallets/withdraw

Supports Polygon local transfer and cross-chain withdrawal via Polymarket Bridge.

**Request:**

```json
{ "userId": "uuid", "toAddress": "0x...", "amount": 100, "chain": "ethereum" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| userId | string | Yes | User ID |
| toAddress | string | Yes | Destination address (EVM `0x...` or Solana base58) |
| amount | number | Yes | Amount in dollars, min $0.01 |
| token | string | No | Polygon only: `"USDC"` or `"USDC.e"` (default `"USDC.e"`) |
| chain | string | No | Target chain (default `"polygon"`). Options: `polygon`, `ethereum`, `arbitrum`, `base`, `optimism`, `bsc`, `solana` |

For Polygon: auto-swaps between USDC/USDC.e if needed.
For cross-chain: consolidates to USDC.e, sends to Bridge deposit address.

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
{ "userId": "uuid", "amount": 100, "chain": "ethereum" }
```

**Response:**

```json
{
  "chain": "Ethereum", "inputAmount": 100, "estimatedOutput": 99.99,
  "estimatedOutputBaseUnit": "99990000",
  "fees": { "gasUsd": 0.003, "totalImpactUsd": 0 },
  "estimatedTimeMs": 25000, "minWithdrawal": 7
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

## Index Investment

### POST /index/preview

Preview strike allocation before investing.

**Request:**

```json
{ "indexType": "BULLISH", "amount": 100, "userId": "abc-123" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| indexType | enum | Yes | BULLISH or BEARISH |
| amount | number | Yes | Investment amount ($), min 10 |
| eventSlug | string | No | Override default current-month event |
| userId | string | No | When provided, calculates swap fee if USDC→USDC.e conversion is needed |

**Response:**

```json
{
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
immediately against available liquidity. If USDC.e balance is insufficient
but native USDC is available, an automatic Uniswap V3 swap is performed
before placing orders.

**Request:**

```json
{ "userId": "uuid", "indexType": "BULLISH", "amount": 100 }
```

Optional: `eventSlug`.

**Response:**

```json
{
  "depositId": "uuid",
  "userId": "user-123",
  "indexType": "BULLISH",
  "totalDeposit": 100,
  "totalAllocated": 98.50,
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
      "orderStatus": "FILLED"
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

---

## Performance

### GET /performance/returns?month=YYYY-MM

Daily return data. Limited to the last 6 months.

**Response:**

```json
[
  {
    "date": "2026-03-01",
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
| btcReturn | BTC cumulative return from month start (0.008 = 0.8%) |
| bullishReturn | Bullish Index cumulative return |
| bearishReturn | Bearish Index cumulative return |
| *Outperformance | indexReturn - btcReturn |

---

## Chart

### GET /chart/btc-strikes

Hourly BTC price data + strike price lines.

**Query params:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| indexType | enum | Yes | — | BULLISH or BEARISH |
| eventSlug | string | No | current month | Event slug |
| from | string | No | month start | ISO 8601 start time |
| to | string | No | now | ISO 8601 end time |

**Response:**

```json
{
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
| indexFilter | enum | No | ALL | ALL / BULLISH / BEARISH |
| timeRange | enum | No | all | 24h / 7d / 30d / all |

**Response:**

```json
{
  "nav": 1520.50,
  "deployedPrincipal": 1000.00,
  "availableBalance": 500.00,
  "pnl": 20.50,
  "totalReturn": 0.0205,
  "returnChart": [
    { "timestamp": "2026-03-12T01:00:00.000Z", "totalReturn": 0.018 }
  ],
  "indexFilter": "ALL",
  "timeRange": "24h"
}
```

| Field | Description |
|-------|-------------|
| nav | Net asset value = position market value + available balance |
| deployedPrincipal | Sum of all FILLED index investments |
| availableBalance | Total USDC available in Safe wallet (USDC.e + native USDC) |
| pnl | Current market value of positions - deployed principal |
| totalReturn | pnl / deployedPrincipal (0.0205 = 2.05%) |

returnChart granularity: 24h → hourly, 7d/30d/all → daily.

---

## Accounting

### GET /accounting/:userId/positions

Current positions with unrealizedPnL.

### GET /accounting/:userId/trades?limit=50

Trade history.

### GET /accounting/:userId/pnl

P&L report: totalDeposits, totalWithdrawals, currentBalance,
totalRealizedPnL, totalUnrealizedPnL, returnPercentage.

---

## Enum Reference

| Enum | Values |
|------|--------|
| IndexType | BULLISH, BEARISH |
| PortfolioIndexFilter | ALL, BULLISH, BEARISH |
| PortfolioTimeRange | 24h, 7d, 30d, all |
| OrderSide | BUY, SELL |
| OrderType | GTC, GTD, FOK, FAK |
| IndexDepositStatus | PENDING, EXECUTING, COMPLETED, PARTIAL, FAILED |
| TradeStatus | PENDING, MATCHED, MINED, CONFIRMED, FAILED |
