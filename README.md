# FVG Master — Automated FVG Trading System

Two complementary pieces:

| File | Purpose |
|---|---|
| `fvg_strategy.pine` | TradingView Pine Script v5 strategy. Visualizes FVG zones, backtests entries/exits, and can fire alerts. **Cannot place live orders by itself.** |
| `fvg_trading_bot.py` | Standalone Python bot. Detects FVGs and trades **directly on Binance or Bybit** via `ccxt`. This is the actually-automated part. |

Pine Script and the Python bot use the same core logic, so the TradingView chart gives you a visual/backtest sanity check of what the bot will do live.

## Why not "just" Pine Script?
TradingView strategies only execute inside TradingView's simulator. To touch a real exchange account you need either:
- TradingView webhook alerts → a server that relays them to the exchange, or
- A standalone bot that talks to the exchange API directly (what `fvg_trading_bot.py` does).

The Python bot is the simpler, more reliable path since it doesn't depend on keeping a TradingView chart/alert running.

## How the FVG logic works
1. Looks at every 3-candle sequence for a price imbalance (gap) of at least `min_gap_atr_mult × ATR`.
2. Marks the gap as a zone to watch.
3. Enters when price retraces back into that zone, optionally filtered by a 200 EMA trend.
4. Stop loss sits just beyond the zone; take profit is a fixed risk:reward multiple (default 2:1).

## Setup — Python bot
```bash
pip install ccxt
export EXCHANGE_API_KEY="your_key"
export EXCHANGE_API_SECRET="your_secret"
python fvg_trading_bot.py
```
Edit the `Config` block at the top of `fvg_trading_bot.py` to set:
- `exchange_name`: `"binance"` or `"bybit"`
- `symbol`, `timeframe`
- `paper_trading`: **leave `True`** until you've watched it run for a while
- risk settings (`risk_per_trade_pct`, `rr_ratio`, etc.)

## Setup — Pine Script (optional, for visualization/backtesting)
1. Open TradingView → Pine Editor → paste `fvg_strategy.pine` → Add to Chart.
2. Open Strategy Tester to review backtest performance.
3. If you want TradingView alerts too (separate from the Python bot), right-click the chart → Add Alert → use the strategy's alert conditions. The JSON payload format is already built in for a webhook bridge if you build one later.

## Security
- Create exchange API keys with **trading enabled, withdrawals disabled**.
- IP-whitelist the key to the server running the bot.
- Never commit API keys to source control — this template reads them from environment variables only.

## Honest limitations of this template
- Position sizing and balance checks are simplified — review them against your exchange's actual margin/lot rules before going live.
- Only one open position at a time; no partial fills, no re-entry logic, no slippage modeling.
- Stop loss / take profit are managed in software (the bot watches price and submits a closing market order) rather than placed as exchange-side conditional orders — if the bot process goes down, those exits won't fire. For real money, layering on exchange-native stop orders is strongly recommended.

## Risk disclaimer
This is an educational template, not financial advice, and not a guarantee of profitable trading. Crypto markets are volatile and automated strategies can lose money quickly, especially with leverage. Backtest thoroughly, run in paper mode for an extended period, and only risk capital you can afford to lose.
