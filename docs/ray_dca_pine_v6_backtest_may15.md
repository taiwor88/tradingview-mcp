# Ray DCA Pine v6 Backtest — May 15, 2026 (V2 of LP7 validation)

**Status:** ❌ **V2 FAIL — divergence between stated spec and live-bot behaviour**

This is the V2 validation gate deliverable. It documents a *finding*, not a
victory: a faithful line-by-line translation of the Ray DCA strategy spec
into Pine v6 does NOT reproduce the live bot's observed behaviour. The Pine
strategy gets stuck in cap-hit since October 2025 and never trades in the
W4 window, while the live bot delivered +$173.89 on BTC in the same window
per the Ray DCA audit.

That gap *is* the finding. The stated mechanics are incomplete.

---

## TL;DR

- **V2 FAIL**: Pine v6 translation of stated Ray DCA spec produces a
  stuck cap-hit cycle, **0 closed trades in W4** (Apr 1 → May 14 2026),
  ~−$181 unrealised P&L.
- **Live bot during same window**: +$173.89 BTC profit (Ray DCA audit on
  ~$501 capital base = +34.7%).
- **Conclusion**: the stated mechanics passed to Pine v6 are
  *incomplete*. The live bot has unstated logic that prevents (or breaks
  out of) the cap-hit-and-stuck state my Pine v6 fell into.

The Pine v6 strategy *compiles cleanly and executes 683 historical
trades* (Jan 2024 → Sep 2025), so the entry/exit mechanics are
functional in isolation. The strategy is not broken — it's accurately
implementing an incomplete spec.

---

## Strategy implementation

**Pine v6 script** saved to TradingView cloud:
- **Title**: `Ray DCA v1`
- **ID**: `USER;5b11fa77844a4787bfd6d638fc261f87` v1.0
- **Source file**: `.ray_dca_v1.pine` (in repo root, committed alongside
  this doc)

**Key parameters** (defaults per stated spec):

| Param | Value |
|---|---|
| `dip_threshold` | 0.01 (1%) |
| `tp_threshold` | 0.008 (0.8%) |
| `max_layers` | 10 |
| `base_position_usd` | $50 |
| `lookback_24h` | 24 bars (= 24h on 1h chart) |
| `initial_capital` | $10,000 |
| `commission` | 0.04% per side (BINANCE perp taker) |
| `pyramiding` | 10 |
| `process_orders_on_close` | true |

**Mechanics implemented (from spec)**:

```
Entry:   close <= max(high, 24) × (1 - 0.01)
         AND current_layers < 10
         → strategy.entry; push (price, size) into layer arrays

Exit:    weighted_avg = Σ(layer_price × layer_size) / Σ(layer_size)
         tp_level     = weighted_avg × (1 + 0.008)
         IF close >= tp_level → strategy.close_all, reset state

No stop loss. Max-layers cap is the only risk control.
```

Custom Pine `table.new` table reads back cycles_completed,
cap_hits, avg_cycle_duration, open_layers, net_pct (read via
`data_get_pine_tables` after backtest).

---

## Full backtest data (BTCUSDT.P 1h, 2024-01-01 → 2026-05-15, 866 days)

### Aggregate metrics (entire backtest)

| Metric | Value |
|---|---|
| Total closed trades | 683 |
| Winning trades | 652 (95.5%) |
| Losing trades | 31 |
| Net profit | +$386.08 |
| Net profit % (on $10K cap) | +3.86% |
| Gross profit | +$398.42 |
| Gross loss | −$12.35 |
| Profit factor | 32.27 |
| Commission paid | $27.66 |
| Max drawdown | −$259.31 (−2.50%) |
| Sharpe ratio | −0.224 |
| Sortino ratio | −0.253 |
| Buy & hold return | +75.7% (over same period) |
| Open layers (at end of run) | 10 |
| Open unrealised P&L | ~−$181 |

The win-rate looks great (95.5%) but the Sharpe/Sortino are negative
because the long tail of stuck-open exposure since Oct 2025 dominates
the equity curve. Buy-and-hold over the same window: +75.7% on the
underlying — the strategy massively under-performs hold-only because
it's been stuck in cap-hit for ~7 months.

