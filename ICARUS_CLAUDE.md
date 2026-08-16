# ICARUS_CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Status: Phase 1 of an 8-phase build. This document will be filled out fully at Phase 8 — it currently
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

Not yet implemented (later phases): trend qualification, break detection, confirmation/signal firing,
vetoes, ATR bracket exits, R-multiple tracking, full state-machine dashboard, alerts.

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
