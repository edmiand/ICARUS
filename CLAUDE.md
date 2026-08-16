# ICARUS V0 — implementation notes

Single file: `ICARUS.pine` (Pine Script v6, ~340 lines). No `strategy()`, no backtesting engine,
no multi-symbol scanning. This file exists to explain two places where the shipped code
deliberately diverges from the original written spec, because both were found through actual
chart testing, not guesswork, and a future session touching this file should understand why
before "fixing" them back to the literal spec text.

## Structure

- **Phase 1** (session/VWAP/swing structure): RTH-anchored VWAP computed manually (never
  `ta.vwap`, which anchors to the overnight session on futures). Swing highs/lows tracked in
  `var float` with an explicit `swingLookback`-bar confirmation delay — never compared against
  the current bar until confirmed, to avoid repainting.
- **Phase 2** (state machine): BREAK → CANCEL/CONFIRM for both bear (SELL) and bull (BUY) setups,
  plus the trend-day lockout veto.
- **Phase 3** (exits + dashboard): stop/target/time-stop exits, 5-row status table, BUY/SELL-only
  alerts.

## Deviation 1: break reference uses the stretch-excursion's own extreme bar, not `last_swing_high/low`

**Original spec text:** BULL arms when `close > last_swing_high` while stretched below VWAP
(mirrors bear's `close < last_swing_low` while stretched above VWAP).

**Why it was changed:** `last_swing_high`/`last_swing_low` only reflect *past* confirmed swing
extremes. On a rally, small pullback lows form close to the current elevated price, so breaking
one while still comfortably stretched above VWAP is geometrically easy — the bear condition
works as written. But on a fast, single-leg decline (no intervening bounce), no swing high ever
confirms near the eventual low — the only candidate is the far-away pre-decline peak. Reclaiming
that peak while still 2%+ below VWAP is numerically impossible: the reclaim price is above the
"still stretched" ceiling implied by VWAP. Confirmed on real data (SPCE, TSLA) via a temporary
debug build — `stretched_bull` fired repeatedly at the lows, `armed_bull` never did, and the
Data Window showed the swing-high reference sitting far outside the reachable stretch band.

**The fix:** track the extreme bar (lowest low, for BULL) of the *current* stretched excursion
and use *that bar's own opposite-side wick* as the reclaim target instead. This anchor is always
close to the extreme by construction — a bar range, not a distant swing. The anchor must survive
after the excursion's `stretched_bull`/`stretched_bear` flag goes false (reclaiming it is what
"no longer stretched" means), so it's only cleared on a new session or once used to arm — see the
comment block above `stretch_bull_low` in the code. Applied symmetrically to BEAR for the same
reason (a fast one-leg rally has the identical problem in reverse).

## Deviation 2: trend-day lockout is a live per-bar condition, not a permanent latch

**Original spec text:** `bear_locked_out`/`bull_locked_out` latch to `true` once
`session_range_pct > 2.5 and in_top/bottom_20 and adx > 30`, and clear only on the next session's
open — explicitly "not optional."

**Why it was changed:** `in_top_20`/`in_bottom_20` use the *running* session high/low, so the bar
that makes a new session extreme is trivially "at the extreme." A permanent latch can't tell a
genuine trend day (price keeps grinding near its extreme for hours — confirmed on a real Micron
trend day) apart from a sharp flush that fully reverses within the hour (confirmed on SPCE and
TSLA) — both look identical at the moment of the extreme. Under the literal spec, both test
symbols locked out for the entire rest of the session after an early flush, permanently vetoing
the exact reversion trade ICARUS is built to catch.

**The fix:** re-evaluate the lockout condition every bar instead of latching. A short persistence
filter (`extremePersistBars = 5`, i.e. ~10 minutes on a 2-min chart) still requires the extreme to
hold for a few bars — filtering a single spike bar — but the condition drops out again once price
has clearly moved away. A sustained trend still locks out within minutes and stays locked as long
as price keeps making new extremes; a flush-and-recover unlocks itself.

## Testing history (for context, not exhaustive)

Both fixes were driven by testing SPCE and TSLA sessions with a real capitulation-and-recovery
shape — exactly the pattern ICARUS targets and exactly where the literal spec text failed
silently (zero signals, no error). Micron and NVDA sessions were also tested: Micron as a
genuine sustained trend day (confirms the lockout still works as a veto), NVDA as a too-low-range
day (confirms the stretch filter correctly produces nothing when there's no real move to catch).

If a future session sees zero signals on a chart and starts second-guessing thresholds, check
first whether the day actually contains the four-condition pattern at all (via `swing_high_val`/
`swing_low_val`/`VWAP Dist %`/`rel_vol` in the Data Window — add them back as temporary
`display.data_window`-only plots if needed) before assuming something is broken.