### Monthly trade distribution

| Month | Closed trades |
|---|---|
| 2024-01 | 21 |
| 2024-02 | 144 |
| 2024-03 | 116 |
| 2024-04 | 10 |
| 2024-05–2024-09 | 0 |
| 2024-10 | 10 |
| 2024-11 | 158 |
| 2024-12 | 114 |
| 2025-01 | 10 |
| 2025-02–2025-04 | 0 |
| 2025-05 | 19 |
| 2025-06 | 0 |
| 2025-07 | 29 |
| 2025-08 | 22 |
| 2025-09 | 0 |
| 2025-10 | 30 |
| **2025-11–2026-04** | **0** ← stuck cap-hit |
| 2026-05 | 10 (marked-to-market open layers, NOT real closes) |

The Oct 2025 30-trade cluster is the strategy filling cap on the BTC
drawdown. From November 2025 onward, no closures because TP threshold
(weighted-avg × 1.008) was never hit again — price moved further away.

### W4 window (BTC, 2026-04-01 → 2026-05-14, 44 days)

| Metric | Value |
|---|---|
| Closed trades in window | **0** |
| Net profit | **$0** |
| Cycles completed | 0 |
| Cap hits | 0 |
| Open layers at window start | 10 (filled Oct 2025) |
| Open layers at window end | 10 (unchanged) |
| Unrealised P&L | ~−$181 |

---

## Cross-validation: BTC W4 vs Ray DCA audit

| | Audit (live bot, Aster) | Pine v6 (BINANCE, 1h, stated spec) |
|---|---|---|
| Window | ~31 days (Apr-May 2026) | 44 days (Apr 1 → May 14 2026) |
| Capital base | ~$501 | $10,000 |
| **Closed P&L** | **+$173.89** (+34.7%) | **$0** (0%) |
| Cycles completed | n/a (audit reports profit only) | 0 |
| Live state at window start | actively cycling | 10 layers stuck since Oct 2025 |

**Divergence magnitude**: live bot makes 35% in 31 days. Pine v6 makes
0% in 44 days. The difference is not "magnitude calibration" — it's a
structural disagreement. Pine v6 is *frozen*; the live bot is *active*.

**Likely source of divergence**: stated mechanics passed to Pine v6
omit something the live bot does. The omitted mechanic must be one
that *prevents the cap-hit-and-stuck state* (e.g. forces an exit, or
prevents all 10 layers filling in the first place, or uses different
TP logic that the stuck state would still satisfy).

---

## Hypothesis: which mechanic is missing?

Four candidates, each consistent with the observed divergence.

### 1. Per-layer TP instead of weighted-avg TP

The spec says exit fires when `close >= weighted_avg × (1 + 0.008)`. But
the live bot may TP **each layer independently** when its own entry
price reaches `layer_entry × 1.008`. With per-layer TP, the first layer
closes when price recovers 0.8% from L1's entry (a much lower bar than
recovering 0.8% above the weighted average of 10 layers stacked through
a drawdown).

Under per-layer TP, after Oct 2025 the strategy would have peeled off
layers one by one as price recovered intermittently, instead of waiting
for the entire stack to clear. This is the most likely candidate — it
*directly* explains why my version got stuck while the live bot kept
cycling.

### 2. 1h granularity vs live bot's 5-min cycle

The spec says "bot's actual 5-min cycle". Backtesting on 1h bars
collapses 12 intra-hour price moves into one candle. If the live bot
sees an intra-hour TP that the 1h candle doesn't show as a close,
many cycles will close in 5-min reality but not in 1h backtest.

This *contributes* to under-counting trades but probably can't account
for 7 months of zero closes if the weighted-avg TP threshold was never
breached on any 1h close in that window either. Verifiable: pull the
1h closes for Nov 2025 onward and check whether *any* exceeded the
stuck stack's `weighted_avg × 1.008`.

### 3. Minimum layer-spacing requirement

