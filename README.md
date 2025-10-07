# SPY 0DTE Backtester and Live/Paper Runner

A single‑file Python system for backtesting and live/paper trading SPY 0DTE option alerts.
It fetches 1‑minute option/underlying data (Polygon) or uses IBKR (ib‑insync) for backtest/live,
selects contracts by premium bands/OTM preferences, simulates realistic fills (ask‑like entry,
bid‑like exits), and manages positions with TP/SL, trailing, and a configurable parabolic tail‑capture.

This README is written to showcase the project for hiring managers and as a practical guide for users.

## Highlights
- Polygon and IBKR backends (switchable by `--mode`).
- Two engines:
  - One‑at‑a‑time sequential backtest (`poly-backtest`, `ib-backtest`).
  - Per‑alert evaluation (`poly-all`, `ib-all`).
- Contract selection by premium band, OTM/ITM preferences, OTM strike shortcuts.
- Realistic fills (ask‑like entry, bid‑like exits) with configurable spread and entry limit offsets.
- Position management:
  - Single TP/SL or multi‑TP splits per trade.
  - Pyramiding adds (combined or split child trades) with optional multi‑TP on adds.
  - Exit on opposite alert (with optional reverse into the new alert).
  - Add on aligned alert (same‑direction add on signal).
- Parabolic tail capture:
  - Early activation (several modes, incl. pure return threshold).
  - Fixed gap or dynamic gap schedule (step/linear interpolation) that trails off the peak return.
  - Optional verbose activation banner and enriched exit attributions.
- What‑if modes that hold to close, disable TP/SL/opposite and compute day extremes.
- Timing diagnostics (`--timings`) and soft‑fail networking (`--net-softfail`).
- Clean one‑line CLI errors (no full usage dump) for typos.

## Quick Start

Prereqs:
- Python 3.10+ (tested on 3.11/3.12/3.13)
- `pip install requests pandas python-dateutil`
- Polygon API key export: `setx POLYGON_API_KEY your_key_here` (Windows) or `export POLYGON_API_KEY=...`

Run a minimal Polygon backtest (sequential engine):

```
python spy_0dte_backtest.py \
  --mode poly-backtest \
  --alerts alerts.csv \
  --start 2025-07-31 --end 2025-07-31 \
  --budget 150 --band-low 0.07 --band-high 0.12 --moneyness any \
  --hold 390 --fill-model realistic \
  --verbose-skip --net-softfail --print-exits
```

Outputs (written to `bt_out/` by default):
- `trades.csv`: every trade with entry/exit details, pnl, extremes, legs.
- `daily_summary.csv`: per‑day aggregates.
- `summary.txt`: total trades, win rate, P&L, profit factor, max drawdown, expectancy.
- `timings.json` when `--timings` is used.

## Modes
- Backtest (sequential, one‑at‑a‑time): `poly-backtest`
- Per‑alert (independent): `poly-all`
- What‑if: `poly-whatif`, `poly-whatif-all`
  - What‑if preloads: hold to EOD, disable TP/SL/opposite, compute day extremes.
- Live/Paper IB: `ib-paper`, `ib-live` (add `--place-orders` and `--confirm-live "I UNDERSTAND"` for live orders)

> Note: IB historical backtest/per‑alert modes have been removed. Use `poly-*` for backtests and `ib-paper`/`ib-live` for execution.

> Tip: `--trade-all` is informational and does not turn `poly-backtest` into per‑alert. Use `--mode poly-all` for per‑alert.

## Contract Selection
- Premium band: `--band-low`, `--band-high` on the option’s minute mid.
- Fill model at the alert minute to compute entry affordability:
  - `--fill-model realistic` (ask‑like entry, bid‑like exit, controlled by `--spread-pct`)
  - `ba_spread`, `ba_hl`, `mid_slip`
- Moneyness & OTM helpers:
  - `--moneyness any|itm|otm`
  - `--otm-strikes N` pick N‑th OTM strike from UL (overrides band when affordable at entry)
- Selection falls back to nearest‑to‑band if no in‑band affordable contract.

## Entries & Offsets
- Optional entry limits: place X dollars below the basis.
  - `--entry-offset-cents 0.02`
  - `--entry-offset-basis ask|mid`
- Offset is clamped to not go below the minute low and not below $0.01.

## Position Management
- TP/SL:
  - Single: `--tp X`, `--sl Y`, basis via `--tp-basis avg|initial`.
  - Multi‑TP: `--tp-multi "1.5@50,2.0@50"` or `"1.25,1.5,2.0"`.
  - Parent rides, adds bank only: `--tp-adds-only`.
  - Adds‑only schedule: `--tp-multi-adds` or `--pyr-tp-multi "..."` with `--pyramid-mode split`.
