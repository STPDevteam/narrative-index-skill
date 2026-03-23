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
- **Collateral**: USDC.e
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

Query available USDC.e balance in the user's Safe wallet.

```
GET /wallets/:userId/balance
```

Returns `formattedBalance` (human-readable dollar string).

```bash
curl https://api.polyvaults.ai/wallets/{userId}/balance
```

---

### 3. get_deposit_address

Get the Safe wallet address where the user should send USDC.e.

```
GET /wallets/:userId/deposit-address
```

Returns `address`, `network` ("Polygon"), `token` ("USDC").

```bash
curl https://api.polyvaults.ai/wallets/{userId}/deposit-address
```

---

### 4. preview_index

Preview how funds would be allocated across strikes before investing.
Only strikes meeting the $1 minimum are returned.

```
POST /index/preview
Body: { "indexType": "BULLISH"|"BEARISH", "amount": 100 }
```

Optional field `eventSlug` overrides the default current-month event.

Returns `allocations[]` (weight, allocation per strike), `droppedStrikes`,
`resolvedStrikes`, `minimumDepositRequired`.

```bash
curl -X POST https://api.polyvaults.ai/index/preview \
  -H 'Content-Type: application/json' \
  -d '{"indexType":"BULLISH","amount":100}'
```

---

### 5. invest_index

Execute the index investment. Places FAK market orders for each qualified
strike from the user's Safe balance.

```
POST /index/invest
Body: { "userId": "...", "indexType": "BULLISH"|"BEARISH", "amount": 100 }
```

Returns `depositId`, `allocations[]` (with `orderId`, `orderStatus`),
`hasPlacedOrders`, `overallStatus` (SUCCESS / PARTIAL / FAILED).

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

Withdraw USDC.e from the user's Safe wallet to an external address.
A 1% fee applies.

```
POST /wallets/withdraw
Body: { "userId": "...", "toAddress": "0x...", "amount": 100 }
```

Returns `transactionHash`, `status` ("SUBMITTED").

```bash
curl -X POST https://api.polyvaults.ai/wallets/withdraw \
  -H 'Content-Type: application/json' \
  -d '{"userId":"abc-123","toAddress":"0xdead...","amount":100}'
```

Query withdrawal fee beforehand with `GET /wallets/withdraw-fee`.

---

## Common Workflows

### Workflow 1 — New User Deposit & Invest

1. **connect_wallet** — obtain `userId` and `depositAddress`
2. Instruct the user to transfer USDC.e to `depositAddress` on Polygon
3. **get_wallet_balance** — confirm deposit arrived
4. **preview_index** — show allocation breakdown; let user choose BULLISH or
   BEARISH and confirm the amount
5. **invest_index** — execute; check `overallStatus`
   - If `PARTIAL`, inform the user which strikes failed
   - If `FAILED`, check balance and retry

### Workflow 2 — Check Investment Performance

1. **get_portfolio** — show NAV, PnL, totalReturn
2. **get_returns** — show daily index returns vs BTC for the relevant month
3. **get_positions** — show per-strike breakdown if the user wants details

### Workflow 3 — Withdraw Funds

1. **get_wallet_balance** — confirm available balance
2. Inform the user about the 1% withdrawal fee
   (`GET /wallets/withdraw-fee` for exact rate)
3. **withdraw** — execute; return `transactionHash` for on-chain tracking

---

## Key Concepts

- **IndexType**: `BULLISH` buys YES on "Will BTC hit $X?" (upside).
  `BEARISH` buys YES on "Will BTC drop below $X?" (downside).
- **Minimum investment**: $10 total. Individual strike allocations must be
  >= $1 and >= 5 shares.
- **Order type**: FAK (Fill-and-Kill) market orders, executed gaslessly.
- **Settlement**: Monthly, on the last day at UTC 23:59.
- **Weight formula**: Proprietary algorithm based on market liquidity,
  ensuring diversified allocation across strikes.
- **Safe wallet**: Platform-managed Gnosis Safe on Polygon. Users never hold
  private keys; the platform signs via encrypted EOA owner keys.
- **USDC.e required**: The platform only accepts **USDC.e** (Bridged USDC,
  `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`) on Polygon. If the user holds
  native USDC (`0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359`), they must swap
  to USDC.e first via Uniswap V3 on Polygon
  (SwapRouter: `0xE592427A0AEce92De3Edee1F18E0157C05861564`).
  Direct link: https://app.uniswap.org/swap?chain=polygon&inputCurrency=0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359&outputCurrency=0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174

---

## Error Handling

| HTTP | Message | Action |
|------|---------|--------|
| 400 | "Insufficient balance" | Ask user to deposit more USDC.e |
| 400 | "Amount below minimum" | Use at least $10 for invest |
| 400 | "No active markets" | Current month event not yet live; try later |
| 400 | "Only the last 6 months are available" | Adjust `month` param |
| 500 | Server error | Retry once; if persistent, report to user |

When an `invest_index` call returns `overallStatus: "PARTIAL"`, inspect
individual `allocations[].orderStatus` to identify which strikes failed and
report them to the user. Do not automatically retry failed strikes unless the
user requests it.