The spec entry condition is `close <= 24h_high × (1 - 0.01)`. Under
this rule, on a sharp drop the strategy can fill **all 10 layers in
just a few bars** — each subsequent bar's 24h_high is roughly the
same, so the entry level is roughly constant, and the strategy stacks
layers as long as price stays below the band. That's exactly what
appears to have happened in Oct 2025: cap filled fast, then price
moved against the stack.

The live bot may require **price to drop another N% from the previous
layer** before adding the next one (e.g. each layer is 1% below the
prior layer's entry, not 1% below the same 24h high). With that rule,
filling all 10 layers requires a 10% drawdown from L1, which is rarer
— and when it happens, the weighted average is *much lower*, so the TP
threshold is easier to hit on recovery.

### 4. Time-based exit / max cycle duration

The live bot may have a **cycle expiry** — e.g. if a cycle hasn't
TP'd within 30 days, close all layers at market regardless. The
spec explicitly says "No stop loss by design — Cap is the only risk
control". But "no stop loss" and "time-based exit" are different
things. A time-bounded exit isn't a stop-loss in the traditional
sense; it's risk management against the exact stuck state I hit.

If this rule existed, the Oct 2025 cap-hit cycle would have force-closed
~30 days later (Nov 2025) at a loss, freeing the strategy to start new
cycles in the months that followed.

---

## Acceptance table (V2 gate)

| # | Criterion | Status | Evidence |
|---|---|---|---|
| 1 | Pine v6 script compiles without errors | ✅ **PASS** | `pine_check` returned `error_count: 0, warning_count: 0`. Live backtest produces 683 closed trades — strategy is executing. |
| 2 | All 12 backtest runs complete | ❌ **FAIL (gate not reached)** | W4 smoke-test gate failed → 11 remaining cells not run, per spec gate rule. |
| 3 | BTC W4 positive return cross-validates audit | ❌ **FAIL** | 0 closed trades in W4 (44 days). $0 closed P&L vs audit's +$173.89. Strategy stuck cap-hit since Oct 2025. |
| 4 | ETH W4 + SOL W4 competitive vs BTC W4 | n/a — gate stopped | Phase 3 not reached. |
| 5 | WHERE-WRONG ≥ 5 caveats | ✅ **PASS** | 5 items documented below. |
| 6 | Counter-evidence ≥ 2 reasons | ✅ **PASS** | 2 reasons documented below. |

**Trial gate result: V2 does NOT validate live bot's BTC W4 +$173.89
result against Pine v6 backtest. Divergence identified, root cause
hypothesised, escalated to V2.1 with mechanics correction (Day 17).**

---

## WHERE-WRONG / counter-evidence to the FAIL itself

The FAIL itself rests on assumptions worth surfacing:

1. **The stated spec is incomplete.** This *is* the finding, but it's
   also a finding that needs corroboration. Pine v6 was translated
   line-by-line from the spec. If the spec were complete, Pine v6
   should reproduce the live bot. The fact it doesn't is the
   divergence — but the divergence's *interpretation* (incomplete
   spec) is itself a hypothesis. Could also be a subtle Pine
   translation bug I haven't spotted (entry condition `<=` vs `<`,
   off-by-one in 24h lookback, etc.).
2. **1h candle granularity vs live bot's 5-min cycle.** 12× coarser
   sampling. Intra-hour wicks that the live bot's 5-min cycle would
   trade on are invisible on 1h closes. This understates trade volume
   and may overstate cycle duration. Especially relevant for
   evaluating Hypothesis 1 (per-layer TP would fire often on 5-min
   resolution that the 1h backtest misses).
3. **Weighted-avg TP assumption.** The spec is explicit about
   weighted-average TP. If the live bot uses per-layer TP, the spec
   itself is wrong, and Hypothesis 1 above explains the divergence.
   Pine v6 faithfully implements the spec, so the issue isn't Pine v6
   — it's the spec input.
4. **Fee model: 0.04% taker per side, no funding.** Conservative
   BINANCE perp taker rate. Aster's actual fee schedule may differ.
   No funding rate modelled at all — over a 7-month stuck cap-hit,
   funding accumulates non-trivially and would worsen the unrealised
   P&L if Aster funding is positive in this window. The audit's
   +$173.89 is *after* fees and funding, so my unrealised −$181 is
   actually *better* than reality.
