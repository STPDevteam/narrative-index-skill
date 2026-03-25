# Narrative Index — Agent Skill

An AI Agent Skill for operating the [Narrative Index Vault](https://polyvaults.ai) platform — a custodial BTC directional index product built on Polymarket prediction markets.

## What it does

This Skill enables AI agents (Claude, Cursor, etc.) to autonomously manage the full investment lifecycle:

- **Wallet management** — create Safe wallets, check balances, get deposit addresses
- **Index investing** — preview allocations, execute BTC Bullish/Bearish index orders
- **Portfolio monitoring** — NAV, PnL, total return, daily performance data
- **Withdrawals** — withdraw to Polygon or cross-chain (ETH, Arbitrum, Base, Optimism, BSC, Solana)

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

## Available Tools

| Tool | Endpoint | Description |
|------|----------|-------------|
| `connect_wallet` | `POST /auth/connect` | Register/login, returns userId and Safe address |
| `get_wallet_balance` | `GET /wallets/:userId/balance` | Query USDC.e + native USDC balance |
| `get_deposit_address` | `GET /wallets/:userId/deposit-address` | Get deposit address (accepts USDC & USDC.e) |
| `preview_index` | `POST /index/preview` | Preview strike allocations (optional userId for swap fee) |
| `invest_index` | `POST /index/invest` | Execute index investment (auto-swaps USDC if needed) |
| `get_positions` | `GET /index/positions/:userId` | View index positions |
| `get_portfolio` | `GET /portfolio?userId=` | NAV/PnL/totalReturn dashboard |
| `get_returns` | `GET /performance/returns?month=` | Monthly daily return data |
| `withdraw` | `POST /wallets/withdraw` | Withdraw to Polygon or cross-chain (7 chains) |
| `withdraw_quote` | `POST /wallets/withdraw-quote` | Preview cross-chain fees and ETA |
| `withdraw_status` | `GET /wallets/withdraw-status/:addr` | Track cross-chain withdrawal progress |
| `supported_chains` | `GET /wallets/supported-chains` | List supported withdrawal chains |
| `get_btc_chart` | `GET /chart/btc-strikes` | BTC price + strike lines chart data |
| `get_accounting_positions` | `GET /accounting/:userId/positions` | Positions with unrealized PnL |
| `get_trades` | `GET /accounting/:userId/trades` | Trade history |
| `get_pnl` | `GET /accounting/:userId/pnl` | P&L report |

## API Base URL

```
https://api.polyvaults.ai
```

## License

MIT
