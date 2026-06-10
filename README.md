# Algodhan — Multi-Strategy Algorithmic Trading Platform

> Production-grade intraday trading system running 3 independent strategy bots on Indian equities (NSE). Built on Fyers API with real-time WebSocket data, per-strategy stock selection, and full trade lifecycle management.

**Live Instance**: smtrade.space | **Infra**: Oracle Cloud ARM (4 OCPU, 24GB) | **Market**: NSE (India) | **Timeframe**: 5-minute

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Data Flow](#2-data-flow)
3. [Trading Strategies](#3-trading-strategies)
4. [Stock Selection Engine](#4-stock-selection-engine)
5. [Trade Lifecycle](#5-trade-lifecycle)
6. [Risk Management](#6-risk-management)
7. [Infrastructure & Deployment](#7-infrastructure--deployment)
8. [API & Frontend](#8-api--frontend)
9. [Configuration Reference](#9-configuration-reference)
10. [Operations & Monitoring](#10-operations--monitoring)
11. [Security Model](#11-security-model)
12. [Scaling Considerations](#12-scaling-considerations)

---

## 1. System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OCI SERVER (smtrade-arm-prod)                         │
│                   Ubuntu 22.04 ARM | 4 OCPU | 24 GB RAM                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    TRADING ENGINE LAYER                               │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│  │  │  SCALPER BOT │  │   PRO BOT   │  │   V8 BOT    │                  │   │
│  │  │ main_scalper │  │  main_pro   │  │  main_v8    │                  │   │
│  │  │   .py        │  │   .py       │  │   .py       │                  │   │
│  │  │              │  │             │  │             │                  │   │
│  │  │ strategy_    │  │ strategy_   │  │ strategy_   │                  │   │
│  │  │ scalper_v1   │  │ pro         │  │ v8_fixed    │                  │   │
│  │  │              │  │             │  │             │                  │   │
│  │  │ ScalperSel.  │  │ ProSelector │  │ V8Selector  │                  │   │
│  │  │              │  │             │  │             │                  │   │
│  │  │ scalper_     │  │ pro_        │  │ v8_         │                  │   │
│  │  │ trades.db    │  │ trades.db   │  │ trades.db   │                  │   │
│  │  └──────┬───────┘  └──────┬──────┘  └──────┬──────┘                  │   │
│  │         │                  │                 │                         │   │
│  │         └──────────────────┼─────────────────┘                        │   │
│  │                            │                                           │   │
│  │  ┌─────────────────────────▼─────────────────────────────────────┐    │   │
│  │  │                    SHARED SERVICES                              │    │   │
│  │  │                                                                │    │   │
│  │  │  ┌──────────┐  ┌────────────┐  ┌───────────┐  ┌───────────┐  │    │   │
│  │  │  │ Fyers WS │  │ broker.py  │  │ notifier  │  │ state_mgr │  │    │   │
│  │  │  │ (ticks)  │  │ (REST API) │  │ (Telegram)│  │ (LTP/Reg) │  │    │   │
│  │  │  └──────────┘  └────────────┘  └───────────┘  └───────────┘  │    │   │
│  │  │                                                                │    │   │
│  │  │  ┌────────────────┐  ┌──────────────────┐                     │    │   │
│  │  │  │ stock_selector │  │ risk_manager.py  │                     │    │   │
│  │  │  │ (Fyers-based)  │  │ (position sizing)│                     │    │   │
│  │  │  └────────────────┘  └──────────────────┘                     │    │   │
│  │  └────────────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        WEB LAYER                                      │   │
│  │                                                                       │   │
│  │  Nginx (SSL/TLS) → FastAPI (:8000) → Next.js (:3000)                │   │
│  │                  → Streamlit (:8501, :8503)                          │   │
│  │                  → Admin API (:8001, VPN only)                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  SUPPORT SERVICES                                                    │   │
│  │  telegram_listener | copy_trader | bot_monitor (watchdog)            │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Network Diagram

```
                    ┌──────────────────┐
                    │   Fyers Servers   │
                    │  (api-t1.fyers.in)│
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │ WebSocket    │ REST API      │
              │ (real-time   │ (candles,     │
              │  LTP ticks)  │  quotes,      │
              │              │  orders)      │
              └──────┬───────┴──────┬───────┘
                     │              │
          ┌──────────▼──────────────▼──────────┐
          │         OCI SERVER                  │
          │         80.225.195.168              │
          │                                    │
          │  ┌──────────────────────────────┐  │
          │  │  3 Bot Processes (systemd)   │  │
          │  │  Each: WS + Main Loop + DB   │  │
          │  └──────────────┬───────────────┘  │
          │                 │                   │
          │  ┌──────────────▼───────────────┐  │
          │  │  Telegram Bot API            │──┼──→ User's Phone
          │  │  (alerts, token refresh)     │  │
          │  └──────────────────────────────┘  │
          │                                    │
          │  ┌──────────────────────────────┐  │
          │  │  Nginx (:443) + Let's Encrypt│──┼──→ smtrade.space
          │  │  → FastAPI → Next.js         │  │    (public web)
          │  └──────────────────────────────┘  │
          └────────────────────────────────────┘
```

---

## 2. Data Flow

### Morning Sequence (09:00 – 09:25)

```
09:00  Bot starts → connects Fyers WebSocket → subscribes to 9 static symbols
09:15  Market opens → WebSocket begins receiving LTP ticks
09:16  SCANNER FIRES (once per day):
       ├── Fyers get_quotes() → batch 50 symbols → LTP + prev_close
       ├── For qualifying stocks: get_candles(symbol, "D", 15 days)
       ├── Compute: ATR%, ADX, RSI, EMA alignment, volume ratio, gap%
       ├── Strategy-specific scorer ranks all stocks
       ├── Top 8 selected → WATCHLIST updated
       ├── WebSocket RESTARTED with new symbols
       └── Telegram: "🔥 [STRATEGY] Watchlist Selected"
09:16  Regime Detection:
       ├── Fetch Nifty50 index candles
       ├── Gap analysis → TRENDING / CHOPPY / NORMAL
       └── Adjust target R:R based on regime
09:25  ENTRY WINDOW OPENS — strategy signal generation begins
```

### Intraday Loop (09:25 – 15:25)

```
Every 1 second:
  ├── Read LTP from WebSocket (thread-safe LTPState)
  ├── manage_active_trades() → check SL/TP/TSL/BE for open positions
  └── update_live_index_data()

Every 5 seconds:
  └── update_dashboard_data() → JSON files for frontend

Every 5 minutes:
  ├── fetch_candles() for each watchlist symbol (10-day 5min history)
  ├── Inject real-time LTP into latest bar
  ├── get_combined_signal(df, index_row) → {signal, entry, sl, target}
  ├── If signal ≠ 0:
  │     ├── can_trade() → daily limits + loss limit check
  │     ├── calculate_quantity(entry, sl) → position size
  │     ├── broker.place_order() → execute (or paper trade)
  │     ├── log_trade() → SQLite
  │     ├── copy_trader.copy_entry() → mirror to clients
  │     └── Telegram alert
  └── Store ATR in LTP_DICT for TSL computation

15:25  SQUARE OFF:
  ├── Exit all open positions at market
  ├── daily_summary() → Telegram + Email
  └── Bot enters standby until next day
```

---

## 3. Trading Strategies

### Strategy Comparison

| Attribute | Scalper V1 | Pro V15 | V8 Fixed |
|-----------|-----------|---------|----------|
| **Style** | Adaptive Multi-Target | VWAP Pullback | Score-Based Trend |
| **Entry Types** | Pullback, VWAP Bounce, Breakout | VWAP + Pivot + MACD + EMA15 | Pullback, EMA Cross, VWAP, Breakout |
| **ADX Gate** | 20 AM / 26 PM | 15 (low bar) | 20 AM / 30 PM |
| **SL** | 1.3× ATR (tight) | 1.2× ATR (tightest) | 1.8× ATR (wide) |
| **Target System** | TP1=1R, TP2=2R, Trail 50% | Fixed 2.5R + partial 1R | Stepped: 1.5R → +1R per step |
| **Max Trades/Day** | 20 | 15 | 12 |
| **Risk/Trade** | 1.0% | 1.5% | 1.5% |
| **Ideal Market** | High ATR + volume spikes | Clean VWAP structure | Strong trends + gaps |
| **Index Filter** | Soft (0.4% deviation block) | Strict (VWAP alignment) | Dynamic (strict if ADX<25) |

### Signal Generation Pipeline

```
df (5-min OHLCV) + LTP injection
        │
        ▼
┌───────────────────────┐
│  Indicator Computation│
│  EMA9/21/50, VWAP,   │
│  ADX, RSI, ATR,      │
│  Volume SMA           │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Trend Score Engine   │
│  5 conditions:        │
│  EMA9>21, EMA21>50,   │
│  Price>VWAP, DI+>DI-, │
│  RSI momentum          │
│  Need 3/5 for trend   │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Entry Detection      │
│  Priority:            │
│  1. Pullback to EMA   │
│  2. VWAP Bounce       │
│  3. Momentum Breakout │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Risk Computation     │
│  SL = ATR × mult     │
│  TP = Entry ± Risk×RR│
│  Qty = (Capital×Risk%)│
│        / Risk/share   │
└───────────────────────┘
```

---

## 4. Stock Selection Engine

### Architecture

```
stock_selector/
├── base_selector.py        ← ABC: Fyers data fetching + metric computation
├── selector_registry.py    ← Maps "scalper"/"pro"/"v8" → selector instance
├── selector_scalper.py     ← ScalperSelector: high ATR, volume spikes
├── selector_pro.py         ← ProSelector: institutional volume, small gaps
├── selector_v8.py          ← V8Selector: trending ADX, wide range, momentum
└── scan_database.py        ← SQLite: daily_scans, stock_daily_ohlc, performance
```

### Selection Criteria per Strategy

| Metric | Scalper | Pro | V8 |
|--------|---------|-----|-----|
| Min ATR% | 0.8% | 0.6% | 1.0% |
| Max ATR% | 5.0% | 4.0% | 6.0% |
| Min Volume | 500K | 300K | 200K |
| Min ADX | 18 | 15 | 18 |
| Volume Ratio Weight | ×1.5 | ×2.0 (key factor) | ×1.0 |
| Gap Penalty | None | Yes (>2% blocked) | Bonus (gaps = trend day) |
| Blacklist | None | None | AXISBANK, INFY, KOTAKBANK, SBIN |

### Data Source Priority

```
1. Fyers get_quotes() → batch LTP + prev_close (50 symbols/call)
2. Fyers get_candles(symbol, "D", 15 days) → daily OHLCV for indicators
3. FALLBACK: yfinance (if Fyers connection fails)
```

---

## 5. Trade Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRADE STATE MACHINE                           │
│                                                                  │
│  ┌──────┐    signal    ┌──────┐   BE hit   ┌─────────┐         │
│  │ SCAN ├────────────→│ OPEN ├───────────→│ BE_MOVED │         │
│  └──────┘             └──┬───┘            └────┬─────┘         │
│                          │                      │                │
│                    SL hit│              PT hit   │  TSL step      │
│                          ▼                      ▼                │
│                    ┌──────────┐          ┌──────────┐           │
│                    │ CLOSED   │          │ PARTIAL  │           │
│                    │ (Loss)   │          │ (25% out)│           │
│                    └──────────┘          └────┬─────┘           │
│                                               │                  │
│                                     Target hit│  or TSL hit      │
│                                               ▼                  │
│                                         ┌──────────┐            │
│                                         │ CLOSED   │            │
│                                         │ (Profit) │            │
│                                         └──────────┘            │
│                                                                  │
│  At 15:25 → ALL OPEN → CLOSED (EOD squareoff)                  │
└─────────────────────────────────────────────────────────────────┘
```

### Exit Mechanisms

| Mechanism | Trigger | Action |
|-----------|---------|--------|
| **Hard SL** | LTP ≤ SL (long) or LTP ≥ SL (short) | Close 100%, log loss |
| **Break-Even** | LTP reaches entry + 0.5R | Move SL to entry + 0.2% (fee cover) |
| **Partial Target** | LTP reaches 1R profit | Exit 25%, move SL to entry |
| **Stepped Target** | LTP reaches target | Lock SL at step-0.5R, extend target by +1R |
| **TSL (ATR-based)** | Price moves favorably | Trail SL by 0.5×ATR per step |
| **EOD Squareoff** | Time = 15:25 IST | Close all at market price |
| **Emergency Kill** | `kill_signal.txt` = "KILL" | Close all immediately |

---

## 6. Risk Management

### Position Sizing Formula

```
Risk Amount = Capital × Risk% per trade
            = Rs 10,000 × 1.5% = Rs 150

Quantity = Risk Amount / Risk per Share
         = Rs 150 / (Entry - SL)
         = Rs 150 / (₹2500 - ₹2468) = 4 shares
```

### Daily Safeguards

| Guard | Threshold | Action |
|-------|-----------|--------|
| Max trades/day | 12-20 (per strategy) | Block new entries |
| Max simultaneous | 3-4 | Block new entries |
| Daily loss limit | -2% of capital | Block all trading |
| Re-entry cooldown | 5-10 minutes per symbol | Skip symbol |
| Regime SIT_OUT | Choppy market detected | Stop all scanning |
| Profit Vault | +15% growth | Auto-bank 10% of capital |

### Database Schema (per bot)

```sql
trades (
    id, date, time, symbol, action, quantity,
    entry, stoploss, target, exit_price, pnl,
    reason, status, pt_level, be_level,
    is_partial, is_be, max_price, initial_sl, steps
)
```

---

## 7. Infrastructure & Deployment

### Server Specifications

| Component | Specification |
|-----------|--------------|
| Instance | Oracle Cloud ARM VM.Standard.A1.Flex |
| CPU | 4 OCPUs (ARM Ampere A1) |
| RAM | 24 GB |
| OS | Ubuntu 22.04 LTS (aarch64) |
| Storage | 50 GB boot volume |
| Network | Public IPv4: 80.225.195.168 |
| Domain | smtrade.space (Cloudflare DNS) |
| SSL | Let's Encrypt (auto-renew) |
| Timezone | Asia/Kolkata (IST) |

### Service Architecture

```
systemd services (9 total):
├── algodhan_scalper.service  → main_scalper.py    (trading)
├── algodhan_pro.service      → main_pro.py        (trading)
├── algodhan_v8.service       → main_v8.py         (trading)
├── api_server.service        → uvicorn :8000      (API)
├── frontend.service          → next start :3000   (web UI)
├── dashboard.service         → streamlit :8503    (control)
├── public_dashboard.service  → streamlit :8501    (public)
├── telegram_listener.service → telegram bot       (commands)
└── watchdog.service          → bot_monitor.py     (health)
```

### Deployment Commands

```bash
# From Windows (push code to server):
python scripts/push_to_server.py --server 80.225.195.168

# On server (deploy 3 bots):
bash scripts/deploy_3bots.sh

# Monitor:
journalctl -u algodhan_scalper -f
journalctl -u algodhan_pro -f
journalctl -u algodhan_v8 -f

# Emergency stop (all bots):
sudo systemctl stop algodhan_scalper algodhan_pro algodhan_v8

# Per-bot kill (without service restart):
echo "KILL" > logs/scalper/kill_signal.txt
```

### Multi-Bot Isolation

Each bot runs as a separate Linux process with isolated:

| Component | Scalper | Pro | V8 |
|-----------|---------|-----|-----|
| Entry point | `main_scalper.py` | `main_pro.py` | `main_v8.py` |
| Trade DB | `logs/scalper/scalper_trades.db` | `logs/pro/pro_trades.db` | `logs/v8/v8_trades.db` |
| Log file | `logs/scalper/scalper_bot.log` | `logs/pro/pro_bot.log` | `logs/v8/v8_bot.log` |
| State file | `logs/scalper/system_state.json` | `logs/pro/system_state.json` | `logs/v8/system_state.json` |
| Kill signal | `logs/scalper/kill_signal.txt` | `logs/pro/kill_signal.txt` | `logs/v8/kill_signal.txt` |
| Telegram tag | ⚡ [SCALPER] | 🏛️ [PRO-V15] | 🛡️ [V8-TREND] |
| Shared | Fyers credentials (.env), LTP WebSocket state, stock_selector.db |

---

## 8. API & Frontend

### API Routes (FastAPI on :8000)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/auth/signup` | POST | User registration |
| `/api/auth/google-complete` | POST | Google OAuth |
| `/api/auth/fyers/callback` | GET | Fyers OAuth token exchange |
| `/api/trading/positions` | GET | Current open positions |
| `/api/trading/kill` | POST | Emergency stop |
| `/api/backtest/run` | POST | Run strategy backtest |
| `/api/public/leaderboard` | GET | Public performance data |
| `/api/reports/daily` | GET | Daily trade report |
| `/api/ws` | WebSocket | Real-time dashboard data |

### Frontend (Next.js 14)

| Page | Route | Function |
|------|-------|----------|
| Landing | `/` | Public SaaS landing page |
| Login | `/login` | JWT + Google OAuth |
| Console | `/secure/console` | Trading dashboard |
| Analytics | `/secure/analytics` | Performance charts |
| Backtest | `/secure/backtest` | Strategy backtesting |
| Risk Desk | `/secure/risk` | Risk monitoring |
| Admin | `/admin` | User management |

### Nginx Security

- HTTPS only (HTTP → 301 redirect)
- HSTS headers (1 year)
- CSP (Content Security Policy)
- Rate limiting: 30 req/s API, 10 req/s auth
- Sensitive file blocking (`.env`, `.db`, `.log`, `.py`)
- Admin panel on VPN-only interface (10.8.0.1:8443)

---

## 9. Configuration Reference

### Environment Variables (.env)

| Variable | Purpose |
|----------|---------|
| `FYERS_CLIENT_ID` | Fyers API app ID |
| `FYERS_ACCESS_TOKEN` | Daily OAuth token (refreshed via Telegram) |
| `FYERS_SECRET_KEY` | App secret for token generation |
| `FYERS_TOTP_KEY` | 2FA TOTP secret for headless login |
| `TELEGRAM_BOT_TOKEN` | Bot token for notifications |
| `TELEGRAM_CHAT_ID` | Target chat for alerts |
| `JWT_SECRET` | Authentication signing key |
| `SMARTTRADE_ENCRYPTION_KEY` | Fernet key for stored tokens |

### Strategy Config (config.py)

```python
STRATEGY = {
    "dynamic_watchlist": True,     # Enable per-strategy stock scanner
    "scalper_mode": False,         # Use scalper selector
    "pro_mode": True,              # Use strategy_pro.py
    "risk_per_trade_pct": 1.5,     # % of capital risked per trade
    "max_trades_per_day": 15,      # Daily trade cap
    "max_simultaneous_trades": 3,  # Max concurrent positions
    "sl_atr_multiplier": 1.2,      # Stop loss = ATR × this
    "final_target_rr": 2.5,        # Target = Risk × this
    "tsl_atr_based": True,         # ATR-based trailing stop
    "tsl_atr_factor": 0.5,         # Trail step = 0.5 × ATR
    "stepped_target_enabled": True, # Extend target on each hit
}
```

---

## 10. Operations & Monitoring

### Daily Checklist

| Time | Action | Method |
|------|--------|--------|
| Before 09:00 | Refresh Fyers token | Telegram bot (paste redirect URL) |
| 09:15 | Verify all 3 bots are active | `systemctl is-active algodhan_*` |
| 09:16 | Check scanner fired | Telegram: "🔥 Watchlist Selected" |
| During market | Monitor via Telegram | Auto-alerts on signals/trades |
| 15:30 | Check daily summary | Telegram + Email report |
| Anytime | Emergency stop | `echo "KILL" > logs/{bot}/kill_signal.txt` |

### Log Locations

```
/home/ubuntu/algodhan/logs/
├── scalper/
│   ├── scalper_bot.log          # Full bot log (rotated daily)
│   ├── scalper_startup.log      # systemd stdout/stderr
│   ├── scalper_trades.db        # SQLite trade database
│   ├── system_state.json        # Dashboard state
│   └── kill_signal.txt          # Emergency stop trigger
├── pro/
│   ├── pro_bot.log
│   ├── pro_trades.db
│   └── ...
├── v8/
│   ├── v8_bot.log
│   ├── v8_trades.db
│   └── ...
├── stock_selector.db            # Shared scan history
├── live_watchlist.json          # Current symbols + LTP
└── api_server.log               # API server logs
```

### Health Checks

```bash
# All bots running?
for s in algodhan_scalper algodhan_pro algodhan_v8; do
  echo "$s: $(systemctl is-active $s)"
done

# WebSocket receiving ticks?
grep "Heartbeat" logs/scalper/scalper_bot.log | tail -1

# Today's trades?
sqlite3 logs/scalper/scalper_trades.db "SELECT * FROM trades WHERE date=date('now')"

# Scanner selections today?
sqlite3 logs/stock_selector.db "SELECT strategy, symbol, score FROM daily_scans WHERE scan_date=date('now') ORDER BY strategy, rank"
```

---

## 11. Security Model

| Layer | Mechanism |
|-------|-----------|
| Network | OCI Security Lists (ports 22, 80, 443 only) |
| Transport | TLS 1.2+ with Let's Encrypt |
| Authentication | JWT + Google OAuth + Fyers OAuth |
| Admin Access | VPN-only (OpenVPN, 10.8.0.1:8443) |
| Token Storage | Fernet encryption (AES-128-CBC) |
| Secrets | `.env` file (never committed, SCP'd separately) |
| Rate Limiting | Nginx: 30 req/s API, 10 req/s auth |
| Headers | HSTS, CSP, X-Frame-Options DENY, nosniff |
| File Access | Nginx blocks .env, .db, .log, .py, .json |
| Bot Security | Token expiry detection → auto-pause + alert |

---

## 12. Scaling Considerations

### Current Capacity

| Resource | Usage | Headroom |
|----------|-------|----------|
| CPU (4 OCPU) | ~15% (3 bots + API) | Can handle 6-8 bots |
| RAM (24 GB) | ~4 GB | 20 GB available |
| WebSocket | 3 connections × ~15 symbols | Fyers allows 100+ |
| API calls | ~50 quotes + ~50 candles at 09:16 | Well within limits |
| Storage | ~200 MB (DBs + logs) | 45 GB free |

### Horizontal Scaling Path

```
Current:  1 server, 3 bots, 50 stocks, paper trading
    ↓
Phase 2:  Add Nifty 100 universe, switch to live trading
    ↓
Phase 3:  Separate data server (WebSocket aggregator)
          + N strategy servers (compute-only)
    ↓
Phase 4:  Multi-broker support (Fyers + Zerodha + Dhan)
          Load-balanced API layer
    ↓
Phase 5:  Client copy-trading at scale
          Dedicated DB server (PostgreSQL)
          Message queue (Redis) between components
```

### Adding a New Strategy

```python
# 1. Create backend/strategies/strategy_new.py
#    Must export: get_combined_signal(df, index_row) → Dict

# 2. Create backend/stock_selector/selector_new.py
#    Extend BaseSelector, implement _score_stock()

# 3. Register in selector_registry.py
_SELECTORS["new"] = NewSelector()

# 4. Create backend/main_new.py (copy from main_pro.py template)

# 5. Create algodhan_new.service

# 6. Deploy: bash scripts/deploy_3bots.sh (update to include new bot)
```

---

## File Structure Reference

```
Algostrategy_pro/
├── backend/
│   ├── main.py                  # Core trading loop (shared by all bots)
│   ├── main_scalper.py          # Scalper bot entry point
│   ├── main_pro.py              # Pro bot entry point
│   ├── main_v8.py               # V8 bot entry point
│   ├── config.py                # All configuration
│   ├── broker.py                # Fyers/Dhan abstraction
│   ├── risk_manager.py          # Position sizing + trade DB
│   ├── ws_manager.py            # WebSocket + heartbeat
│   ├── state_manager.py         # Thread-safe LTP + Regime
│   ├── notifier.py              # Telegram + Email
│   ├── market_scanner.py        # Nifty100 heat-map scanner
│   ├── regime_detector.py       # Market regime classification
│   ├── copy_trader.py           # Client copy-trading
│   ├── telegram_listener.py     # Telegram command handler
│   ├── api_server.py            # FastAPI entry point
│   ├── strategies/
│   │   ├── strategy_scalper_v1.py
│   │   ├── strategy_pro.py
│   │   ├── strategy_v8_fixed.py
│   │   └── strategy.py (base/legacy)
│   ├── stock_selector/
│   │   ├── base_selector.py
│   │   ├── selector_registry.py
│   │   ├── selector_scalper.py
│   │   ├── selector_pro.py
│   │   ├── selector_v8.py
│   │   └── scan_database.py
│   └── api/
│       ├── routers/ (auth, admin, trading, backtest, public, ws, reports)
│       ├── database.py, deps.py, models.py, helpers.py
│       └── ...
├── frontend/                    # Next.js 14 SaaS console
├── scripts/
│   ├── deploy_3bots.sh          # Deploy all strategy bots
│   ├── deploy_full.sh           # Full stack deployment
│   └── push_to_server.py        # SCP code to OCI
├── *.service                    # systemd service files
├── algodhan_nginx.conf          # Nginx config
├── .env.example                 # Environment template
└── README.md                    # This file
```

---

*Last updated: June 9, 2026 | Version: Multi-Strategy v3.0*
