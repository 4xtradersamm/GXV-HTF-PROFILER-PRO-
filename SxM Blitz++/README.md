# SxM Blitz++

A Pine Script v6 indicator that builds a full ICT-style fractal time framework — dividers, dealing ranges, true opens, premium/discount, trend bias, SMT divergence, turtle soup liquidity sweeps, and CISD confirmation — on top of **one single time engine**, so every feature agrees on what "day," "week," "session," or "quarter" means instead of drifting out of sync with each other.

> **License note:** This script is released under the **Mozilla Public License 2.0**. It also embeds and extends the **GXV SMT_Divergences[LITE✦]** engine by **Gregorius_XV** (TradingView: Gregorius_XV), used with attribution per the MPL-2.0 terms of the original work, and imports the published **`GXVSanctuary/AssetCorrelationUtils/5`** library for correlated-asset pair resolution. See [Credits & License](#credits--license) below.

---

## Table of Contents

1. [Design Philosophy](#design-philosophy)
2. [Core Time Engine](#core-time-engine)
3. [Timeframe Hierarchy](#timeframe-hierarchy)
4. [Dividers](#dividers)
5. [Dealing Range](#dealing-range)
6. [Premium / Discount](#premium--discount)
7. [True Opens](#true-opens)
8. [Trend Engine](#trend-engine)
9. [SSMT (SMT Divergence)](#ssmt-smt-divergence)
10. [HTF SSMT → MTF tCISD Alignment](#htf-ssmt--mtf-tcisd-alignment)
11. [Turtle Soup](#turtle-soup)
12. [tCISD (Change in State of Delivery)](#tcisd-change-in-state-of-delivery)
13. [Dashboard](#dashboard)
14. [Alerts](#alerts)
15. [HTF Candle Panel (bonus module)](#htf-candle-panel-bonus-module)
16. [Settings Reference](#settings-reference)
17. [Credits & License](#credits--license)
18. [Disclaimer](#disclaimer)

---

## Design Philosophy

Most multi-timeframe ICT indicators compute "day," "week," and "session" boundaries separately for every feature that needs them — one clock for the divider labels, another for the dealing range, another for true opens — and those clocks quietly disagree the moment a holiday, a DST change, or a data gap shows up.

SxM Blitz++ has **one time engine** (Core Calendar, below) that every other feature reads from. Nothing downstream computes its own day/week/session math independently. The one deliberate exception is the SMT engine, which keeps its own internal clock because its 13-rank range-reconstruction logic is tightly coupled to it (documented in-code and below).

A second design rule: every recurring boundary is detected by **comparing this bar's value to the previous bar's value** (`eM != eM[1]`, `tD != tD[1]`, etc.), never by matching an exact calendar value (`day == 1`). Exact-match detection silently breaks on higher timeframes where no bar ever lands precisely on that value — a bug this build specifically fixes.

---

## Core Time Engine

**Inputs:** `Timezone` group — one manual UTC offset (`-4` for EDT, `-5` for EST; defaults to `-4`).

Every date/time field the rest of the script uses (`eY`, `eM`, `eD`, `eH`, `eMin`, `eDow`) is derived once from this offset. There is no per-feature timezone logic anywhere else.

On top of that, a second derived clock — the **trading-day clock** — shifts everything by 6 hours so that **18:00 EST becomes the daily rollover point** (matching the real futures/forex session open) instead of midnight. This trading-day clock (`tD`, `tDow`) drives every "Day" concept used for **structure**: the weekday dividers (Mon→Fri), week boundaries, and the Dealing Range. Calendar month/quarter/year boundaries (`eM`, `eY`) stay on the plain, unshifted clock, since those don't shift with the daily session convention.

---

## Timeframe Hierarchy

One resolver (`primaryNew` / `parentNew`) decides, purely from the chart's own resolution, which two boundaries are in play — and every other feature (dividers, labels, dealing range, true opens) reads these two signals instead of re-deriving them:

| Chart TF | Primary divider | Parent divider |
|---|---|---|
| 1–5m | Every 90 minutes | Every 6 hours (session) |
| 15m | Every 6 hours (session) | Daily |
| 30m–1H | Daily | Weekly |
| 2–4H | Weekly | Monthly |
| 1D | Monthly | Every 3 months (quarter) |
| 1W+ | Every 3 months (quarter) | Yearly |

---

## Dividers

**Inputs:** `Dividers` group — show/hide vertical dividers and their labels, independently.

Draws a vertical divider line and a label at every `primaryNew` boundary, using whichever label makes sense for that tier: session name (Asia/London/NY AM/NY PM), weekday (MON–FRI), week-of-month (W1–W4), month-of-quarter (M1–M3), or quarter-of-year (QT1–QT4).

---

## Dealing Range

**Inputs:** `Dealing Range` group — show/hide, extend-through-parent toggle, color.

Computes a high/low box from a fixed "first sub-period" window, then holds and optionally extends that box for the rest of its parent period:

| Chart TF | Range window | Resets |
|---|---|---|
| 1–5m | 1st 90-minute quarter of the session | every session |
| 15m | Asia session (18:00–00:00 EST), always | every day |
| 30m–1H | Monday | every week |
| 2–4H | Week 1 of the month | every month |
| 1D | Week 1 of the month | **every month** |
| 1W+ | — | **no Dealing Range at this tier** |

---

## Premium / Discount

Derived with no inputs of its own: the **50% midpoint** of the current Dealing Range. Price above the midpoint is Premium, below is Discount. This feeds only the Trend Engine — it is not a separate filter anywhere else.

---

## True Opens

**Inputs:** `True Opens` group — show/hide the drawn marker, history length, color.

Every level's true open (except Day) follows the same rule: **the open of that level's 2nd child period** — Tuesday (2nd day of the week), week 2 (2nd week of the month), month 2 (2nd month of the quarter), April (2nd quarter of the year). Day's true open follows the identical rule one level deeper: since a Day's children are its 4 sessions, Day's true open is the **2nd session's open — 00:00 EST (midnight)** — even though the Day *divider* itself rolls at 18:00.

Two markers are drawn per chart tier where relevant: a **primary** (the parent bracket's true open — the headline reference for that tier) and, on the three most intraday tiers, a **secondary** (the active bracket's own true open, faded):

| Chart TF | Primary | Secondary |
|---|---|---|
| 1–5m | Q2 of the session (90-min quarter #2) | Session Open (literal) |
| 15m | TDO — Day True Open (00:00 EST) | Session TO (Q2 of session) |
| 30m–1H | TWO — Week True Open (Tuesday) | TDO (00:00 EST) |
| 2–4H | TMO — Month True Open (week 2) | — |
| 1D | TQO — Quarter True Open (month 2) | — |
| 1W+ | TYO — Year True Open (April) | — |

---

## Trend Engine

**Inputs:** `Trend Engine` group — one master enable toggle.

A simple three-factor **OR**: the trend is bullish if *any* of the following agree (bearish is the mirror image), not a majority vote:

- **MACD state** — `macdLine > macdSignalLine` (a continuous state check, not a crossover event). Standard 12/26/9 lengths, fixed internally rather than exposed as inputs.
- **Premium/Discount** — price below the Dealing Range's 50% midpoint = bullish, above = bearish.
- **True Open** — price above the current tier's primary True Open = bullish, below = bearish.

---

## SSMT (SMT Divergence)

**Inputs:** several groups — `SMT Visual Settings`, `SMT Gates Settings`, `SMT Render Settings`, `SMT Session Filter Settings`, `SMT Asset Pair Resolver Settings`.

This is the **full GXV SMT_Divergences[LITE✦] engine** by Gregorius_XV, embedded directly (not a simplified port):

- **13 timeframe ranks computed simultaneously** — 15m, 30m, 1H, 90m, 4H, 6H, 7H, 12H, 1D, 1W, 1M, 3M, 12M — independent of the chart's own resolution, each with its own show/hide toggle and color.
- **Real auto pair mapping** via the imported `GXVSanctuary/AssetCorrelationUtils/5` library — Dyad / Triad / Tetrad correlation types, GXT or Classic correlation modes, or manual pair entry with inversion.
- **Deterministic range-key reconstruction**: each rank tracks the current vs. previous range's high/low; a divergence is valid only when *exactly one* of the two compared markets sweeps its corresponding extreme (strict `>`/`<` — a touch is not a sweep).
- **LIVE → CONFIRMED → INVALIDATED lifecycle**, rank-dominance cleanup (configurable: remove, or fade lower-priority overlapping events), aesthetic bypass, session-time filtering, and display caps — all from the original engine.

Confirmed events across all enabled ranks/pairs are aggregated into two booleans (`bullishSSMTEvent` / `bearishSSMTEvent`) that feed tCISD (below) — the same two signals regardless of which or how many ranks actually fired.

**Note on clocks:** this engine deliberately keeps GXV's own session/day clock rather than this script's EST engine, since its rank/bucket math is tightly coupled to it in the original design.

A separate, simpler symbol picker (Auto or manual Symbol A / B) exists only to feed the HTF Alignment gate and the tCISD reference-candle search below — it is independent of the SSMT engine's own correlation resolver.

---

## HTF SSMT → MTF tCISD Alignment

**Inputs:** `HTF SSMT -> MTF tCISD` group — enable toggle, higher timeframe, validity window (bars).

An optional permission gate: when enabled, tCISD may only arm in the direction of the most recent SMT divergence detected on a separate, higher timeframe (using the simple two-symbol picker), within a configurable bar window. Off by default.

---

## Turtle Soup

**Inputs:** `Turtle Soup` group — swing lookback, max markers per timeframe, current-TF and up to 5 additional HTF overlays.

A classic stop-run reversal: price sweeps *below* a prior swing low and closes back *above* it (bullish), or sweeps *above* a prior swing high and closes back *below* it (bearish).

---

## tCISD (Change in State of Delivery)

**Inputs:** `tCISD Settings` group — show/hide, reference-candle search window, line extension, setup validity window, strict trend requirement, arm-from-Turtle-Soup toggle, bull/bear colors.

tCISD arms when **both** conditions hold:

1. A qualifying event exists — an SSMT divergence (from the full engine above) **or**, optionally, a same-timeframe Turtle Soup sweep — matched by direction (a bullish event can only arm a bullish setup, never a bearish one).
2. The Trend Engine agrees with that same direction (any one of MACD / Premium-Discount / True Open).

Once armed, it searches back for the nearest opposite-colored reference candle and watches for a close beyond that candle's body extreme to confirm. A small independent marker ("SSMT `<rank>`" or "TS") is stamped at the exact bar it arms, tagged with the specific rank that fired (e.g. "SSMT 4H") when the source is a higher timeframe than the chart — this marker is drawn outside the SMT engine's own cleanup pipeline, so it can never be deleted by rank-dominance arbitration even if the underlying SMT line is.

---

## Dashboard

**Inputs:** `Dashboard` group — show/hide.

A compact table (top-right) showing: current SMT cycle name and quarter number, overall trend (BULL/BEAR/NEUTRAL), the current session/sub-quarter, weekday, week-of-month, month-of-quarter, quarter-of-year, and — for each of Day/Week/Month/Quarter/Year — whether price is trading above or below that level's True Open.

---

## Alerts

Six alert conditions, all firing on confirmation: Bullish/Bearish SSMT, Bullish/Bearish HTF SSMT, Bullish/Bearish tCISD.

---

## HTF Candle Panel (bonus module)

A separate, self-contained module (carried over unchanged) that renders miniature synthetic candles for 90-Minute, 6-Hour, and Daily timeframes off to the side of the chart, independent of everything above. Includes its own PSP (correlated-pair sweep) overlay, Fair Value Gap detection, sweep/midpoint trace lines, and full per-timeframe color customization. This module does **not** feed the Trend Engine or any other feature — it is a standalone visual reference panel.

---

## Settings Reference

| Group | What it controls |
|---|---|
| Timezone | Manual UTC offset (single source of truth for the whole script) |
| Dividers | Vertical divider lines/labels |
| Dealing Range | Show/hide, extend toggle, color |
| True Opens | Show/hide drawn markers, history length, color |
| Trend Engine | Master enable toggle |
| SSMT Settings (legacy) | Symbol picker feeding HTF Alignment + tCISD reference search only |
| SMT Visual / Gates / Render / Session Filter / Asset Pair Resolver | Full GXV SMT engine configuration |
| HTF SSMT -> MTF tCISD | Optional higher-timeframe alignment gate |
| Turtle Soup | Swing lookback, markers, multi-timeframe overlays |
| tCISD Settings | Arming, confirmation, and display behavior |
| Dashboard | Show/hide |
| Layout / Timeframes / PSP Settings / Trace Lines / (90m, 6H, Daily) Colors / Fair Value Gap & Sweeps | HTF Candle Panel (bonus module) |

---

## Credits & License

This script is released under the **Mozilla Public License 2.0** (https://mozilla.org/MPL/2.0/).

It embeds the **SMT range reconstruction model, deterministic range-key logic, cleanup authority model, Auto Session behavior, and Aesthetic Bypass system** from **GXV SMT_Divergences[LITE✦]**, © Gregorius_XV (TradingView: **Gregorius_XV**), used and extended with attribution per that work's own MPL-2.0 redistribution terms. It also imports the published **`GXVSanctuary/AssetCorrelationUtils/5`** library, by the same author, for live correlated-asset pair resolution.

If you fork, port, or republish this script, please preserve credit to Gregorius_XV for the above components, per the original license terms.

---

## Disclaimer

This indicator is a technical-analysis and charting tool. It does not constitute financial advice, and nothing it displays — dividers, ranges, true opens, SMT/CISD markers, or trend state — should be treated as a signal to trade without your own independent judgment and risk management. Past patterns have no guaranteed relationship to future price behavior.
