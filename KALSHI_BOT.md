# Kalshi Arb Bot — Full Documentation

**Last updated:** March 1, 2026  
**Status:** 🔴 LIVE — trading real money  
**Monitor:** https://mdaleyy99.github.io/kalshi-monitor/  
**Report:** https://mdaleyy99.github.io/kalshi-monitor/report.html

---

## Overview

A latency arbitrage bot targeting Kalshi's 15-minute BTC/ETH Up/Down CLOB contracts. It exploits the lag between Coinbase spot price moves and Kalshi market repricing — when BTC/ETH moves fast, Kalshi limit orders briefly become mispriced. The bot buys the stale side, holds until Kalshi reprices, then exits.

**Settlement:** Kalshi 15-min contracts settle against CF Benchmarks BRTI — a 60-second average of Coinbase + Kraken prices. Coinbase feed IS the signal.

---

## Strategy

```
1. Track BTC/ETH spot via Coinbase WebSocket (real-time)
2. On each tick, compute 45s momentum
3. If |momentum| >= 0.20% → check for edge on current 15-min window
4. Compute fair_value = f(spot, floor_strike, momentum, time_remaining)
5. If fair > kalshi_ask + 5¢ → buy the stale ask
6. Exit when Kalshi reprices (TP +5¢), stop loss (-4¢), or 90s timeout
```

**Why it works:**  
Kalshi has human limit orders that sit in the CLOB. After a fast BTC move, it takes 5-30 seconds for those orders to get updated. The bot buys the stale price before humans cancel/reprice.

---

## Current Config

| Parameter | Value | Description |
|---|---|---|
| `LIVE` | `True` | Real money trading |
| `TRADE_DOLLARS` | `$10` | Per trade budget |
| `MAX_OPEN` | `3` | Max concurrent positions |
| `MAX_DAILY` | `100` | Max trades per day |
| `COOLDOWN_SECS` | `30s` | Min time between trades on same series |
| `MOMENTUM_WINDOW` | `45s` | Lookback for momentum calc |
| `MOMENTUM_MIN_PCT` | `0.20%` | Min move to trigger signal check |
| `EDGE_MIN_CENTS` | `5¢` | Min (fair - ask) required to enter |
| `TP_CENTS` | `+5¢` | Take profit threshold |
| `SL_CENTS` | `-4¢` | Stop loss threshold |
| `TIME_EXIT_SECS` | `90s` | Hard timeout regardless of P&L |
| `POLL_SECS` | `4s` | Position monitor poll interval |
| `API_THROTTLE` | `3s` | Min time between Kalshi market fetches |

---

## Markets Traded

| Series | Contract | Description |
|---|---|---|
| `KXBTC15M` | BTC 15-min windows | BTC ends above floor strike? |
| `KXETH15M` | ETH 15-min windows | ETH ends above floor strike? |

Windows reset every 15 minutes. Floor strike = BTC/ETH price at window open. Bot trades both YES (up momentum) and NO (down momentum).

---

## Fair Value Formula

```python
def fair_value(spot, floor_strike, momentum_pct, secs_remaining):
    gap_pct = (spot - floor_strike) / floor_strike * 100.0
    sensitivity = 200.0 * sqrt(300.0 / max(30.0, secs_remaining))
    base = 50.0 + gap_pct * sensitivity
    momentum_boost = momentum_pct * 15.0
    fair = base + momentum_boost
    return clamp(fair, 2.0, 98.0)
```

- **Gap sensitivity** scales with `1/√time_remaining` — same % gap is worth more when time is short
- **Momentum boost** adds directional edge: +0.30% move in 45s ≈ +4.5¢ toward favorable direction
- Calibrated against live data: spot=$67,836, floor=$67,782 (+0.08%), 355s rem → market=66¢

---

## Files

```
kalshi/
├── bot.py                  Main bot — strategy, fair value, order logic
├── kalshi_client.py        Kalshi REST API client (RSA auth)
├── price_feed.py           Coinbase WebSocket price feed + momentum
├── watchdog.py             Auto-restart bot if dead or frozen
├── monitor.html            Live monitor page (served via GitHub Pages)
├── report.html             Trade report with full stats + charts
├── start.sh                Start script (bot + Cloudflare tunnel)
├── kalshi_private_key.pem  RSA private key for Kalshi API auth
└── KALSHI_BOT.md           This file
```

---

## API Credentials

| Key | Value |
|---|---|
| Kalshi Key ID | `REDACTED` |
| Kalshi PEM | `kalshi/kalshi_private_key.pem` |
| Kalshi API Base | `https://api.elections.kalshi.com` |
| GitHub Token | `ghp_REDACTED` |
| GitHub Repo | `mdaleyy99/kalshi-monitor` |
| Kalshi Balance | `$199.95` |

---

## Running Processes

