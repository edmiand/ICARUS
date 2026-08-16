# ICARUS_CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Status: Phase 7 of an 8-phase build. This document will be filled out fully at Phase 8 — it currently
covers only what exists in the code today.**

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

## Development Workflow

1. Edit `ICARUS.pine`
2. Copy the full file contents
3. Open TradingView → Pine Editor → paste → Save → Add to Chart
4. Compilation errors appear immediately in the Pine Editor console
5. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX, ES/SI futures)

## Architecture (current — Phase 0)

1. **Constants** — `DASHBOARD_MAX_ROWS = 24`, `DASH_WIDTH` padding string.
2. **Types** — `TodSession` UDT (wraps `array<float>`) for Time-of-Day RVOL history.
3. **Inputs** — grouped by function:
   - `grp_setup` (🎯 Reversal Setup): swing pivot strength, min trend bars, max bars to confirm, close-vs-wick
     break requirement, session range anchor, min tick stop.
   - `grp_veto` (🛑 Vetoes): trend-day lockout, HTF veto + timeframe, time-of-day gates.
   - `grp_exit` (📐 Exits): stop ATR multiplier, Target 2 mode/R-multiple, time stop.
   - `grp_rvol` (📊 Relative Volume): RVOL mode + lookback sessions.
   - `grp_vis` (Dashboard & Visuals): dashboard/extended-metrics/state-marker/level toggles.
   - All exit/veto/setup inputs are declared now per spec but not yet wired to logic — that lands in
     Phases 1–6.
