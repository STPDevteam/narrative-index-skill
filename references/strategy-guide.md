# Narrative Index — Strategy Guide

## Contents

- [Product Concept](#product-concept) — what the platform does
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

Narrative Index Vault creates BTC directional index products using Polymarket
prediction markets. Users gain leveraged exposure to BTC price movements by
purchasing baskets of binary outcome contracts that settle monthly.

---

## Strategy Types

### BTC Bullish Index

- Buys **YES** on multiple "Will BTC hit $X?" contracts (UP direction)
- Profits when BTC breaks through strike prices upward
- More strikes breached = higher return (leverage effect)

### BTC Bearish Index

- Buys **YES** on multiple "Will BTC drop below $X?" contracts (DOWN direction)
- Profits when BTC falls through strike prices downward
- Can be used to hedge existing BTC spot positions

---

## Polymarket Market Structure

```
Event (monthly)
  └─ "What price will Bitcoin hit in March 2026?"
      ├─ Market: "Will BTC hit $85,000?"   → conditionId → tokenId(YES), tokenId(NO)
      ├─ Market: "Will BTC hit $90,000?"   → conditionId → tokenId(YES), tokenId(NO)
      ├─ Market: "Will BTC hit $95,000?"   → conditionId → tokenId(YES), tokenId(NO)
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

- All contracts settle on the **last day of the month at UTC 23:59**
- Settlement is binary: YES pays $1.00, NO pays $0.00
- Returns depend on how many strikes were breached by settlement time

---

## Performance Scenarios

### Bullish Index

| Scenario | BTC Move | Index Return | Notes |
|----------|----------|--------------|-------|
| Sustained rally | +30% | ~+42% | Multiple strikes breached |
| Spike & pullback | +8% EOM | ~+18% | Early breaches lock in gains |
| Sideways | ~0% | ~-5% | Time value decay |
| Crash | -20% | ~-80% | All YES contracts expire worthless |

### Bearish Index

| Scenario | BTC Move | Index Return | Notes |
|----------|----------|--------------|-------|
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
