# Narrative Index — Strategy Guide

## Contents

- [Product Concept](#product-concept) — what the platform does
- [Supported Assets](#supported-assets) — multi-asset coverage
- [Strategy Types](#strategy-types) — Bullish vs Bearish index
- [Polymarket Market Structure](#polymarket-market-structure) — event, market, token hierarchy
- [Weight Calculation](#weight-calculation) — allocation algorithm
- [Iterative Pruning](#iterative-pruning) — minimum order constraints
- [Order Execution](#order-execution) — FAK gasless orders
- [Settlement](#settlement) — monthly binary settlement
- [Performance Scenarios](#performance-scenarios) — expected returns by scenario
- [Risks](#risks) — key risk factors

---

## Product Concept

Narrative Index Vault creates multi-asset directional index products using
Polymarket prediction markets. Users gain leveraged exposure to price movements
of various assets — crypto, commodities, and metals — by purchasing baskets of
binary outcome contracts that settle monthly.

---

## Supported Assets

The platform supports multiple underlying assets across three categories:

| Category | Assets | Price Source |
|----------|--------|-------------|
| **Crypto** | BTC, ETH, SOL | Binance spot price |
| **Energy** | Crude Oil (WTI) | Pyth Network (rolling futures) |
| **Metals** | Gold, Silver | Pyth Network (spot) |

Each asset has independent Polymarket events with their own strike prices.
New assets can be added via the asset registry configuration.

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

Funds are distributed across qualifying strikes using a proprietary
weighting algorithm based on market liquidity. The algorithm ensures
diversified allocation — no single strike dominates the portfolio.

---

## Iterative Pruning

After initial allocation, strikes that do not meet minimums are removed
one at a time (worst-deficit first), and weights are recalculated:

1. Each strike must have allocation >= **$1.00** (Polymarket minimum order)
2. Each strike must buy >= **5 shares** (Polymarket minimum share count)
3. Price must be within **$0.01 – $0.99**

The pruning loop removes the single worst-deficit strike per iteration until
all remaining strikes satisfy these constraints.

---

## Order Execution

- Order type: **FAK** (Fill-and-Kill) — immediate partial/full fill, remainder cancelled
- Execution: **Gasless** via Polymarket Builder Program relayer
- Each strike is an independent order; one failure does not block others

---

## Settlement

- All contracts settle monthly. Polymarket uses **Eastern Time (ET)** for
  market creation and settlement timing.
- Settlement is binary: YES pays $1.00, NO pays $0.00
- Returns depend on how many strikes were breached by settlement time
- Markets for different assets may be created on different days of the month
  (typically the 1st–3rd). The platform automatically handles late market
  creation.

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
6. **Commodity-specific risk** — Oil uses rolling futures contracts; Gold and
   Silver use spot oracle feeds. Price source differences may affect
   settlement outcomes vs spot expectations.
