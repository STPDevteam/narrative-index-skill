# Narrative Index — Strategy Guide

## Contents

- [Product Concept](#product-concept) — what the platform does
- [Supported Assets](#supported-assets) — multi-asset coverage
- [Product Families](#product-families) — INDEX, NarrativeBasket, World Cup
- [Strategy Types](#strategy-types) — Bullish vs Bearish index
- [Polymarket Market Structure](#polymarket-market-structure) — event, market, token hierarchy
- [Weight Calculation](#weight-calculation) — allocation algorithm
- [Iterative Pruning](#iterative-pruning) — minimum order constraints
- [Order Execution](#order-execution) — FAK gasless orders
- [Signing And Wallet Security](#signing-and-wallet-security) — Deposit Wallet, Safe, CLOB auth
- [Settlement](#settlement) — monthly binary settlement
- [Performance Scenarios](#performance-scenarios) — expected returns by scenario
- [Risks](#risks) — key risk factors

---

## Product Concept

Polyvaults creates productized Polymarket baskets. Users get exposure to
asset-direction indices, narrative baskets, and event-specific bracket products
without selecting every outcome token manually. Deposits can arrive as USDC.e or
native USDC on Polygon; trading collateral is prepared as pUSD before CLOB
execution.

---

## Supported Assets

The platform supports multiple underlying assets across three categories:

| Category | Assets | Price Source |
|----------|--------|-------------|
| **Crypto** | BTC, ETH, SOL | Binance spot price |
| **Energy** | Crude Oil (WTI) | Yahoo CL=F, MEXC futures fallback |
| **Metals** | Gold, Silver | Pyth Network (spot) |

Current active asset-direction indices are BTC, ETH, and OIL. SOL, GOLD, and
SILVER remain registered but `coming_soon` until market liquidity is sufficient.

---

## Product Families

### INDEX

One asset plus one direction, addressed by product keys such as `btc-bullish`,
`btc-bearish`, `eth-bullish`, or `oil-bearish`. Legacy `/index/*` endpoints
continue to route into this product family.

### NarrativeBasket

Managed strategy definitions, currently including `narrative-basket:taco-v1`.
Each definition lists pre-vetted Polymarket event strikes, sides, and clusters.
TACO weights eligible strikes by event open interest and can auto-compound
profitable settlements back into the same product.

### World Cup 2026 Brackets

Managed products under `worldcup-2026:*`. They buy team sub-markets for a
round, support presets (`europe`, `south-am`, `host-nations`, `top-seeded`) or
custom `teamRefs`, and can auto-roll to the next stage when enabled. Users pick
an `exitAfterStageKey` (group stage through final) to stop rolling after that
stage settles. Cash waiting for a future round is `lockedBalance` until
reinvested or released via signed `stop-rolling` (scoped by `rootDepositId`).

---

## Strategy Types

### Bullish Index

- Buys **YES** on multiple "Will [asset] hit $X?" contracts (UP direction)
- Profits when the asset breaks through strike prices upward
- More strikes breached = higher return (leverage effect)

### Bearish Index

- Buys **YES** on multiple "Will [asset] drop below $X?" contracts (DOWN direction)
- Profits when the asset falls through strike prices downward
- Can be used to hedge existing spot positions

Both strategies work identically across all supported assets — only the
underlying price feed and Polymarket event differ.

---

## Polymarket Market Structure

```
Event (monthly, per asset)
  └─ "What price will Bitcoin hit in April 2026?"
      ├─ Market: "Will BTC hit $85,000?"   → conditionId → tokenId(YES), tokenId(NO)
      ├─ Market: "Will BTC hit $90,000?"   → conditionId → tokenId(YES), tokenId(NO)
      ├─ Market: "Will BTC hit $95,000?"   → conditionId → tokenId(YES), tokenId(NO)
      └─ ...

  └─ "What price will Oil hit in April 2026?"
      ├─ Market: "Will Oil hit $65?"       → conditionId → tokenId(YES), tokenId(NO)
      ├─ Market: "Will Oil hit $70?"       → conditionId → tokenId(YES), tokenId(NO)
      └─ ...
```

- Each market has a **YES price** (implied probability of hitting) and **NO price**
- YES + NO ≈ $1.00
- Example: YES at $0.45 means 45% implied probability; profit if hit = $0.55 per share

---

## Weight Calculation

Funds are distributed by product:

- **INDEX**: liquidity and price aware strike weighting for one asset/direction.
- **TACO / NarrativeBasket**: eligible event OI proportion.
- **World Cup**: `subMarketOpenInterest × buyPrice` for each selected team.

All products feed into the same allocation engine and order executor. Invalid
prices and allocations below Polymarket minimum order size are pruned before
orders are placed.

---

## Iterative Pruning

After initial allocation, strikes that do not meet minimums are removed
one at a time (worst-deficit first), and weights are recalculated:

1. Each strike must meet Polymarket minimum order amount, currently **$1.00**
2. Product-specific admission checks must pass, such as OI thresholds
3. Price must be within **$0.01 – $0.99**

The pruning loop removes the single worst-deficit strike per iteration until
all remaining strikes satisfy these constraints.

---

## Order Execution

- Order type: **FAK** (Fill-and-Kill) — immediate partial/full fill, remainder cancelled
- Execution: **Gasless** via Polymarket CLOB V2 and builder attribution
- Collateral: **pUSD**, prepared from existing pUSD, USDC.e, or native USDC as needed
- Slippage: orders use a bounded worst price; default slippage is 2%
- Each strike is an independent order; one failure does not block others
- The early per-user/platform active-position caps have been removed so
  auto-compounding winners can keep rolling growing balances.

---

## Signing And Wallet Security

- New users normally use Polymarket **Deposit Wallets** (`POLY_1271`); legacy
  users may still use Safe (`POLY_GNOSIS_SAFE`).
- EIP-712 **V2** domain: `{ name: "Polyvaults", version: "2", chainId: signatureChainId }`.
  Fund messages include `fundsChainId: 137` (Polygon execution). Base Smart
  Wallet users sign on Base (`8453`) while funds execute on Polygon.
- `autoCompound` is part of the invest typed-data message (not a post-sign flag).
- World Cup `stop-rolling` and `retry-roll` require signatures with `rootDepositId`
  to scope the action to one chain.
- Withdraw signatures bind `token` and destination `chain` to prevent replay.
- The main API never decrypts owner EOA keys. It calls an isolated
  signing-service that is the only service with AWS KMS decrypt permission and
  access to `user_wallets.encryptedPrivateKey`.
- Signing-service policy can run in `off`, `audit`, or `enforce` mode. It can
  deny raw transaction signing, rate-limit signing, require mTLS, validate
  relayer destination call data, and bind withdrawals to the user's EIP-712
  `Withdraw` signature.
- CLOB API key derivation must be signed by the owner EOA. Contract wallet
  addresses should not be used as the CLOB L1 `POLY_ADDRESS`; Deposit Wallet
  orders still use `POLY_1271` at the order layer.
- CLOB V2 uses per-order `POLY_BUILDER_CODE` for builder attribution.

---

## Settlement

- Asset-direction monthly contracts settle on the Polymarket event schedule.
  Polymarket uses **Eastern Time (ET)** for monthly asset events.
- Settlement is binary: YES pays $1.00, NO pays $0.00
- Returns depend on how many selected outcomes resolve in favor
- Markets for different assets may be created on different days of the month
  (typically the 1st–3rd). The platform automatically handles late market
  creation.
- Auto-redemption runs every 15 minutes, redeems resolved winning CTF tokens to
  pUSD, and charges a 5% fee on positive profit. Users with a referrer split
  the fee: 2.5% platform + 2.5% referrer (accrued, manual monthly payout).
- Auto-compound products can reinvest net proceeds automatically. TACO rolls
  back into itself; World Cup products can roll into the next stage product
  when the schedule gate is open.

---

## Performance Scenarios

### Bullish Index (applies to any asset)

| Scenario | Asset Move | Index Return | Notes |
|----------|-----------|--------------|-------|
| Sustained rally | +30% | ~+42% | Multiple strikes breached |
| Spike & pullback | +8% EOM | ~+18% | Early breaches lock in gains |
| Sideways | ~0% | ~-5% | Time value decay |
| Crash | -20% | ~-80% | All YES contracts expire worthless |

### Bearish Index (applies to any asset)

| Scenario | Asset Move | Index Return | Notes |
|----------|-----------|--------------|-------|
| Sustained decline | -22% | ~+38% | Multiple breakdown strikes triggered |
| Flash crash recovery | -5% EOM | ~+12% | Early breakdowns lock in gains |
| Sideways | ~0% | ~-5% | Time value decay |
| Rally | +25% | ~-75% | All breakdown contracts expire worthless |

---

## Risks

1. **Directional risk** — wrong direction can cause significant loss
2. **Time decay** — correct direction but insufficient volatility still loses
3. **Liquidity risk** — thin markets may cause worse fill prices
4. **Smart contract risk** — depends on Polymarket and Polygon network health
5. **Oracle risk** — settlement price relies on external oracle feeds
6. **Collateral/routing risk** — investments rely on pUSD wrapping, CLOB V2,
   relayer behavior, and bridge/swap paths.
7. **Auto-roll risk** — World Cup rolling may pause with locked cash when the
   next stage is not yet open, a roll fails (`rollRetry`), or the user must
   manually `retry-roll` / `stop-rolling` to release funds.
8. **Commodity-specific risk** — Oil uses CL=F/MEXC futures-oriented sources;
   Gold and Silver use spot oracle feeds. Price source differences may affect
   settlement outcomes vs spot expectations.
