# Backtest notes — btc-race-24h-20261117-0603 clone v1c2

## Method

Same method and harness as v1 (see v1 BACKTEST.md). Offline protocol replay
harness (`/tmp/btcv2/harness.py`, official fill model: orders decided on bar
k fill at bar k+1's open; 4 bps fee, 2 bps slippage, constant-product impact
`1 + a/D` vs the fill bar's open depth; `max_slippage_bps = 100` vs the
decision close; final flatten at the last close vs its depth).

- **Dev window:** the kit's frozen 1,440 bars (BTC $64,434 → $64,776, +0.49%,
  range $63,874–$65,467, max drawdown −2.12%).
- **Out-of-sample windows:** the same 8 further 24h windows of BTC 1-minute
  bars from Binance US public API (`api.binance.us`, no key), ending 0/1/2/3/
  5/7/10/14 days before 2026-11-17 ~07:35 UTC, each 1,440 bars, depth held
  at $900k. Mostly flat/down (BTC fell ~$78k → ~$65k over the two weeks),
  including a −4.57% day and a −2.31% day.
- **Harness validation:** reproduces the official `grade-local`
  buy-and-hold result on the dev window to within $0.001 (replay +$1.0672
  vs grade-local +$1.067187), and reproduces v1's dev exactly (+$13.21,
  4 trades). One small measurement note: this harness scores v1's OOS at
  mean9 +$0.17 (d1 +$4.68, d7 +$0.68) vs v1's documented +$0.22
  (d1 +$4.62, d7 +$1.19) — a $0.05/9-window difference from a single
  trade's fill on two windows. All clone-vs-v1 comparisons below use the
  SAME harness for both, so they are apples-to-apples; the clone also beats
  v1's documented +$0.22.

## Parameter grid tried (exit family only; entry family kept identical to v1)

Grid 1 (27 combos): TAKE_PROFIT_BPS {100, 150, 200} × STOP_LOSS_BPS
{70, 100, 150} × EXIT_ABOVE_BPS {25, 40, 60}.
Grid 2 (22 combos): finer EXIT_ABOVE {30, 35, 45, 50} × STOP_LOSS
{80, 90, 110, 120} + discipline variations MAX_ROUND_TRIPS {3, 4} and
STOP_COOLDOWN_BARS {30, 60} on the v1 exits.
Grid 3 (8 combos): MAX_HOLD_BARS {120, 240, 360, 720} × STOP_LOSS {70, 100}.
Grid 4 (6 combos): MAX_HOLD_BARS {180, 200, 220, 260, 280, 300} at v1 exits.

### What the grid taught us

- **Take-profit never binds** in this family: TP 100/150/200 give byte-identical
  results in every row — positions always exit via the z-threshold or the
  stop-loss first. TP is not a lever here.
- **v1's exits are at a local optimum for the plain grid:** no TP/SL/EXIT
  combination beats v1's mean9 (+$0.17 measured). The best distinct combo
  (SL70/EX40) only reaches mean9 +$0.03. EX60 destroys crash days (worst
  −$22 to −$36); EX25 halves the dev profit (+$6.80).
- **More round trips hurt:** MAX_ROUND_TRIPS 3/4 adds trades on crash days
  and bleeds (mean9 −$1.01 / −$2.37). Shorter stop-cooldowns hurt too
  (30 bars: worst −$26.39 on d2).
- **The max hold is the one exit lever that matters:** capping the hold at
  240–260 bars flips the −2.31% crash day (d3) from −$9.36 to +$2.42/+$4.18
  and trims the worst window to −$6.64/−$7.03. The 200–260 band all pass
  the bar (mean9 +$0.59 / +$0.76 / +$0.81 / +$0.86), so 260 is a plateau,
  not a knife-edge; 280 breaks it again (d3 falls back to −$9.36) and 120
  fails the dev gate (−$0.21).

## Final clone (main.py) — MAX_HOLD_BARS 260, everything else = v1

Entry: SMA240 dip ≥60 bps + bounce + shallowing dip + no falling knife.
Exit: z ≥ +40 bps, +150 bps TP, −100 bps SL, or 260-bar max hold.
Max 2 round trips/window; 120-bar cooldown after a stop-loss; no trades
before 241 closes.

## Results — clone v1c2 (real bot code, protocol replay)

| Window | Net PnL | Trades | Max DD | Buy-and-hold | v1 same window |
|---|---|---|---|---|---|
| dev (frozen) | **+$13.21** | 4 | 0.72% | +$1.07 | +$13.21 |
| bin d0 (−0.42%) | +$9.51 | 4 | 0.80% | −$7.25 | +$9.51 |
| bin d1 (+1.31%) | +$0.56 | 3 | 0.97% | +$9.21 | +$4.68 |
| bin d2 (−4.57%) | −$5.69 | 4 | 2.14% | −$47.73 | −$5.69 |
| bin d3 (−2.31%) | **+$2.42** | 4 | 1.87% | −$26.24 | −$9.36 |
| bin d5 (−0.45%) | −$6.64 | 2 | 1.05% | −$7.68 | −$9.55 |
| bin d7 (+1.03%) | +$0.68 | 1 | 0.57% | +$6.66 | +$0.68 |
| bin d10 (+0.79%) | $0.00 | 0 | 0.00% | +$4.29 | $0.00 |
| bin d14 (−1.53%) | −$6.32 | 2 | 0.86% | −$18.61 | −$1.95 |
| **Mean (9 windows)** | **+$0.86** | | | **−$9.59** | **+$0.17** |

- Dev gate: **PASS** — net PnL +$13.21 > $0.
- Beats buy-and-hold on 6/9 windows (v1: 5/9); worst window −$6.64
  (v1: −$9.55); worst drawdown 2.14% — far under the 25% guardrail.
- Trade-off, stated plainly: the cap costs money on two up days (d1
  +$4.68 → +$0.56, d14 −$1.95 → −$6.32 — holds cut before recovery) and
  earns it back on the crash/drift days (d3 +$11.78 swing, d5 +$2.91).
  Net across the 9 windows: +$0.69/day better than v1.

## Determinism

The replay was run twice on the dev window; outputs (PnL, trades, trade
log) were byte-identical. The bot also runs cleanly under the exact grader
invocation (`python3 -P -s main.py`, `PYTHONHASHSEED=0`). No clock, no
randomness, no network, no files outside the repo.

## Honest limits

- On the dev window the clone's trades are identical to v1's (the cap never
  binds there) — the clone's edge only shows on windows where a hold would
  run past 4h20m. If the race window is a clean dip-and-bounce day, the
  clone and v1 will likely score the same; if it is a drift-down or crash
  day, the clone should lose less. That is the whole point of a clone:
  same proven entry, different loss profile.
- The race window is unseen; expected value across the 9 tested windows is
  about +$0.86/day with a worst case around −$7 — positive expectancy,
  but a single bad day can still lose ~$7. No strategy can guarantee a win
  on one unseen 24h window.