5. **Survivorship / data quality on TV BINANCE feed.** BTCUSDT.P data
   on BINANCE has been continuous for years, so survivorship isn't a
   factor for BTC specifically. Data quality (gaps, wrong wicks)
   could affect entry-fill prices, but not enough to explain 7 months
   of zero closures.

## Counter-evidence (≥2)

1. **The strategy COULD work if mechanics were complete.** The 683
   trades pre-Oct-2025 (Jan 2024 → Sep 2025) show the entry signal,
   layer accumulation, weighted-avg TP, and reset logic all execute
   correctly. 95.5% win rate over 683 trades with profit factor 32 in
   the periods the strategy *was* cycling. The Pine implementation is
   functional in isolation; what fails is the spec's robustness to
   prolonged drawdown (the Oct 2025 BTC drawdown was the first event
   in the backtest window large enough to lock the strategy without
   recovery).
2. **The live bot's +$173.89 on BTC is empirical proof the real
   mechanics work.** The audit is on a real account with real fills.
   The live bot can't be cap-hit-and-stuck during the audit window
   because the audit reports active P&L gain. So whatever mechanic
   the live bot uses *does* avoid (or break out of) the stuck state.
   The hypothesis space above is bounded — one of {per-layer TP,
   layer-spacing min, time-based exit, or some combination} explains
   the gap.

---

## Implications for ETH + SOL expansion recommendation

**Not yet evaluable.** ETH and SOL W4 backtests were not run per the
W4-gate-failure stop rule. Until V2.1 produces a Pine v6 that
*reproduces* the live bot's BTC W4 +$173.89, running ETH/SOL against
this version of Pine v6 would just propagate the same cap-hit-stuck
failure mode to new assets, giving zero useful information about
expansion thesis.

Expansion-thesis evaluation deferred to V2.1.

---

## Next steps

| Day | Owner | Action |
|---|---|---|
| Day 16 | VPS CC | Extract actual live bot mechanics from code/state/log. Verify which of Hypothesis 1-4 applies. |
| Day 17 | Mac CC (this MCP) | Receive V2.1 spec with corrected mechanics. Retry Pine v6 backtest on BTC W4. If W4 cross-validates audit, run the remaining 11 cells. |
| ≥ Day 17 | VPS CC | V3 Frame A simulator: simulates using the ACTUAL bot code, bypassing the spec-translation gap entirely. |
| ≥ Day 18 | Both | V5 cross-frame audit: Frame A (actual code) vs Frame B (Pine v6 corrected spec). If they converge, expansion thesis is well-validated. If they diverge, deeper investigation. |

---

## Reference artefacts

- **Pine v6 source**: `.ray_dca_v1.pine` (repo root, committed)
- **TV cloud script**: `USER;5b11fa77844a4787bfd6d638fc261f87` v1.0,
  title "Ray DCA v1"
- **Backtest chart**: BINANCE:BTCUSDT.P, 1h, 1224 bars loaded
  (2026-03-25 → 2026-05-15)
- **Branch**: `feat/ray-dca-pine-v6-backtest` on
  `taiwor88/tradingview-mcp` fork (NOT opened as PR; awaits V2.1)

## Operational notes for V2.1

- **DOM-attach hazard recurred** during Phase 2b: `pine_open + pine_compile`
  clobbered the Test1 RSI Sweep cloud script (third documented instance
  of this defect). Restored via REST `save/new` with
  `allow_overwrite=true`. Verified via GET. Logged as D9 candidate for a
  follow-up MCP patch (already noted in the D8 PR description).
- **TV chart bar-load** required a hard chart reload to extend 1h
  history past 15 days. All synthetic-event approaches failed
  (8 methods tried). The reload yielded 51 days, enough for W4 but
  not full historical sweep — V2.1 may need similar manual scroll-back
  for W1/W2/W3 once mechanics are corrected.
- **TV strategy `reportData.performance.all` reflects FULL backtest
  range**, not the chart's visible range. Per-window metrics require
  in-JS filtering of the raw `trades` array by `x.tm` (exit timestamp
  in ms), as done here. Documenting for V2.1 to avoid the same false
  start.
