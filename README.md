# mt5-gold-scalper-ea

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-MetaTrader%205-blue.svg)](https://www.metatrader5.com/)
[![Language](https://img.shields.io/badge/Language-MQL5-orange.svg)](https://www.mql5.com/)
[![Symbol](https://img.shields.io/badge/Symbol-XAUUSD%20%2F%20GOLD-yellow.svg)](https://github.com/MT5-Automation-Labs/mt5-gold-scalper-ea)

MetaTrader 5 Expert Advisors that scalp gold (`GOLD` / `XAUUSD`) on M1 by trading statistical extremes back toward the mean.

The recommended build is **TickRobust**. It decides once per closed bar, refuses trades whose expected snap-back does not clear round-trip cost, and places the take-profit as a broker-side limit at the moving average. The other files are earlier mean-reversion builds and a dual-strategy variant kept for comparison.

## Which file to run

| Build | Source | Magic | Use it when |
| --- | --- | --- | --- |
| **TickRobust** (recommended) | [`Experts/XAU_Quant_Reversion_TickRobust.mq5`](Experts/XAU_Quant_Reversion_TickRobust.mq5) | `777555` | Live trading and Every Tick testing |
| M1 OHLC | [`Experts/XAU_Quant_Reversion_M1_OHLC.mq5`](Experts/XAU_Quant_Reversion_M1_OHLC.mq5) | `777333` | Research on the bar-close Z-score model with spread, velocity, and trade-log filters |
| M1 Every Tick | [`Experts/XAU_Quant_Reversion_M1_EveryTick.mq5`](Experts/XAU_Quant_Reversion_M1_EveryTick.mq5) | `777333` | Comparing the original tick-by-tick Z-score scalper |
| Dual strategy | [`Experts/XAU_Quant_Reversion_Breakout.mq5`](Experts/XAU_Quant_Reversion_Breakout.mq5) | `777333` / `777444` | Running mean reversion and a Donchian breakout on one chart |

`M1 OHLC`, `M1 Every Tick`, and the mean-reversion leg of the dual strategy share magic `777333`. Attach only one of those to an account, or change `InpMagic`, or they will manage each other's positions.

All four expect an **M1** chart. The symbol input defaults to `GOLD` and falls back to `XAUUSD` when `GOLD` is not in the terminal.

## Requirements

- MetaTrader 5 with MetaEditor
- A gold symbol quoted in points (`SYMBOL_POINT`). Stops in this project are **points**, not pips
- The terminal economic calendar, if the news filter stays enabled
- AutoTrading enabled on the account and on the chart

## Installation

1. Copy the `.mq5` files from [`Experts/`](Experts/) into the terminal data folder: `MQL5/Experts/`.
2. Open each file in MetaEditor and compile (`F7`). A successful compile writes a matching `.ex5` beside the source.
3. In MetaTrader 5, attach the Expert Advisor to a **GOLD** or **XAUUSD** chart set to **M1**.
4. Confirm the inputs, then enable **AutoTrading**.

Compiled binaries are not stored in this repository.

## Repository layout

```text
Experts/                                   Deployable Expert Advisors
  XAU_Quant_Reversion_TickRobust.mq5       Closed-bar mean reversion (recommended)
  XAU_Quant_Reversion_M1_OHLC.mq5          Bar-close Z-score build
  XAU_Quant_Reversion_M1_EveryTick.mq5     Tick-level Z-score build
  XAU_Quant_Reversion_Breakout.mq5         Mean reversion plus Donchian breakout
reference/
  IdealEA_1mOHLC.txt                       Earlier source snapshot, not for deployment
```

Renames from the previous flat layout:

| Previous path | Current path |
| --- | --- |
| `XAU_Quant_Reversion_TickRobust.mq5` | `Experts/XAU_Quant_Reversion_TickRobust.mq5` |
| `XAU_Quant_Reversion_m1_OLHC.mq5` | `Experts/XAU_Quant_Reversion_M1_OHLC.mq5` |
| `XAU_Quant_Reversion_m1_EveryTick.mq5` | `Experts/XAU_Quant_Reversion_M1_EveryTick.mq5` |
| `XAU_Quant_Reversion_Breakout.mq5` | `Experts/XAU_Quant_Reversion_Breakout.mq5` |
| `IdealEA_1mOHLC.txt` | `reference/IdealEA_1mOHLC.txt` |

The OHLC filename spelling is corrected from `OLHC` to `OHLC`. Recompile after updating an existing terminal copy.

## Strategy

Price is scored against a simple moving average:

```text
Z = (price - SMA) / StdDev
```

A large negative Z is a downside stretch (buy candidate). A large positive Z is an upside stretch (sell candidate). Entries are allowed only while the market looks like a range: ADX is capped, ATR must sit inside a band around its recent average, spread is capped, and the session is limited to broker hours 10:00–20:00.

Shared protective behaviour on every build:

- High-importance **USD** calendar events only. New entries pause before and after the event. Open trades can be flattened before the release.
- A daily loss limit, expressed as a percent of the day's starting balance. After it trips, positions managed by that magic are closed and new entries stop until the next day.
- Friday flatten at `InpFridayCloseHour` (default 20:00) and a weekend block on the three single-strategy files.
- Position size from the stop distance and a risk percent of balance, checked against free margin before `OrderSend`.

### TickRobust

TickRobust is the build written so Every Tick results and closed-bar results describe the same decisions.

Decisions that can wait — entry, Z-score exit, time exit, breakeven, and take-profit retargeting — run **once, at the open of a new bar, from shift-1 values**. Intra-bar, only the server-side stop and take-profit can change the position. Daily-loss, weekend, and pre-news closes still run on the tick, because those are account-safety actions.

**Entry, all required**

| Rule | Default |
| --- | --- |
| Closed-bar Z at or beyond the threshold | `InpEntryZ = 2.2` |
| Turn confirmation: the last close stopped extending the stretch | `InpRequireTurn = true` |
| Closed-bar ADX at or below the cap | `InpADXMax = 22` |
| ATR between 0.4× and 2.0× its 50-bar average | `InpMinATRRatio` / `InpMaxATRRatio` |
| Spread at or below 50 points | `InpMaxSpreadPts = 50` |
| StdDev at least 3× full round-trip cost | `InpMinSDCostMult = 3` |
| Distance from fill to the mean at least 4× that cost | `InpMinTPCostMult = 4` |
| Optional alignment with the closed H1 SMA | period `50`; set `InpFilterPeriodH1 = 0` to disable |
| Outside the news window, inside the session, and past any loss cooldown | cooldown `15` minutes |

Round-trip cost is the current spread plus `InpExtraCostPts`. Set `InpExtraCostPts` to the broker's round-trip commission converted into gold points. Leave it at `0` when the account is spread-only.

**Exit, in order**

1. Server-side limit take-profit at the SMA. A limit fill does not pay the spread on the way out. Each new bar retargets that price to the latest closed-bar SMA, and will not pull it inside a floor that still covers cost.
2. Closed-bar Z back inside `±0.2` (`InpExitZ`), if the limit has not filled.
3. Time exit after `40` bars (`InpMaxHoldBars`).
4. Breakeven after `1.0 × ATR` of open profit (`InpBreakevenATR`).
5. Server-side stop at `max(800 points, 2.5 × ATR)`.

### M1 OHLC

Bar-close Z-score mean reversion with the filters that were added while tuning the OHLC model: a 20-tick spread average, an ATR-relative spread cap (12% of M1 ATR), a short price-velocity block, a loss cooldown, ADX, a volatility-adjusted entry Z, an H1 SMA filter, and a CSV trade log.

Default entry Z is `2.0`, raised by `0.5` when ATR exceeds 1.5× its 100-bar average. The hard stop is `800` points and the hard take-profit is `1500` points. The working exit is a return of Z to `±0.5`, with an ATR trail updated on a new bar. Maximum hold is `30` minutes.

Judge this file on **1 minute OHLC** modelling. TickRobust is the file intended for real-tick tests.

### M1 Every Tick

The earlier scalper. It evaluates Z from the live bid on every tick, enters beyond `±2.0`, and exits on a Z return to `±0.5`, an ATR trail, the hard stop (`800`), or the hard take-profit (`1500`). It has the session window, spread cap, H1 filter, news filter, and daily loss limit. It does not have the ADX gate, cost gate, turn confirmation, velocity filter, or loss cooldown added in the later files.

### Dual strategy

Two strategies on one chart, each with its own magic and position limit.

| Leg | Magic | Entry | Stop / hard target / trail |
| --- | --- | --- | --- |
| Mean reversion | `777333` | Z beyond `±2.4` while ADX is below `20` | `800` / `1500` / `1.5 × ATR` |
| Trend breakout | `777444` | Close beyond a `30`-bar Donchian band while ADX is at least `30`, EMA `50` agrees, and the DI spread is at least `5` | `1000` / `2000` / `2.0 × ATR` |

The breakout leg enters on a new bar and waits `10` bars after a position closes. Mean reversion can scale out half the position when Z is inside `±0.2`, then trail the remainder. Either leg can be switched off with `InpUseReversion` or `InpUseBreakout`.

With dynamic risk enabled, the breakout leg uses 30% of the mean-reversion tier. With it disabled, the inputs are `InpRiskPct` (default `10`) and `InpTBRiskPct` (default `3`).

## Risk tiers

When `InpUseDynamicRisk` is `true`, risk per trade and the daily loss limit follow account equity:

| Equity | Risk per trade | Daily loss limit |
| --- | --- | --- |
| Under $500 | 10% | 25% |
| $500 to under $2,000 | 7% | 20% |
| $2,000 to under $5,000 | 5% | 15% |
| $5,000 to under $20,000 | 3% | 10% |
| $20,000 and above | 1.5% | 7% |

Set `InpUseDynamicRisk` to `false` to use the fixed inputs instead. Those defaults are **2%** risk and **15%** daily loss on TickRobust, and **10%** risk and **20%** daily loss on the other single-strategy files.

Ten percent of equity per trade can wipe a small account in a short losing sequence. Use a demo account and a risk setting you can afford to lose before attaching any of these files to live capital.

## Backtesting

- Test **TickRobust** with **Every tick based on real ticks**. Because entries and discretionary exits are taken from closed bars, OHLC and real-tick results should stay close. A large gap usually means the spread, commission, or symbol specification does not match the inputs.
- Set `InpExtraCostPts` before reading a TickRobust report.
- Expect fewer TickRobust trades than the earlier files. The cost gate drops stretches that do not clear the spread.
- Test `M1 OHLC` on **1 minute OHLC**. Its tick-path behaviour is not the acceptance test for that file.
- Model spread explicitly. A zero-spread test will overstate every market-order variant in this repository.

## Disclaimer

These Expert Advisors are published for education and research. Leveraged gold trading can lose the entire deposit. Nothing in this repository is a promise of profit, and historical tests do not predict live results. You are responsible for the symbol settings, the risk inputs, and the decision to run a strategy on a live account.

## License

Released under the [MIT License](LICENSE).

## Author

**Neo Malesa** — [n30dyn4m1c](https://github.com/n30dyn4m1c)
