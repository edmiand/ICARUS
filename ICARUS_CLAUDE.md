# ICARUS_CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Pine Script v6 TradingView indicator** — a standalone intraday reversal/exhaustion system. The
sole source file is `ICARUS.pine`. There is no build system, package manager, or test runner; development
means editing the `.pine` file and pasting it into TradingView's Pine Editor to compile and validate.

`ICARUS` is not a fork of `SuperLazyTrade` and shares no code with it, though it borrows several
implementation patterns (asset-profile assignment, Time-of-Day RVOL, dashboard construction) documented
in `SuperLazyTrade`'s own `CLAUDE.md`.

## The Premise

SuperLazyTrade rewards trend continuation. ICARUS trades the **failure** of a trend that has already run.
The setup is a three-state sequence, and a signal fires only on the transition into state 3:

1. **TRENDING** — an orderly directional move exists and has extended far enough (in daily-ATR units) to
   be worth giving back.
2. **BREAK** — structure fails: the prior swing low (or high) is taken out on expanding volume.
3. **CONFIRM** — the bounce/retest fails to reclaim → fire signal.

Extension is not the signal, it's the fuel gauge — it measures how much room exists to unwind once a break
happens. This is the mirror image of SuperLazyTrade's Gate 2 (Stretch), where high extension is a penalty:
here it is a prerequisite.

Direction convention: `_bear` state tracks the **bearish reversal** setup (an uptrend that may break down,
firing a SELL); `_bull` state mirrors it (a downtrend that may break up, firing a BUY). Every rule in the
system has an exact mirror across the two directions.

## Development Workflow

1. Edit `ICARUS.pine`
2. Copy the full file contents
3. Open TradingView → Pine Editor → paste → Save → Add to Chart
4. Compilation errors appear immediately in the Pine Editor console
5. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX, ES/SI futures)

## Architecture

The script is organized into sequential sections (read top-to-bottom, order matters in Pine Script):

1. **Constants** — `DASHBOARD_MAX_ROWS = 24`, `DASH_WIDTH` padding string.
2. **Types** — `TodSession` UDT (wraps `array<float>`) for Time-of-Day RVOL history.
3. **Inputs** — grouped by function:
   - `grp_setup` (🎯 Reversal Setup): swing pivot strength, min trend bars, max bars to confirm, close-vs-wick
     break requirement, session range anchor, min tick stop.
   - `grp_veto` (🛑 Vetoes): trend-day lockout, HTF veto + timeframe, time-of-day gates.
   - `grp_exit` (📐 Exits): stop ATR multiplier, Target 2 mode/R-multiple, time stop.
   - `grp_rvol` (📊 Relative Volume): RVOL mode + lookback sessions.
   - `grp_vis` (Dashboard & Visuals): dashboard/extended-metrics/state-marker/level toggles.
