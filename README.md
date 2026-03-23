# Narrative Index — Agent Skill

An AI Agent Skill for operating the [Narrative Index Vault](https://polyvaults.ai) platform — a custodial BTC directional index product built on Polymarket prediction markets.

## What it does

This Skill enables AI agents (Claude, Cursor, etc.) to autonomously manage the full investment lifecycle:

- **Wallet management** — create Safe wallets, check balances, get deposit addresses
- **Index investing** — preview allocations, execute BTC Bullish/Bearish index orders
- **Portfolio monitoring** — NAV, PnL, total return, daily performance data
- **Withdrawals** — withdraw USDC.e to external wallets

## Installation

### Cursor IDE

Copy the skill directory into your project:

```bash
git clone git@github.com:STPDevteam/narrative-index-skill.git .cursor/skills/narrative-index
```

Or add as a user-level skill (available across all projects):

```bash
git clone git@github.com:STPDevteam/narrative-index-skill.git ~/.cursor/skills/narrative-index
```

### Claude Code

```bash
git clone git@github.com:STPDevteam/narrative-index-skill.git .claude/skills/narrative-index
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
| `get_wallet_balance` | `GET /wallets/:userId/balance` | Query USDC.e available balance |
| `get_deposit_address` | `GET /wallets/:userId/deposit-address` | Get deposit address |
| `preview_index` | `POST /index/preview` | Preview strike allocations |
| `invest_index` | `POST /index/invest` | Execute index investment |
| `get_positions` | `GET /index/positions/:userId` | View index positions |
| `get_portfolio` | `GET /portfolio?userId=` | NAV/PnL/totalReturn dashboard |
| `get_returns` | `GET /performance/returns?month=` | Monthly daily return data |
| `withdraw` | `POST /wallets/withdraw` | Withdraw from Safe to external address |

## API Base URL

```
https://api.polyvaults.ai
```

## License

MIT