- Pyramiding:
  - Threshold: `--pyramid-threshold-pct 0.65` (vs base entry).
  - Budget/max: `--pyramid-budget`, `--pyramid-max-adds`.
  - Split vs combined: `--pyramid-mode split|combined` (split spawns independent child trades).
  - Same contract: `--pyramid-force-same`.
  - Add on aligned alert (ignore threshold): `--add-on-aligned`.
- Opposite alert exits / reverse:
  - Exit: `--exit-on-opposite` with `--exit-opposite-no-high N` minutes.
  - Reverse immediately: `--reverse-on-opposite` (re‑enter on the opposite alert at the exit minute, backtest).

## Parabolic Tail Capture
- Turn on: `--parabolic-capture`.
- Activation: `--parabolic-activate-mode classic|either|two_up_only|return|ret_or_classic`
  - Return mode threshold: `--parabolic-activate-return X` (e.g., 0.10 = +10%)
  - Activation basis: `--parabolic-activate-basis initial|avg|trail`
  - After add only: `--parabolic-after-add`
- Floors:
  - Start floor: `--parabolic-trail-start` (0 allowed)
  - Tighten: `--parabolic-tighten-gain`, `--parabolic-trail-tight`
  - Floor min: `--parabolic-trail-floor` (0 allowed)
- Gap trailing (peak − gap):
  - Fixed gap: `--parabolic-gap-pct`
  - Dynamic schedule: `--parabolic-gap-schedule "ret:gap,..."` with `--parabolic-gap-interp step|linear`
    - Below first threshold uses `--parabolic-gap-pct` as fallback if set.
  - Effective stop = max(all candidates), so it never decreases.
- Observability:
  - Activation banner: `--parabolic-verbose` prints schedule level, chosen gap, floor, and stop multiple.
  - Exit detail: SL_PARABOLIC_GAP(gap=G, basis=B) when gap stop triggered.

## Engine Behavior
- poly-backtest (sequential):
  - One position at a time; skips alerts until current trade exits.
  - Use `--print-exits` to print per‑trade exit lines during the run.
- poly-all (per‑alert):
  - Evaluates each alert independently. Useful for quick exploration and chatty logs.

## Data & Environment
- Polygon: `POLYGON_API_KEY` env var required for `poly-*` modes.
- IBKR: TWS/Gateway for `ib-*` modes; see `--ib-host`, `--ib-port`, `--ib-account`.

## Timings & Robustness
- `--timings`: write `timings.json` and print HTTP counts/latencies/backoffs.
- `--net-softfail`: treat network errors as empty data and continue instead of aborting.
- Quiet errors: CLI prints one‑line errors for bad flags (no usage spam).

## Examples
- Early parabolic control with return activation:
```
python spy_0dte_backtest.py --mode poly-backtest --alerts alerts.csv \
  --start 2025-07-31 --end 2025-07-31 --budget 150 --band-low 0.07 --band-high 0.12 \
  --tp 2.0 --sl 0.8 --fill-model realistic \
  --parabolic-capture --parabolic-activate-mode return --parabolic-activate-return 0.15 \
  --parabolic-trail-start 0.30 --parabolic-gap-schedule "2.0:1.70,3.0:2.20,5.0:3.20" --parabolic-gap-interp linear \
  --trail-basis initial --parabolic-activate-basis initial --print-exits
```

## Code Map (single file)
- Data fetchers: Polygon (`fetch_underlying_bars`, `fetch_option_bars`) and IB (`ib_*` helpers).
- Selection: `pick_option_for_alert`, `pick_option_for_add`.
- Simulation: `simulate_trade` (TP/SL, trailing, parabolic, pyramiding, exits, what‑if).
- CLI/Main: `main()` builds the parser, sets globals, runs the chosen engine, writes outputs.

## Project Quality & Next Steps
This is a feature‑rich single‑file trading/backtest system with:
- Realistic market mechanics (bid/ask modeling, per‑minute data).
- Robust strategy controls (parabolic schedule/interp, offsets, reverse, adds/multi‑TP).
- Diagnostics (timings, banners, quiet CLI errors).

Areas to improve (great next steps for a hiring panel):
- Split into modules (`data/`, `engine/`, `strategy/`, `cli/`) and add type hints throughout.
- Unit tests for selection, fill models, parabolic schedule, and exit attributions; smoke tests for engines.
- Requirements/lockfile and CI (GitHub Actions) for lint/format/test.
- Local data cache to reduce API calls; optional async batch fetches; better retry shaping.
- Structured logging (JSON) and richer CSV schema docs.
- Config presets (e.g., `profiles/*.yaml`) for reproducible runs.

## License
No license is provided here. Add one if you plan to open‑source the repo.