4. **Asset profile assignment** — same two-step pattern as SuperLazyTrade: `switch` on `syminfo.type` →
   `asset_category` (STOCK/FUND/FUTURES/CRYPTO/MARKET, default → MARKET), then an `if/else if` chain sets
   `profile_name` and the six threshold variables (`ext_min`, `atr_break_mult`, `rvol_break`,
   `adx_trend_min`, `retrace_min`, `retrace_max`). See [Profile Parameters Table](#profile-parameters-table).
5. **Core indicator calculations** — EMA(9/20/50), ATR(14), RSI(14), ADX(14,14) via `ta.dmi`, daily ATR via
   `request.security(..., lookahead_off)`.
6. **Session anchoring** — RTH-anchored (default) or calendar-day ("Full Session") session open/high/low.
   See [Session Anchoring](#session-anchoring--rth-not-calendar-day) below — the most consequential
   departure from SuperLazyTrade's design; everything downstream depends on it being correct.
7. **RTH-anchored VWAP** — manually accumulated cumulative price×volume / volume, reset on the same
   session boundary as #6. Deliberately does **not** use `ta.vwap()`.
8. **Time-of-Day Relative Volume** — ported from SuperLazyTrade's `TodSession` block, adapted to key off
   `is_new_range` instead of a calendar-day boundary. See
   [Time-of-Day RVOL Session Boundary](#time-of-day-rvol-session-boundary--adapted-not-verbatim).
9. **Non-repainting swing structure** — manually-tracked confirmed swing highs/lows (`last_swing_high`,
   `prev_swing_high`, and the `_low` equivalents, each with a paired `_bar` index), plus
   `structure_bullish`/`structure_bearish`/`structure_neutral`. Deliberately does **not** use
   `ta.pivothigh`/`ta.pivotlow` (banned for signal logic — see [Pine v6 Gotchas](#pine-script-v6-gotchas)).
   See [Swing Confirmation Delay](#swing-confirmation-delay).
10. **State 1: Trend Qualification** — `trend_qualified_bear`/`trend_qualified_bull` (`var bool` latches),
    each requiring: structure held `trendMinBars` bars (via a self-referencing streak counter), EMA
    cascade agreement, price on the correct side of `anchored_vwap`, `adx >= adx_trend_min`, and session
    extension `>= ext_min`. Persists through up to 10 consecutive failing bars, then resets; also resets
    on session rollover. See [Trend Qualification Latch](#trend-qualification-latch).
11. **State 2: Break Detection** — `break_armed_bear`/`break_armed_bull` (`var bool` latches), each
    requiring, from an already-qualified trend: the prior confirmed swing violated, a close beyond both
    `ema20` and `anchored_vwap`, `rel_vol >= rvol_break`, and bar range `>= atr * atr_break_mult`. Arming
    stores `break_bar_index_*`, the invalidation reference (`break_bar_high_bear`/`break_bar_low_bull`),
    and `break_extreme_*` (`session_high`/`session_low` at break time, for the retracement math in #12).
    Cancels on timeout (`confirmMaxBars`), on a reclaim past the invalidation reference, or on session
    end. Arming one direction immediately cancels the other (rule 8). No signal fires yet.
12. **State 3: Confirmation and Signal** — from an armed break, tracks the "bounce" (the peak of any
    green-closing bar since the break, and the low of the bar that set that peak — mirror for bullish),
    then fires `signal_sell`/`signal_buy` on a confirmed red/green bar that closes below/above the
    bounce's low/high, within `confirmMaxBars`, with retracement inside `[retrace_min, retrace_max]` of
    `session_range`. Strict alternation via `b_fired`/`s_fired` (ported from SuperLazyTrade). Records
    `entry_price_*`/`invalidation_*` for #14. Fires `"SELL ⚡ UNWIND"` / `"BUY ⚡ SNAPBACK"` labels.
    See [Bounce Tracking](#bounce-tracking-and-the-lower-high-test).
13. **Vetoes** — `bear_vetoed`/`bull_vetoed`, consumed by State 3's confirmation test:
    - *Trend-Day Lockout* (`useTrendDayLockout`) — a session-level latch (`bear_locked_out`/
      `bull_locked_out`) that trips when session range exceeds 1.5x daily ATR, close sits in the top/
      bottom 20% of session range, and ADX is both `> 30` and rising (3-bar). Once tripped it holds for
      the rest of the session.
    - *HTF Veto* (`useHtfVeto`) — hard veto (not a penalty) from `request.security` at `htfTimeframe`:
      blocks SELL when the HTF EMA9/EMA20/close all agree bullish, blocks BUY on the bearish mirror.
    - *Time-of-Day Gates* (`useTimeGates`) — blocks both directions inside `noEntryOpenMins`/
      `noEntryCloseMins` of the RTH open/close, computed directly from ET hour/minute rather than the
      `time()` session-string pattern used elsewhere. See [Pine v6 Gotchas](#pine-script-v6-gotchas).
    - *Consolidation Cancel* (Veto 4) needed no new code — already enforced by State 2's `confirmMaxBars`
      cancellation.
    Dashboard "Vetoes" row lists any active veto by name, direction-tagged where relevant.
14. **ATR Bracket Exits and R-Multiple Tracking** — on entry, computes and freezes `stop_*`/`target1_*`/
    `target2_*`/`risk_*` once (a bracket, not a trailing stop): stop beyond the invalidation reference
    (floored at `minTickStop` ticks, flagged on the dashboard when floored), Target 1 = VWAP at entry,
    Target 2 per `target2Mode`. Exit fires on whichever of STOP/T1/T2/TIME touches first (STOP checked
    first, conservatively). Closing a position clears the corresponding alternation guard
    (`b_fired`/`s_fired`). Realized R multiples push into `trade_r_results_long`/`_short` (FIFO-capped at
    50); Avg R is the headline dashboard metric, win rate secondary. See
    [ATR Bracket Design Notes](#atr-bracket-design-notes).
15. **Dashboard** — `table.new` at `position.bottom_right`, `DASHBOARD_MAX_ROWS = 24`; rendered only on
    `barstate.islast`. 18 fixed-count rows (8 base + 10 under Extended Metrics) — no variable-length
    section, so no overflow guard is needed anywhere. See
    [Dashboard Row Layout](#dashboard-row-layout) for the exact row map.
16. **Alerts** — `alertcondition` for SELL, BUY, T1, T2, STOP (no separate TIME alertcondition — matches
    the spec's exact list). A single dynamic `alert()` fires on every SELL/BUY entry with
    `syminfo.ticker`, direction, entry price, and stop baked into the message, so one alert setup can
    drive every monitored symbol.

---

## Profile Parameters Table

All threshold variables are assigned once per bar based on `syminfo.type`, without `var` (they reassign
every bar). The switch's default branch (`=>` with no match) makes MARKET the true catch-all for index,
forex, and unknown types — there is no separate DEFAULT profile.

| Profile | `ext_min` | `atr_break_mult` | `rvol_break` | `adx_trend_min` | `retrace_min` | `retrace_max` |
|---------|-----------|------------------|--------------|-----------------|---------------|---------------|
| MARKET INDEX 🏦 *(index, forex, unknown)* | 2.0 | 1.3 | 1.8 | 25 | 0.20 | 0.55 |
| ETF / FUND 📊 | 2.2 | 1.4 | 2.0 | 18 | 0.25 | 0.55 |
| STOCK 🚀 | 3.0 | 1.5 | 2.2 | 20 | 0.25 | 0.50 |
| FUTURES ⚡ | 2.5 | 1.4 | 1.8 | 15 | 0.25 | 0.55 |
| CRYPTO 🪙 | 4.0 | 1.6 | 2.0 | 28 | 0.30 | 0.60 |

Crypto is out of scope in practice: RTH time gates would suppress it entirely, and a 24h instrument has no
meaningful session-extension concept. The row exists for completeness only. Do not pool R-multiple
statistics across asset classes — track and evaluate each independently (see the calibration notes in the
original phase spec for why: ETFs, stocks, and futures have structurally different signal densities and
R-per-trade profiles).

---

## Key Design Decisions

**Session Anchoring — RTH, not calendar-day:** SuperLazyTrade derives `is_new_session` from a calendar-day
change (`ta.change(time("D"))`). ICARUS instead anchors `session_open_price` / `session_high` /
`session_low` (and the manual VWAP) to the first RTH bar of the day (`0930-1600:23456`, `rangeAnchor =
"RTH"`, the default). This applies to **all instrument types**, not just futures — ICARUS only trades
09:30–16:00 by design — but it matters most for futures, where a calendar-day anchor pulls in the
6pm-ET/midnight overnight session and would otherwise show 2–3 ATR of phantom "extension" at 09:35 that
was actually built in Asian/European hours, with `session_high`/`session_low` (and therefore
`retrace_depth`) measured against a denominator that has nothing to do with the move being traded. The
`"Full Session"` option falls back to the calendar-day detector for 24h instruments running with Time
Gates disabled; it is untested, and extension figures under it are not comparable to RTH-anchored ones.

**RTH-anchored VWAP:** `ta.vwap()` anchors to the exchange session, which begins at the overnight open on
futures — Target 1 (VWAP) would be dragged to the wrong price. `anchored_vwap` is instead accumulated
manually (`cum_pv` / `cum_vol`, both `var float`) and reset on the exact same `is_new_range` boundary as
the session high/low, so VWAP and the extension gate always agree on what "this session" means.

**Time-of-Day RVOL Session Boundary — adapted, not verbatim:** SuperLazyTrade's `TodSession` structure and
fallback logic are reused as-is, but the session boundary is `is_new_range` (this system's RTH-anchored
session) rather than SuperLazyTrade's calendar-day change, and bar-slot accumulation (`array.push` into
`tod_current_session.vols`, `tod_slot_idx` increment) is gated by `range_active` — in RTH mode this means
overnight bars are simply skipped rather than occupying a bar-slot. Without this, slot indices would drift
between sessions and the break-bar RVOL test (the core discriminator in State 2, per the calibration
notes) would silently compare against the wrong slot.

**Swing Confirmation Delay:** a swing high at absolute bar `M` requires `swingLookback` bars of hindsight
on its right side — it can only be confirmed once bar `M + swingLookback` has closed. `is_pivot_high` /
`is_pivot_low` are evaluated as `high[swingLookback] == ta.highest(high, 2*swingLookback+1)` (the mirror
for lows), and the resulting state mutation is gated by `barstate.isconfirmed`. Concretely, with the
default `swingLookback = 5`, a swing printed on bar 100 is not visible to `last_swing_high` /
`last_swing_low` until bar 105 closes — a fixed 5-bar delay, identical in live trading and historical
replay (all historical bars are "confirmed" during replay, so the delay there is just the natural bar
sequence). No signal path in this system can read a swing before that delay elapses, because
`last_swing_high`/`last_swing_low` (and their `prev_*` predecessors) are the *only* swing references
anywhere in the script. `structure_bullish`/`structure_bearish` are pure reads of that already-confirmed
state, so they add no repaint risk of their own.

**Trend Qualification Latch:** `trend_qualified_bear` (a qualifying UPTREND, `structure_bullish`, that may
break down) and `trend_qualified_bull` (the mirror, off `structure_bearish`) are independent `var bool`
latches, not forced mutually exclusive at this stage. In practice they rarely coexist, because
`structure_bullish`/`structure_bearish` already can't both be true on the same bar and the streak counters
(`bull_structure_streak`/`bear_structure_streak`) that gate qualification reset to 0 the instant structure
flips. But because a latch persists through up to 10 consecutive bars of failing conditions before
resetting, it is theoretically possible for both to be true briefly if structure whipsaws while one latch
is still inside its failure window — the dashboard's "State" row breaks that tie by displaying bear first,
deterministically. Rule 8 ("one direction at a time") is not enforced here because it applies to the
break-arming stage below, not to State 1 eligibility.

**Break arming is independent of the qualification latch that spawned it:** once `bear_break_trigger` arms
`break_armed_bear`, the arm's own three cancel conditions (timeout, reclaim, session end) are the only
things that clear it — `trend_qualified_bear` going false afterward does **not** cancel an active arm. This
is not an oversight: the moment a break happens, price is by definition below `anchored_vwap`, which
immediately fails one of `bear_setup_conditions`' five terms, so `trend_qualified_bear`'s fail-streak starts
counting the instant the break prints. If arming depended on qualification staying true, almost every arm
would self-cancel within a few bars purely from its own success, which is backwards. The dashboard's
"State" row reflects this: `⚡ ARMED` takes display priority over `TRENDING`, so a stale/reset qualification
latch underneath an active arm is invisible by design, not a bug.

**Rule 8 enforcement point:** "one direction at a time" is enforced exactly where arming happens — the
instant `break_armed_bear` (or `_bull`) latches true, the opposite direction's armed state and all its
fields are force-reset in the same bar, before that direction's own arm/cancel branch runs later in the
same `if barstate.isconfirmed` block. It is not enforced at the qualification stage (see
[Trend Qualification Latch](#trend-qualification-latch)) because qualification is eligibility, not
commitment — only an armed break is a live setup that would conflict with an opposite-direction one.

**Bounce Tracking and the "Lower High" Test:** the arm's own invalidation check only cancels on a bar
*closing* back above `break_bar_high_bear` — an intrabar wick above it does not cancel the arm. That means
"the bounce peaked below break_bar_high" cannot be assumed true just because the arm is still active; it is
tracked independently as `bounce_high_bear`, the running max `high` across every green-closing bar since
the break, paired with `bounce_peak_low_bear` (the low of whichever bar set that peak — "the bounce bar's
low"). Tracking resets to a clean slate at the instant of a fresh arm (`bear_break_armed_now`), not on
cancellation — since the tracking loop only ever runs while `break_armed_bear` is true, stale values from
a cancelled episode are simply never read, so a reset on cancel would be redundant.

**Interpretation: what counts as "the bounce".** A real bounce can span several green bars before rolling
over. This implementation tracks the *running peak* across all green-closing bars since the break — not
just the first one, and not just the most recent one — on the reasoning that "the bounce" means the whole
rally attempt, and its relevant peak/low pair is whichever bar actually set the high. This is a judgment
call the spec doesn't pin down exactly.

**Time Gates use ET hour/minute directly, not the `time()` session-string pattern:** everywhere else in
this file, RTH boundaries are tested via `time(timeframe.period, "0930-1600:23456")` (returns non-`na`
inside the session). The Time Gates veto instead computes `hour(time, "America/New_York") * 60 +
minute(time, "America/New_York")` and compares directly to 570 (09:30) and 960 (16:00) minutes-since-
midnight. This is necessary because the veto needs *distance* from the boundary (minutes since open,
minutes to close), not just an in/out-of-session boolean. Like every other RTH reference in this file, it
is hardcoded to America/New_York and inappropriate for crypto / 24h futures without disabling
`useTimeGates`.

**HTF Veto is unconditional `request.security`, gated at the read site:** `request.security` for
`htf_ema9`/`htf_ema20`/`htf_close` runs every bar regardless of `useHtfVeto`, per the Pine convention of
not making `request.security` calls conditional. `useHtfVeto` only gates whether `htf_bullish`/
`htf_bearish` are allowed to contribute to `bear_vetoed`/`bull_vetoed`.

**ATR Bracket Design Notes:**

- *Everything is frozen at entry, including "EMA50."* Session Open and the Fixed-R formula are inherently
  static once entry price and risk are known, but `target2Mode = "EMA50"` could plausibly mean either "the
  EMA50 value at entry" (frozen) or "exit when price touches the live, moving EMA50" (a trailing target).
  This implementation freezes it at entry, for consistency with the other two modes and with the framing
  of "ATR **Bracket** Exits" — a bracket order has fixed legs set once, not a target that chases price.
- *Exit priority order (STOP > T1 > T2 > TIME) is a judgment call, not specified in the phase text.* A
  single OHLC bar can't tell an indicator whether price touched the stop or a target first intrabar, so
  STOP is checked first as the conservative assumption. T1 is checked before T2 because Target 1 (VWAP)
  is normally closer to entry than Target 2, so it would be touched first in the typical case anyway.
- *`timeStopBars = 0` is not documented as "disabled."* Unlike `minTickStop`, whose input label explicitly
  says "0 = off," `timeStopBars`'s label does not. This implementation takes that at face value: `0` means
  the time stop fires on the very next bar after entry (`bar_index - entry_bar >= 0` is immediately true),
  not "disabled." As implemented, setting `timeStopBars = 0` will force nearly every trade to exit after
  one bar via the TIME branch.
- *Position-close clears only the alternation guard, not sizing.* ICARUS is an `indicator()`, not a
  `strategy()` — there is no real position size or P&L account. "Closing the position" means: record one
  realized R multiple, flip `in_position_*` false, and clear `b_fired`/`s_fired` so the next qualifying
  signal can fire. There is no partial-exit/scaling model — whichever of T1/T2/STOP/TIME touches first
  closes the entire (notional) trade for R-tracking purposes.

**Dashboard Row Layout:** all rows are fixed-count (no variable-length loop like SuperLazyTrade's
gate-detail word-wrap), so no `if row >= DASHBOARD_MAX_ROWS: break` guard is needed anywhere in this
dashboard — every combination of inputs produces exactly the same 18 rows (0-17) when Extended Metrics is
on, or 8 rows (0-7) when it's off. Both are far under `DASHBOARD_MAX_ROWS = 24`.

| Row | Label | Notes |
|-----|-------|-------|
| 0 | Header (merged) | `ICARUS V1 \| <short state>` |
| 1 | Profile | |
| 2 | State | `UNQUALIFIED` / `TRENDING ↑↓` / `⚡ ARMED` / `🎯 IN TRADE` |
| 3 | Extension | active direction vs `ext_min`, falls back to `max(bear, bull)` when neither side is active |
| 4 | Structure | `HH/HL ↑` / `LH/LL ↓` / `NEUTRAL` |
| 5 | Retracement | only meaningful while armed; `—` otherwise |
| 6 | Vetoes | |
| 7 | Position | |
| 8 | separator | extDash only |
| 9 | Swing H/L | extDash only |
| 10 | RVOL | extDash only |
| 11 | ADX + rising | extDash only |
| 12 | HTF Bias | extDash only |
| 13 | Bars Since Break | extDash only |
| 14 | separator | extDash only |
| 15 | Avg R — LONG | extDash only |
| 16 | Avg R — SHORT | extDash only |
| 17 | Trades / Win% | extDash only, secondary/dimmed |

**IN TRADE outranks ARMED, which outranks TRENDING:** this priority chain is safe to treat as
non-overlapping rather than a tie-break, because entering a trade clears the corresponding `break_armed_*`
flag in the same bar the entry fires — `in_position_*` and `break_armed_*` for the same direction are
never simultaneously true by construction.

**No composite score, by design:** unlike SuperLazyTrade, ICARUS has no 0-100 scoring engine. Every gate
in this system is a hard boolean condition (qualify / don't qualify, veto / don't veto), per the phase
spec's explicit "explicitly out of scope" list. Do not add one without discussing it first.

**Extension is a prerequisite, not a penalty:** the inverse of SuperLazyTrade's Gate 2 (Stretch). Do not
port stretch-penalty logic across — a large `ext_min` reading here *qualifies* a setup rather than
disqualifying it.

**Alerts:** `alertcondition` covers SELL, BUY, T1, T2, STOP — deliberately no separate TIME
alertcondition, matching the exact list given. The single dynamic `alert()` call (on `signal_sell`/
`signal_buy`) uses `alert.freq_once_per_bar_close`, consistent with those signals only ever firing on a
confirmed bar in the first place.

---

## Pine Script v6 Gotchas

Common failure modes encountered while building this script:

- **Boolean `na` on the first bar:** `rth_active[1]` is `na` (not `false`) on bar 0 / start-of-history, so
  a naive `rth_active and not rth_active[1]` new-range detector silently fails to fire on the very first
  bar. Guarded explicitly: `rth_active and (na(rth_active[1]) or not rth_active[1])`.
- **Arrays cannot nest** — `array<array<float>>` does not compile; `TodSession` (wrapping `array<float>`)
  is the standard workaround, identical to SuperLazyTrade's usage.
- **`request.security` lookahead:** `atr_daily` and the HTF EMA/close fetch both use
  `lookahead = barmerge.lookahead_off`. Always pass the expression directly into `request.security` —
  do not pre-compute into a `var` first.
- **`str.format_time(time, format, timezone)`** is the correct call for rendering a Unix-ms timestamp into
  a readable string — not `str.tostring`, which has no time-format overload for raw `int` time values.
- **`ta.pivothigh`/`ta.pivotlow` are banned for signal logic** in this script (not a general Pine
  limitation, a deliberate project rule) — they carry a built-in lookback offset that confirms N bars late
  in a way that's easy to lose track of. Swing highs/lows are instead tracked via an explicit
  `high[swingLookback] == ta.highest(high, 2*swingLookback+1)` test, gated by `barstate.isconfirmed`, so
  the confirmation delay is visible in the code rather than hidden inside a built-in.
- **Functions cannot reassign a value-type global variable** (`float`/`int`/`bool`/`string`), even one
  declared with `var` — only reference types (`array`/`matrix`/`map`/UDT) can be mutated from inside a
  function, and only via a parameter. This is why `add_r_result(array<float> arr, ...)` takes the array as
  a parameter and mutates it via `array.push`/`array.shift`, while the dozens of `var` scalar state
  variables throughout this file (`break_armed_bear`, `entry_price_bear`, `stop_bear`, etc.) are all
  mutated inline at their call sites rather than through helper functions.
- **A self-referencing series variable** (e.g. `bull_structure_streak = structure_bullish ?
  nz(bull_structure_streak[1]) + 1 : 0`) is a standard, legal Pine idiom for a consecutive-bar counter —
  the `[1]` reads the same variable's value from the previous bar. No `var` keyword is needed or wanted
  here; `var` would break the "recompute every bar" semantics this pattern relies on.
- **`hour()`/`minute()` accept an explicit timezone as a second argument** (`hour(time, "America/New_York")`)
  — this is how the Time Gates veto gets ET-local minutes-since-midnight regardless of the chart's own
  timezone setting, independent of the `time(timeframe.period, "session-string")` idiom used elsewhere.

---

## Compile-Check Checklist

No test runner exists. After any edit, verify manually in this order:

1. **Paste into Pine Editor → zero compilation errors** before proceeding.
2. **Load an NVDA or SPY 2-min chart** → confirm the dashboard appears at bottom-right with all 8 base
   rows, and (with Extended Metrics on) all 18 rows, no runtime error.
3. **Load an ES or SI 2-min futures chart** → confirm the Extension row reads near zero at 09:35 ET, not
   2–3 ATR. If it does not, the RTH anchor is broken — nothing downstream can be trusted until this is
   fixed. Toggle `Session Range Anchor` to `Full Session` on the same chart and confirm Extension jumps up
   (expected — it now includes the overnight range).
4. **Swing confirmation delay:** on a chart with `Swing Pivot Strength = 5`, pick an obvious swing high on
   the chart and confirm `Last Swing High` (when `Show Swing / Invalidation Levels` is on) only updates to
   that price 5 bars after it printed, never sooner.
5. **Structure classification:** manually walk a few bars where price makes two ascending swing
   highs/lows in a row → confirm the dashboard's Structure row shows `HH/HL ↑`; two descending → `LH/LL
   ↓`; mixed → `NEUTRAL`.
6. **Trend qualification:** find or force a bar where all five State 1 conditions hold → confirm the State
   row flips to `TRENDING ↑` (or `↓`) with a plausible extension figure; confirm it persists through a
   brief (<10 bar) dip in conditions and resets to `UNQUALIFIED` after 10 consecutive failing bars.
7. **Break arm/cancel:** force a break trigger (or find one in history) → confirm State shows `⚡ ARMED
   (bar N/confirmMaxBars)`, incrementing each bar; let it run past `confirmMaxBars` bars with no
   confirmation → confirm it cancels back to `TRENDING` or `UNQUALIFIED`, not stuck on `ARMED`. Separately
   confirm a close back above/below the invalidation reference also cancels it before the bar count runs
   out.
8. **Signal firing and alternation:** let a SELL fire → confirm the `SELL ⚡ UNWIND` label appears below
   the bar (BUY `⚡ SNAPBACK` above — this inverted convention is intentional, see the phase spec) →
   confirm no second SELL fires until either a BUY fires or the open SELL position closes via T1/T2/STOP/
   TIME.
9. **Exit priority and labels:** with a position open, confirm exactly one of STOP/T1/T2/TIME fires when
   its level is touched (never more than one on the same bar), the corresponding label appears at the
   correct price, and the Position row returns to `FLAT` immediately after.
10. **Stop tick floor:** on a thin futures contract, set `Min Stop Distance (ticks)` above the natural
    ATR-based stop distance → confirm the Position row shows `⚠FLOOR` next to the stop price when an entry
    fires, and that the widened distance is reflected in the displayed R value.
11. **R-multiple tracking:** let several trades close across both directions → confirm `Avg R — LONG` /
    `Avg R — SHORT` update independently, `Trades / Win%` shows correct counts per side, and a stopped-out
    trade contributes exactly `-1.00R` (since stop exits realize exactly `-risk`).
12. **Vetoes:** toggle each veto on/off independently (`Trend-Day Lockout`, `HTF Veto`, `Time-of-Day
    Gates`) → confirm the Vetoes row shows the correct name(s) when active and `NONE` (gray) when none are,
    and confirm a vetoed direction genuinely cannot produce a signal while its veto is active.
13. **Rule 8 (one direction at a time):** force conditions where a bearish break would arm while a bullish
    break is already armed → confirm the bullish arm is immediately cancelled the instant the bearish one
    latches, never both `⚡ ARMED` simultaneously.
14. **Dashboard row count:** with Extended Metrics ON, count exactly 18 rows rendered (0-17); with it OFF,
    exactly 8 (0-7); confirm no Pine runtime row-index error in either configuration.
15. **Alerts:** create both alert types (`alertcondition`-based, and the plain "Alert on ICARUS" dynamic
    `alert()`) → confirm the dynamic alert's message includes the correct ticker, direction, entry price,
    and stop price when a SELL or BUY fires.

---

## Version

The script file is `ICARUS.pine` (version-agnostic name). It is currently baselined at **V1** with no
changelog history tracked in the file or in this document — treat the current source as the reference
behavior going forward.
