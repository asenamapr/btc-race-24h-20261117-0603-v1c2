[![make.fun · btc-race-24h-20261117-0603](https://make.fun/badge/btc-race-24h-20261117-0603.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![rank](https://make.fun/badge/btc-race-24h-20261117-0603/asenamapr/btc-race-24h-20261117-0603/rank.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![reward](https://make.fun/badge/btc-race-24h-20261117-0603/asenamapr/btc-race-24h-20261117-0603/reward.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)

# BTC Mean-Reversion Bot (SMA240 Dip Buyer — 4h Max Hold) — make.fun `btc-race-24h-20261117-0603`

Entry for the [make.fun bounty](https://make.fun/bounty/btc-race-24h-20261117-0603): Race a BTC trading bot live for 24 hours. Parameter-varied clone of the v1 "SMA240 Dip Buyer" (same entry family; the one change is a 260-bar maximum hold that cuts drift-down bleed — see BACKTEST.md).

Entry for the [make.fun bounty](https://make.fun/bounty/btc-race-24h-20261117-0603): Race a BTC trading bot live for 24 hours.

A bot that trades cbBTC (CBBTC) against USDC on 1-minute bars with $1,000 of
paper money, for the make.fun competition "Race a BTC trading bot live for
24 hours". Python 3.13, standard library only.

## Strategy

1-minute BTC noise (~6 bps/min) is far below the ~30 bps round-trip cost
(4 bps fee + 2 bps slippage + constant-product impact), so 1-minute signals
lose. This bot trades **multi-hour mean reversion** instead:

- **Anchor:** the 240-minute (4h) SMA of closes.
- **Entry** (flat only, full cash): close ≥ 60 bps **below** the 240-min SMA,
  **and** close above the 15-min SMA (short-term bounce), **and** the dip is
  shallowing (today's z-score above the z-score 15 bars ago), **and** the
  60-min SMA sits no more than 100 bps below the 240-min SMA (no falling
  knife).
- **Exit** (whole position): close recovers to 40 bps **above** the 240-min
  SMA, **or** +150 bps over the fill price (take profit), **or** −100 bps
  under it (stop loss), **or** held 260 bars (4h20m max hold — the clone's
  one change vs v1's 1440).
- **Discipline:** at most **2 round trips** per window (repeated dip-buys in
  a downtrend bleed fees) and a **120-bar cooldown** after a stop-loss exit.
- **Cold start:** no trading until 241 closes have been seen, so every
  indicator is fully formed.

Orders are full-size (up to 100% of equity), `max_slippage_bps = 100`, one
position at a time, long only. Fully deterministic: plain float arithmetic
over the bars as they arrive — no clock, no randomness, no network, no files.

## Results (see BACKTEST.md for full detail)

| Window | Net PnL | Trades | Max DD | Buy-and-hold |
|---|---|---|---|---|
| Dev (frozen) | **+$13.21** | 4 | 0.72% | +$1.07 |
| 8 other recent 24h BTC windows | mean over all 9 windows **+$0.86**, beats B&H 6/9, worst −$6.64 | — | ≤ 2.14% | mean −$9.59 |

The bot beats buy-and-hold on the dev window by ~$12 and on most other
windows, and loses far less than buy-and-hold on crash windows
(e.g. +$2.42 vs −$26.24 on a −2.31% day). The 260-bar max hold is what
fixes v1's worst habit: holding a losing dip-buy to the end of a
drifting-down day (v1's worst window was −$9.55 on exactly that). It is
not magic: on strongly down-trending days a long-only bot still loses a
little — it just loses much less than holding, because it trades rarely,
stops after 2 round trips, and never holds a position longer than 4h20m.

## Files

- `main.py` — the bot (entrypoint, run as `python3 -P -s main.py`)
- `vendor/makefun_bars.py` — byte-identical copy of the kit's Python SDK shim
- `BACKTEST.md` — backtest notes (dev + out-of-sample windows, determinism)
- `LICENSE` — MIT
- `.makefun/owner.json` — claim file for the payout wallet

## Run / grade locally

From the kit directory (the kit is what the server grades with):

```sh
node cli/makefun.mjs grade-local --kit . --bot <path-to-this-repo> --driver docker
```

Note: on a machine without Docker, `grade-local --driver process` cannot run
a Python bot (the process sandbox is Node-only). The numbers above were
produced with an offline replay harness implementing the harness fill model
exactly (fee 4 bps, slippage 2 bps, constant-product impact vs bar open
depth); that harness reproduces the official `grade-local` buy-and-hold
result on the dev window to within $0.001 (+$1.0673 vs +$1.067187), so the
+$13.21 dev figure is trustworthy to within a cent. Re-run the official
grade before submitting.