```bash
# Bot (live trading)
python3 -u bot.py > /tmp/kalshi_bot.log 2>&1 &

# Watchdog (auto-restart)
python3 -u watchdog.py > /tmp/kalshi_watchdog.log 2>&1 &
```

**Check status:**
```bash
ps aux | grep -E "bot.py|watchdog.py" | grep -v grep
tail -20 /tmp/kalshi_bot.log
tail -20 /tmp/kalshi_watchdog.log
```

**Kill everything:**
```bash
pkill -f "bot.py"
pkill -f "watchdog.py"
```

**Restart:**
```bash
cd /Users/michaeldaley/.openclaw/workspace/kalshi
pkill -f "bot.py"; pkill -f "watchdog.py"; sleep 1
python3 -u bot.py > /tmp/kalshi_bot.log 2>&1 &
python3 -u watchdog.py > /tmp/kalshi_watchdog.log 2>&1 &
```

---

## State & Monitoring

**State file:** `/tmp/kalshi_state.json` — written every 30s and on every trade event

```json
{
  "ts": 1772342806.91,
  "mode": "LIVE",
  "balance": 199.95,
  "btc": 67509.67,
  "eth": 2022.0,
  "btc_mom": 0.01,
  "eth_mom": 0.02,
  "open_positions": [],
  "daily_trades": 0,
  "daily_pnl": 0.0,
  "trades": [...]
}
```

**Local HTTP server:** `http://localhost:7070/state` (CORS open)

**GitHub state:** Pushed every 15s to `mdaleyy99/kalshi-monitor/contents/state.json`  
Monitor fetches via GitHub API with `?t=Date.now()` cache buster.

---

## Watchdog

`watchdog.py` runs independently and checks every 60s:

1. **Bot process alive** — `pgrep -f bot.py`; restarts if dead
2. **State freshness** — if state > 120s old while bot is running, kills + restarts (frozen bot)
3. **Monitor page** — HTTP check every 5 min; logs if unreachable

Logs: `/tmp/kalshi_watchdog.log`

---

## Paper Trading Results (Feb 28 – Mar 1, 2026)

| Metric | Value |
|---|---|
| Total Trades | 13 |
| Win Rate | 76.9% (10W / 3L) |
| Total P&L | +$30.39 |
| Avg Win | +$3.28 |
| Avg Loss | -$0.81 |
| Profit Factor | 12.6× |
| Best Trade | +$14.52 (ETH YES @ 30¢ → 74¢, 33 contracts) |
| Worst Trade | -$1.40 (ETH NO SL) |
| Avg Edge at Entry | 23.8¢ |
| Avg Hold Time | ~18s |

Exit type breakdown: 10 TP / 3 SL / 0 Timeout

---

## Trade Entry Logic (step by step)

```
tick arrives from Coinbase WS
  → compute 45s momentum
  → if |momentum| < 0.20%: skip
  → if API was called < 3s ago: skip (throttle)
  → if open positions >= 3: skip
  → if daily trades >= 100: skip
  → if same series traded < 30s ago: skip (cooldown)
  → fetch current open market from Kalshi
  → compute fair_yes = fair_value(spot, floor, momentum, time_rem)
  → compute fair_no = 100 - fair_yes
  → yes_edge = fair_yes - yes_ask
  → no_edge = fair_no - no_ask
  → if best edge < 5¢: skip
  → contracts = floor($10 / (entry_ask / 100))
  → place order (LIVE) or log only (PAPER)
  → spawn monitor_position thread
```

---

## Position Monitor Logic

Polls every 4 seconds:
```
current_bid >= entry_ask + 5¢  →  EXIT TP  (take profit)
current_bid <= entry_ask - 4¢  →  EXIT SL  (stop loss)
elapsed >= 90s                 →  EXIT TM  (timeout)
market closed                  →  EXIT     (window ended)
```

P&L = `(exit_bid - entry_ask) / 100 * contracts`

---

## Known Limitations

1. **State resets on reboot** — `/tmp/` is cleared; daily P&L and trade log lost until persistent storage is added
2. **No position recovery** — if bot restarts mid-position, open positions are lost from tracking
3. **Single machine** — if Mac sleeps, bot dies (mitigated by watchdog, but Mac sleep is the root cause)
4. **13 paper trades is small sample** — live results may differ; monitor closely first 50 trades
5. **GitHub push rate limit** — 15s minimum between pushes means monitor max 35s behind reality

---

## Next Steps / Improvements

- [ ] Move state to persistent file in workspace (not `/tmp/`)
- [ ] Add position recovery on restart (read open positions from Kalshi API)
- [ ] Add daily P&L reset at midnight
- [ ] Increase `TRADE_DOLLARS` if first 20 live trades are profitable
- [ ] Consider VPS deploy for 24/7 uptime without Mac dependency
- [ ] Add Kalshi WebSocket for faster repricing detection vs REST polling
- [ ] Tune `MOMENTUM_MIN_PCT` and `EDGE_MIN_CENTS` based on live data
