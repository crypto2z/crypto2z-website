---
title: Building a volatility-aware DCA bot on Coinbase Advanced Trade
date: 2025-04-10
category: dca-bot
tags: python, coinbase, flask, systemd
---

I've been dollar-cost averaging into crypto manually for a while — same amounts, same days, no thought required. But I wanted to go a step further: what if the bot bought *more* when prices dipped significantly? And what if those dip thresholds were based on each coin's actual historical volatility rather than a single number I made up?

Here's how I built it.

## The core idea

Standard DCA: buy $2 of each coin on Monday, Wednesday, and Friday. No matter what price is doing.

Volatility-aware DCA: same schedule, but if a coin drops more than its historical dip threshold in 24 hours, trigger an *additional* buy on top of the regular one.

The thresholds are derived from 180 days of OHLCV data — so BTC's threshold is tighter (it moves less violently) and AVAX's is wider (it moves a lot).

## Analyzing 180 days of history

I pulled historical daily OHLCV data for all 8 coins from CoinGecko and wrote `analyze_volatility.py` to find the threshold that captured meaningful dips without triggering on normal daily noise:

```python
import pandas as pd

def get_dip_threshold(prices, percentile=20):
    daily_changes = prices.pct_change().dropna()
    negative_changes = daily_changes[daily_changes < 0]
    threshold = negative_changes.quantile(percentile / 100)
    return round(abs(threshold) * 100, 1)
```

The resulting thresholds in `rules.py`:

```python
DIP_THRESHOLDS = {
    'BTC':  7.5,
    'ETH':  10.6,
    'ADA':  11.6,
    'AVAX': 13.5,
    'ATOM': 10.1,
    'POL':  12.8,
    'XLM':  10.1,
    'XRP':  9.5,
}
```

## The bot structure

The bot runs as a systemd user service on a DigitalOcean droplet. On buy days (Mon/Wed/Fri), it:

1. Fetches current prices and 24h change for each coin from Coinbase Advanced Trade API
2. Places a $2 market order for each coin via `-USDC` pairs
3. Checks if any coin has dropped beyond its dip threshold — if so, places an additional buy
4. Logs everything to a SQLite database

Key things I learned about the Coinbase Advanced Trade API:
- Timestamps must be ISO 8601 format with timezone (`2025-04-10T14:00:00Z`)
- Private key in `.env` needs real newlines, not `\n` literals — must be quoted
- Market orders use `quote_size` (dollar amount), not `base_size`

## The Flask dashboard

A lightweight Flask app runs as a second systemd service on port 5000, firewalled to my home IP only. It shows:

- Last buy time and amounts per coin
- Current portfolio values (fetched live)
- Recent trade log
- Dip threshold status — which coins are near their trigger

```python
@app.route('/')
def index():
    trades = db.get_recent_trades(limit=50)
    prices = coinbase.get_prices(COINS)
    return render_template('dashboard.html', trades=trades, prices=prices)
```

## The trailing stop system

After a few months running the basic bot, I added a trailing stop for non-BTC coins. Every day at 10 AM:

1. Fetch 30-day high for each non-BTC coin
2. Compare current price to that high
3. If current price is more than 30% below the 30-day high → sell 50% of the position

BTC is excluded entirely — it's my cornerstone holding and I don't want any automation touching it.

```python
def check_trailing_stops():
    for coin in NON_BTC_COINS:
        high_30d = get_30d_high(coin)
        current = get_current_price(coin)
        drop_pct = (high_30d - current) / high_30d * 100
        if drop_pct >= 30:
            sell_half_position(coin)
            log_stop_trigger(coin, drop_pct)
```

## Running it in production

Both services defined in `~/.config/systemd/user/`:

```ini
[Unit]
Description=DCA Bot

[Service]
WorkingDirectory=/root/dca-bot
ExecStart=/root/dca-bot/venv/bin/python bot.py
Restart=always
Environment=XDG_RUNTIME_DIR=/run/user/0

[Install]
WantedBy=default.target
```

The `XDG_RUNTIME_DIR` line is essential for systemd user services running as root — without it the service fails silently.

## What's next

I want to add position sizing based on conviction — allocating more to coins that are both dipping AND showing on-chain accumulation signals. That's a rabbit hole for another weekend.

All the volatility thresholds are visible on the [Tools page](/tools.html) if you want to see the numbers.
