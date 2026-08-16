# ICARUS V0

**Intraday Collapse And Reversal Unwind Signal** — Pine Script v6, TradingView.

A proof-of-concept indicator, not a product. It answers one question:

> When an extended intraday trend breaks structure and the bounce fails, does price revert to
> VWAP often enough to be worth trading?

## The setup

A SELL fires when all four hold (BUY is the mirror):

1. **Stretched from VWAP** — price is at least `Min Distance from VWAP (%)` above VWAP.
2. **Structure broke** — close below a recent support level, on volume ≥ `Break Bar RVOL`.
3. **Bounce failed** — price bounced, peaked below the break bar's high, and closed back down.
4. **Not a trend day** — the trend-day lockout is not currently active.

Signals fire only on confirmed bars — never intrabar, never repainted.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| Min Distance from VWAP (%) | 2.0 | Entry filter and profit target distance |
| Swing Strength | 5 | Bars each side required to confirm a swing high/low |
| Break Bar RVOL | 2.0 | Volume multiple (vs. 20-bar average) required on the break bar |
| Max Bars to Confirm | 8 | Window after a break for the bounce-and-fail to complete |
| Stop Buffer Above Bounce (%) | 0.15 | Stop placed this far beyond the failed bounce extreme |
| Time Stop (bars) | 40 | Force-exit a trade after this many bars if neither stop nor target hit |
| Trend-Day Lockout | on | Vetoes new signals on days that look like a sustained trend, not a reversion |
| Show Dashboard | on | Bottom-right status table |
| Show Swing Levels | on | Plots the tracked swing high/low as stepped lines |

## Exits

- **Stop**: failed-bounce extreme + buffer
- **Target**: live VWAP (a moving magnet, not a fixed price)
- **Time stop**: `Time Stop (bars)` after entry

Exactly one of `TGT` / `STOP` / `TIME` fires per position, labeled with the realized % move.
One position at a time — no new signal while one is open.

## Testing protocol

Run on 2-minute charts, RTH only. Screen each morning for **ATR% > 2.5** — low-volatility
symbols (SPY, ES) won't produce a 2% VWAP reversion and should be excluded from the test set.
Intended universe: high-beta single names (NVDA, TSLA), sector ETFs (SOXX, SMH), metals/energy
futures (SI, CL, NG).

Expect **0–2 signals per symbol per day, most days zero**. There is deliberately no built-in
win-rate or R-multiple tracking — read the exit labels off the chart and hand-tally target-hits
vs. stop-hits for the first 20 sessions. The concept passes if target-hits meaningfully outnumber
stop-hits.

See `CLAUDE.md` for implementation details, the two deliberate deviations from the original spec,
and why they were necessary.