4. **Asset profile assignment** — same two-step pattern as SuperLazyTrade: `switch` on `syminfo.type` →
   `asset_category` (STOCK/FUND/FUTURES/CRYPTO/MARKET, default → MARKET), then an `if/else if` chain sets
   `profile_name` and the six threshold variables (`ext_min`, `atr_break_mult`, `rvol_break`,
   `adx_trend_min`, `retrace_min`, `retrace_max`). See [Profile Parameters Table](#profile-parameters-table).
5. **Core indicator calculations** — EMA(9/20/50), ATR(14), RSI(14), ADX(14,14) via `ta.dmi`, daily ATR via
   `request.security(..., lookahead_off)`.
6. **Session anchoring** — RTH-anchored (default) or calendar-day ("Full Session") session open/high/low.
   See [Session Anchoring](#session-anchoring) below — this is the most consequential departure from
   SuperLazyTrade's design and everything downstream depends on it being correct.
7. **RTH-anchored VWAP** — manually accumulated cumulative price×volume / volume, reset on the same
   session boundary as #6. Deliberately does **not** use `ta.vwap()`.
8. **Time-of-Day Relative Volume** — ported from SuperLazyTrade's `TodSession` block, adapted to key off
   `is_new_range` instead of a calendar-day boundary. See [Key Design Decisions](#key-design-decisions).
9. **Scaffold dashboard** — a minimal placeholder table (Profile, RVOL, Session Anchor + timestamp, raw
   Extension) so Phase 0's session-anchoring work is visible on a chart. Replaced by the full
   state-machine dashboard in Phase 7.
10. **Non-repainting swing structure** — manually-tracked confirmed swing highs/lows (`last_swing_high`,
    `prev_swing_high`, and the `_low` equivalents, each with a paired `_bar` index), plus
    `structure_bullish`/`structure_bearish`/`structure_neutral`. See
    [Swing Confirmation Delay](#swing-confirmation-delay) below. Deliberately does **not** use
    `ta.pivothigh`/`ta.pivotlow` (banned for signal logic per the phase spec — see
    [Pine v6 Gotchas](#pine-script-v6-gotchas-encountered-so-far)).
11. **State 1: Trend Qualification** — `trend_qualified_bear`/`trend_qualified_bull` (`var bool` latches),
    each requiring: structure held `trendMinBars` bars (via a self-referencing streak counter), EMA
    cascade agreement, price on the correct side of `anchored_vwap`, `adx >= adx_trend_min`, and session
    extension `>= ext_min`. Persists through up to 10 consecutive failing bars, then resets; also resets
    on session rollover. See [Trend Qualification Latch](#trend-qualification-latch) below.
12. **State 2: Break Detection** — `break_armed_bear`/`break_armed_bull` (`var bool` latches), each
    requiring, from an already-qualified trend: the prior confirmed swing violated, a close beyond both
    `ema20` and `anchored_vwap`, `rel_vol >= rvol_break`, and bar range `>= atr * atr_break_mult`. Arming
    stores `break_bar_index_*`, the invalidation reference (`break_bar_high_bear`/`break_bar_low_bull`),
    and `break_extreme_*` (`session_high`/`session_low` at break time, for Phase 4's retracement math).
    Cancels on timeout (`confirmMaxBars`), on a reclaim past the invalidation reference, or on session
    end. Arming one direction immediately cancels the other (rule 8). No signal fires yet — arming only
    sets up Phase 4's confirmation test.
13. **State 3: Confirmation and Signal** — from an armed break, tracks the "bounce" (the peak of any
    green-closing bar since the break, and the low of the bar that set that peak — mirror for bullish),
    then fires `signal_sell`/`signal_buy` on a confirmed red/green bar that closes below/above the
    bounce's low/high, within `confirmMaxBars`, with retracement inside `[retrace_min, retrace_max]` of
    `session_range`. Strict alternation via `b_fired`/`s_fired` (ported from SuperLazyTrade). Records
    `entry_price_*`/`invalidation_*` for Phase 6. Fires `"SELL ⚡ UNWIND"` / `"BUY ⚡ SNAPBACK"` labels.
    See [Bounce Tracking](#bounce-tracking-and-the-lower-high-test) below.
14. **Vetoes** — `bear_vetoed`/`bull_vetoed`, consumed by State 3's confirmation test:
    - *Trend-Day Lockout* (`useTrendDayLockout`) — a session-level latch (`bear_locked_out`/
      `bull_locked_out`) that trips when session range exceeds 1.5x daily ATR, close sits in the top/
      bottom 20% of session range, and ADX is both `> 30` and rising (3-bar). Once tripped it holds for
      the rest of the session.
    - *HTF Veto* (`useHtfVeto`) — hard veto (not a penalty) from `request.security` at `htfTimeframe`:
      blocks SELL when the HTF EMA9/EMA20/close all agree bullish, blocks BUY on the bearish mirror.
    - *Time-of-Day Gates* (`useTimeGates`) — blocks both directions inside `noEntryOpenMins`/
      `noEntryCloseMins` of the RTH open/close, computed directly from ET hour/minute rather than the
      `time()` session-string pattern used elsewhere. See [Pine v6 Gotchas](#pine-script-v6-gotchas-encountered-so-far).
    - *Consolidation Cancel* (Veto 4) needed no new code — already enforced by Phase 3's `confirmMaxBars`
      cancellation.
    Dashboard "Vetoes" row lists any active veto by name, direction-tagged where relevant.
15. **ATR Bracket Exits and R-Multiple Tracking** — on entry, computes and freezes `stop_*`/`target1_*`/
    `target2_*`/`risk_*` once (a bracket, not a trailing stop): stop beyond the invalidation reference
    (floored at `minTickStop` ticks, flagged on the dashboard when floored), Target 1 = VWAP at entry,
    Target 2 per `target2Mode`. Exit fires on whichever of STOP/T1/T2/TIME touches first (STOP checked
    first, conservatively). Closing a position clears the corresponding alternation guard
    (`b_fired`/`s_fired`) — this is what Phase 4 deferred. Realized R multiples push into
    `trade_r_results_long`/`_short` (FIFO-capped at 50); Avg R is the headline dashboard metric, win rate
    secondary. See [ATR Bracket Design Notes](#atr-bracket-design-notes) below.

16. **Dashboard** — the full 18-row state-machine dashboard (replaces the scaffold used through Phases
    0-6): Header, Profile, State (now including 🎯 IN TRADE, which outranks ARMED), Extension,
    Structure, Retracement, Vetoes, Position as fixed base rows; a separator, Swing H/L, RVOL, ADX,
    HTF Bias, Bars Since Break, a separator, Avg R — LONG, Avg R — SHORT, and Trades/Win% under
    Extended Metrics. See [Dashboard Row Layout](#dashboard-row-layout) below for the exact row map and
    what was dropped from the scaffold.

Not yet implemented (later phases): alerts.

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
meaningful session-extension concept. The row exists for completeness only.

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
futures — Target 1 (VWAP, introduced in Phase 6) would be dragged to the wrong price. `anchored_vwap` is
instead accumulated manually (`cum_pv` / `cum_vol`, both `var float`) and reset on the exact same
`is_new_range` boundary as the session high/low, so VWAP and the extension gate always agree on what "this
session" means.

**Time-of-Day RVOL session boundary — adapted, not verbatim:** SuperLazyTrade's `TodSession` structure and
fallback logic are reused as-is, but the session boundary is `is_new_range` (this system's RTH-anchored
session) rather than SuperLazyTrade's calendar-day change, and bar-slot accumulation (`array.push` into
`tod_current_session.vols`, `tod_slot_idx` increment) is gated by `range_active` — in RTH mode this means
overnight bars are simply skipped rather than occupying a bar-slot. Without this, slot indices would drift
between sessions and the break-bar RVOL test (the core discriminator in Phase 3, per the calibration notes)
would silently compare against the wrong slot.

**Swing Confirmation Delay:** a swing high at absolute bar `M` requires `swingLookback` bars of hindsight
on its right side — it can only be confirmed once bar `M + swingLookback` has closed. `is_pivot_high` /
`is_pivot_low` are evaluated as `high[swingLookback] == ta.highest(high, 2*swingLookback+1)` (the mirror
for lows), and the resulting state mutation is gated by `barstate.isconfirmed`. Concretely, with the
default `swingLookback = 5`, a swing printed on bar 100 is not visible to `last_swing_high` /
`last_swing_low` until bar 105 closes — a fixed 5-bar delay, identical in live trading and historical
replay (all historical bars are "confirmed" during replay, so the delay there is just the natural bar
sequence). No signal path in this system can read a swing before that delay elapses, because
`last_swing_high`/`last_swing_low` (and their `prev_*` predecessors) are the *only* swing references
anywhere in the script — there is no separate, faster-updating pivot value anything could accidentally
read instead. `structure_bullish`/`structure_bearish` are pure reads of that already-confirmed state, so
they add no repaint risk of their own.

**Trend Qualification Latch:** `trend_qualified_bear` (a qualifying UPTREND, `structure_bullish`, that may
break down) and `trend_qualified_bull` (the mirror, off `structure_bearish`) are independent `var bool`
latches, not forced mutually exclusive at this stage. In practice they rarely coexist, because
`structure_bullish`/`structure_bearish` already can't both be true on the same bar and the streak counters
(`bull_structure_streak`/`bear_structure_streak`) that gate qualification reset to 0 the instant structure
flips. But because a latch persists through up to 10 consecutive bars of failing conditions before
resetting, it is theoretically possible for both to be true briefly if structure whipsaws while one latch
is still inside its failure window — the dashboard's "State" row breaks that tie by displaying bear first,
deterministically. Rule 8 ("one direction at a time") is not enforced here because it applies to Phase 3's
"armed" break state, not to State 1 eligibility — Phase 3 will need to explicitly cancel the opposite
side's armed state when one arms.

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
same `if barstate.isconfirmed` block. It is not enforced at Phase 2's qualification stage (see
[Trend Qualification Latch](#trend-qualification-latch)) because qualification is eligibility, not
commitment — only an armed break is a live setup that would conflict with an opposite-direction one.

**Bounce Tracking and the "Lower High" Test:** the arm's own invalidation check (Phase 3) only cancels on
a bar *closing* back above `break_bar_high_bear` — an intrabar wick above it does not cancel the arm. That
means "the bounce peaked below break_bar_high" (Phase 4, condition 2) cannot be assumed true just because
the arm is still active; it is tracked independently as `bounce_high_bear`, the running max `high` across
every green-closing bar since the break, paired with `bounce_peak_low_bear` (the low of whichever bar set
that peak — "the bounce bar's low" from the spec). Tracking resets to a clean slate at the instant of a
fresh arm (`bear_break_armed_now`), not on cancellation — since Phase 4's tracking loop only ever runs
while `break_armed_bear` is true, stale values from a cancelled episode are simply never read, so a reset
on cancel would be redundant.

**Interpretation: what counts as "the bounce".** The spec describes the bounce as a single event ("at
least one bar... closed green", "the bounce peaked", "the bounce bar's low") but a real bounce can span
several green bars before rolling over. This implementation tracks the *running peak* across all
green-closing bars since the break — not just the first one, and not just the most recent one — on the
reasoning that "the bounce" means the whole rally attempt, and its relevant peak/low pair is whichever bar
actually set the high. This is a judgment call, not something the spec pins down exactly.

**Time Gates use ET hour/minute directly, not the `time()` session-string pattern:** everywhere else in
this file, RTH boundaries are tested via `time(timeframe.period, "0930-1600:23456")` (returns non-`na`
inside the session). The Time Gates veto instead computes `hour(time, "America/New_York") * 60 +
minute(time, "America/New_York")` and compares directly to 570 (09:30) and 960 (16:00) minutes-since-
midnight. This is necessary because the veto needs *distance* from the boundary (minutes since open,
minutes to close), not just an in/out-of-session boolean — the session-string form only answers "is this
bar inside the window," not "by how much." Like every other RTH reference in this file, it is hardcoded to
America/New_York and inappropriate for crypto / 24h futures without disabling `useTimeGates`.

**HTF Veto is unconditional `request.security`, gated at the read site:** `request.security` for
`htf_ema9`/`htf_ema20`/`htf_close` runs every bar regardless of `useHtfVeto`, per the Pine convention of
not making `request.security` calls conditional. `useHtfVeto` only gates whether `htf_bullish`/
`htf_bearish` are allowed to contribute to `bear_vetoed`/`bull_vetoed` — this avoids the recalculation
churn/edge cases that conditional `request.security` calls can trigger.

**ATR Bracket Design Notes:**

- *Everything is frozen at entry, including "EMA50."* Session Open and the Fixed-R formula are inherently
  static once entry price and risk are known, but `target2Mode = "EMA50"` could plausibly mean either "the
  EMA50 value at entry" (frozen) or "exit when price touches the live, moving EMA50" (a trailing target).
  This implementation freezes it at entry, for consistency with the other two modes and with the phase's
  own framing — "ATR **Bracket** Exits" — a bracket order has fixed legs set once, not a target that
  chases price. If a moving EMA50 target was intended, that's a different (and larger) change: it would
  mean Target 2 is no longer a fixed level at all, which has knock-on effects for how "current R" and the
  eventual realized R at exit are computed.
- *Exit priority order (STOP > T1 > T2 > TIME) is a judgment call, not specified in the phase text.* A
  single OHLC bar can't tell an indicator whether price touched the stop or a target first intrabar,
  so STOP is checked first as the conservative assumption. T1 is checked before T2 because Target 1 (VWAP)
  is normally closer to entry than Target 2, so it would be touched first in the typical case anyway.
- *`timeStopBars = 0` is not documented as "disabled."* Unlike `minTickStop`, whose input label explicitly
  says "0 = off," `timeStopBars`'s label does not. This implementation takes that at face value: `0` means
  the time stop fires on the very next bar after entry (`bar_index - entry_bar >= 0` is immediately true),
  not "disabled." If a disable convention was intended for `0`, flag it — as implemented, setting
  `timeStopBars = 0` will force nearly every trade to exit after one bar via the TIME branch.
- *Position-close clears only the alternation guard, not sizing.* ICARUS is an `indicator()`, not a
  `strategy()` — there is no real position size or P&L account. "Closing the position" here means: record
  one realized R multiple, flip `in_position_*` false, and clear `b_fired`/`s_fired` so the next
  qualifying signal can fire. There is no partial-exit/scaling model — whichever of T1/T2/STOP/TIME
  touches first closes the entire (notional) trade for R-tracking purposes.

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

**Dropped from the scaffold:** the Phase 0 "Session Anchor" row (anchor mode + timestamp) is not in the
Phase 7 spec's row list and has been removed — the scaffold dashboard was explicitly temporary
("replaced by the full dashboard in Phase 7" was noted in every phase's dashboard comment). Anchor
sanity-checking is still indirectly possible via the Extension row (it should read near zero at 09:35 on
RTH-anchored futures), just without the explicit timestamp readout. `session_open_time` is still tracked
internally (harmless, unused elsewhere) in case a future phase wants it back.

**IN TRADE outranks ARMED, which outranks TRENDING:** this priority chain is safe to treat as
non-overlapping rather than a tie-break, because entering a trade (Phase 6) clears the corresponding
`break_armed_*` flag in the same bar the entry fires — `in_position_*` and `break_armed_*` for the same
direction are never simultaneously true by construction.

**Extension is a prerequisite, not a penalty:** the inverse of SuperLazyTrade's Gate 2 (Stretch). Do not
port stretch-penalty logic across — a large `ext_min` reading here *qualifies* a setup rather than
disqualifying it. (Full qualification logic — `trend_qualified_bear/bull` — lands in Phase 2; today's
dashboard shows only a raw, unqualified extension figure for anchor verification.)

---

## Pine Script v6 Gotchas (encountered so far)

- **Boolean `na` on the first bar:** `rth_active[1]` is `na` (not `false`) on bar 0 / start-of-history, so
  a naive `rth_active and not rth_active[1]` new-range detector silently fails to fire on the very first
  bar. Guarded explicitly: `rth_active and (na(rth_active[1]) or not rth_active[1])`.
- **Arrays cannot nest** — `array<array<float>>` does not compile; `TodSession` (wrapping `array<float>`)
  is the standard workaround, identical to SuperLazyTrade's usage.
- **`request.security` lookahead:** `atr_daily` uses `lookahead = barmerge.lookahead_off`. This is
  display-only in Phase 0 (no signal reads it yet) but the flag is set now so nothing downstream has to
  remember to add it later.
- **`str.format_time(time, format, timezone)`** is the correct call for rendering a Unix-ms timestamp
  (`session_open_time`) into a readable dashboard string — not `str.tostring`, which has no time-format
  overload for raw `int` time values.

---

## Compile-Check Checklist (partial — Phase 0 only)

Full checklist arrives at Phase 8. For now, verify:

1. **Paste into Pine Editor → zero compilation errors** before proceeding.
2. **Load an NVDA or SPY 2-min chart** → scaffold dashboard appears at bottom-right showing Profile, RVOL,
   Session Anchor, and raw Extension.
3. **Load an ES or SI 2-min futures chart** → confirm the "Extension (raw)" dashboard row reads near zero
   at 09:35 ET, not 2–3 ATR. If it does not, the RTH anchor is broken — stop and fix before Phase 1.
4. **Toggle `Session Range Anchor` to "Full Session"** on the same futures chart → confirm the Extension
   figure jumps up (now including the overnight range) — this is expected/documented behavior for that
   mode, not a bug.
5. **Toggle `Relative Volume Mode`** between `Rolling 20-bar` and `Time-of-Day` → confirm the RVOL
   dashboard row's multiplier and mode label (`TOD` / `20-bar` / `20-bar*`) change accordingly.
6. **Profile check** — load a stock, an ETF, and a futures chart → confirm the Profile row shows the
   correct profile name for each.

---

## Version

The script file is `ICARUS.pine` (version-agnostic name), currently baselined at **V1**. Phase 0 of 8;
see the phase spec for the full build plan.
