# Narrative Index — Agent Skill

An AI Agent Skill for operating the [Narrative Index Vault](https://polyvaults.ai) platform — a custodial multi-asset directional index product built on Polymarket prediction markets.

## What it does

This Skill enables AI agents (Claude, Cursor, etc.) to autonomously manage the full investment lifecycle across multiple assets (BTC, ETH, SOL, Oil, Gold, Silver):

- **Asset discovery** — browse available assets, check market status and liquidity
- **Wallet management** — create Safe wallets, check balances, get deposit addresses
- **Index investing** — preview allocations, execute Bullish/Bearish index orders for any asset
- **Portfolio monitoring** — NAV, PnL, total return, daily performance data, per-asset breakdown
- **Chart data** — hourly price data with strike lines for any supported asset
- **Withdrawals** — withdraw to Polygon or cross-chain (ETH, Arbitrum, Base, Optimism, BSC, Solana)
- **Early redemption** — market-sell active positions before settlement with 2% profit fee
- **Auto-redemption** — resolved markets are automatically redeemed every hour

## Installation

### npx skills add (Recommended)

Install from GitHub using the [skills CLI](https://www.npmjs.com/package/skills):

```bash
npx skills add STPDevteam/narrative-index-skill
```

Install globally (available across all projects):

```bash
npx skills add STPDevteam/narrative-index-skill -g
```

Install to a specific agent:

```bash
npx skills add STPDevteam/narrative-index-skill -a cursor
npx skills add STPDevteam/narrative-index-skill -a claude-code
```

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
| ETH | Ethereum | Crypto | Coming Soon |
| SOL | Solana | Crypto | Coming Soon |
| OIL | Crude Oil | Energy | Active |
| GOLD | Gold | Metals | Coming Soon |
| SILVER | Silver | Metals | Coming Soon |

## Available Tools

| Tool | Endpoint | Description |
|------|----------|-------------|
| `connect_wallet` | `POST /auth/connect` | Register/login, returns userId and Safe address |
| `get_wallet_balance` | `GET /wallets/:userId/balance` | Query USDC.e + native USDC balance |
| `get_deposit_address` | `GET /wallets/:userId/deposit-address` | Get deposit address (accepts USDC & USDC.e) |
| `get_assets` | `GET /assets` | List all registered assets with status |
| `get_market_status` | `GET /market/status` | Check market availability (single or all assets) |
| `preview_index` | `POST /index/preview` | Preview strike allocations (supports `asset` param) |
| `invest_index` | `POST /index/invest` | Execute index investment (supports `asset` param) |
| `get_positions` | `GET /index/positions/:userId` | View index positions (includes asset info) |
| `get_portfolio` | `GET /portfolio?userId=` | NAV/PnL/totalReturn dashboard (supports `asset` filter) |
| `get_portfolio_breakdown` | `GET /portfolio/breakdown` | Per-direction metrics (supports `asset` filter) |
| `get_returns` | `GET /performance/returns?month=` | Monthly daily return data (supports `asset` param) |
| `get_chart` | `GET /chart/strikes` | Asset price + strike lines chart data (supports `asset` param) |
| `withdraw` | `POST /wallets/withdraw` | Withdraw to Polygon or cross-chain (7 chains) |
| `withdraw_quote` | `POST /wallets/withdraw-quote` | Preview cross-chain fees and ETA |
| `withdraw_status` | `GET /wallets/withdraw-status/:addr` | Track cross-chain withdrawal progress |
| `supported_chains` | `GET /wallets/supported-chains` | List supported withdrawal chains |
| `get_accounting_positions` | `GET /accounting/:userId/positions` | Positions with unrealized PnL |
| `get_trades` | `GET /accounting/:userId/trades` | Trade history |
| `get_pnl` | `GET /accounting/:userId/pnl` | P&L report |
| `early_redeem` | `POST /index/redeem` | Market-sell positions (requires signature) |

## API Base URL

```
https://api.polyvaults.ai
```

## License

MIT
