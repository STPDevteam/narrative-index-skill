# Polyvaults Index — Agent Skill

An AI Agent Skill for operating [Polyvaults](https://polyvaults.ai) — a custodial Polymarket index platform covering asset-direction indices, NarrativeBasket products such as TACO, and World Cup 2026 bracket products.

## What it does

This Skill enables AI agents (Claude, Cursor, etc.) to autonomously manage the full investment lifecycle across Polyvaults products:

- **Asset discovery** — browse available assets, check market status and liquidity
- **Wallet management** — challenge-based wallet connect, deposit addresses, pUSD/USDC balances, locked and withdrawable funds
- **Index investing** — preview allocations and execute Bullish/Bearish asset-direction products
- **Product investing** — operate `/products/:productKey/*` for TACO, World Cup, and INDEX products
- **Portfolio monitoring** — NAV, PnL, total return, daily performance data, per-asset and per-product breakdowns
- **World Cup positions** — auto-roll chains, locks, retry/stop-rolling via signed endpoints
- **Referral rewards** — permanent referral codes, fee split, reward dashboard
- **Chart data** — hourly price data with strike lines for any supported asset
- **Withdrawals** — withdraw to Polygon or cross-chain (ETH, Arbitrum, Base, Optimism, BSC, Solana)
- **Early redemption** — market-sell active positions before settlement with 5% profit fee on positive profit
- **Auto-redemption / auto-roll** — resolved markets are redeemed to pUSD; TACO and World Cup products can auto-compound/roll when enabled
- **Multi-chain signatures** — EIP-712 V2 with `signatureChainId` + `fundsChainId` for Base Smart Wallet support
- **Remote signing awareness** — account for Deposit Wallet vs Safe signing, CLOB auth, and signing-service policy enforcement when debugging orders

## Installation

### bunx skills add (Recommended)

Install from GitHub using the [skills CLI](https://www.npmjs.com/package/skills):

```bash
bunx skills add STPDevteam/narrative-index-skill
```

Install globally (available across all projects):

```bash
bunx skills add STPDevteam/narrative-index-skill -g
```

Install to a specific agent:

```bash
bunx skills add STPDevteam/narrative-index-skill -a cursor
bunx skills add STPDevteam/narrative-index-skill -a claude-code
```

If your environment does not use Bun, `npx skills add ...` remains equivalent.

### ClawHub

Install from the [ClawHub](https://clawhub.com) skill registry:

```bash
clawhub install narrative-index-skill
```

Or search first:

```bash
clawhub search narrative-index
clawhub info narrative-index-skill
clawhub install narrative-index-skill
```

### Manual Installation

Clone and copy into your agent's skills directory:

```bash
git clone https://github.com/STPDevteam/narrative-index-skill.git

# For Cursor (project-scoped, shared with team)
cp -r narrative-index-skill .agents/skills/narrative-index-skill

# For Cursor (global)
cp -r narrative-index-skill ~/.cursor/skills/narrative-index-skill

# For Claude Code (project-scoped)
cp -r narrative-index-skill .claude/skills/narrative-index-skill

# For Claude Code (global)
cp -r narrative-index-skill ~/.claude/skills/narrative-index-skill
```

### Claude API

Upload via the `/v1/skills` endpoint. See the [Skills API documentation](https://platform.claude.com/docs/en/build-with-claude/skills-guide).

### claude.ai

1. Download this repo as a zip
2. Go to Settings > Features
3. Upload the zip file

## File Structure

```
├── SKILL.md                     # Main skill file (loaded when triggered)
├── README.md                    # This file
└── references/
    ├── api-reference.md         # Complete API endpoint reference
    └── strategy-guide.md        # Strategy concepts and market mechanics
```

## Supported Assets

| Symbol | Name | Category | Status |
|--------|------|----------|--------|
| BTC | Bitcoin | Crypto | Active |
| ETH | Ethereum | Crypto | Active |
| SOL | Solana | Crypto | Coming Soon |
| OIL | Crude Oil | Energy | Active |
| GOLD | Gold | Metals | Coming Soon |
| SILVER | Silver | Metals | Coming Soon |

## Current Product Families

| Family | productKind | Examples |
|--------|-------------|----------|
| Asset-direction index | `INDEX` | `btc-bullish`, `btc-bearish`, `eth-bullish`, `oil-bearish` |
| NarrativeBasket | `MANAGED` | `narrative-basket:taco-v1` |
| World Cup 2026 bracket | `MANAGED` | `worldcup-2026:europe:r48-32`, `worldcup-2026:custom:r48-32` |

## Available Tools

| Tool | Endpoint | Description |
|------|----------|-------------|
| `connect_wallet` | `GET /auth/challenge`, `POST /auth/connect` | Register/login; returns `sessionToken`, wallet readiness flags |
| `get_wallet_info` | `GET /wallets/:userId` | Wallet type, `isDeployed`, `isApproved` |
| `get_wallet_balance` | `GET /wallets/:userId/balance` | Query USDC.e + native USDC + pUSD + locked/withdrawable balances |
| `get_deposit_address` | `GET /wallets/:userId/deposit-address` | Get deposit address (accepts USDC & USDC.e) |
| `get_assets` | `GET /assets` | List all registered assets with status |
| `get_market_status` | `GET /market/status` | Check market availability (single or all assets) |
| `preview_index` | `POST /index/preview` | Legacy asset-direction preview |
| `invest_index` | `POST /index/invest` | Legacy asset-direction investment |
| `get_positions` | `GET /index/positions/:userId` | View index positions (includes asset info) |
| `get_portfolio` | `GET /portfolio?userId=` | NAV/PnL/totalReturn dashboard (supports `asset` filter) |
| `get_portfolio_breakdown` | `GET /portfolio/breakdown` | Per-direction metrics (supports `asset` filter) |
| `list_products` | `GET /products` | List INDEX and MANAGED products |
| `get_product_definition` | `GET /products/:productKey/definition` | Product metadata and managed basket definitions |
| `get_product_health` | `GET /products/:productKey/health` | Tradability and per-strike health |
| `preview_product` | `POST /products/:productKey/preview` | Recommended product preview endpoint |
| `invest_product` | `POST /products/:productKey/invest` | Product investment (signs `autoCompound`) |
| `redeem_product` | `POST /products/:productKey/redeem` | Product-level early exit (may fallback to stop-rolling) |
| `get_product_portfolios` | `GET /portfolio/products` | Product-level holdings list |
| `get_product_portfolio` | `GET /portfolio/products/:productKey` | Single product portfolio detail |
| `stop_rolling` | `POST /products/:productKey/stop-rolling` | Release World Cup auto-roll lock (requires signature + `rootDepositId`) |
| `retry_roll` | `POST /products/:productKey/retry-roll` | Retry failed World Cup auto-roll |
| `get_worldcup_positions` | `GET /products/worldcup-2026/positions` | World Cup chain status, locks, roll retry |
| `get_worldcup_dashboard_groups` | `GET /portfolio/worldcup-2026/dashboard-groups` | Variant-level World Cup portfolio |
| `get_worldcup_potential_return` | `POST /products/:productKey/potential-return` | World Cup multi-round return estimate |
| `get_worldcup_catalog` | `GET /products/worldcup-2026/catalog` | World Cup stages, teams, presets, exit stages |
| `get_referral` | `GET /referral/:userId` | Permanent referral code and rewards |
| `get_returns` | `GET /performance/returns?month=` | Monthly daily return data (supports `asset` param) |
| `get_chart` | `GET /chart/strikes` | Asset price + strike lines chart data (supports `asset` param) |
| `withdraw` | `POST /wallets/withdraw` | Withdraw to Polygon or cross-chain (signs token + chain) |
| `withdraw_quote` | `POST /wallets/withdraw-quote` | Preview cross-chain fees and ETA |
| `withdraw_status` | `GET /wallets/withdraw-status/:addr` | Track cross-chain withdrawal progress |
| `supported_chains` | `GET /wallets/supported-chains` | List supported withdrawal chains |
| `early_redeem` | `POST /index/redeem` | Legacy asset+direction market-sell (requires signature) |

## Security Notes

- The main API server does not hold AWS KMS decrypt permission or plaintext owner keys.
- All operational owner EOA signatures go through the isolated signing-service.
- New users normally use Polymarket Deposit Wallets (`POLY_1271`); legacy users may still use Safe (`POLY_GNOSIS_SAFE`).
- CLOB API-key auth is signed by the owner EOA; orders use `POLY_1271` for Deposit Wallets.
- EIP-712 V2: `domain.chainId` = wallet chain, `fundsChainId: 137` in message. Legacy V1 still accepted.
- `autoCompound`, `rootDepositId` (stop-rolling/retry-roll), and withdraw `token`/`chain` are signed fields.
- Product investments no longer enforce per-user/platform active-position caps.

## API Base URL

```
https://api.polyvaults.ai
```

## License

MIT
