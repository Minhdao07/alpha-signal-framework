# Alpha Signal Research Framework

A cross-sectional equity signal-research pipeline that extracts **idiosyncratic** alpha
and evaluates it the way a hedge-fund quant would. The goal is **rigorous, honest
evaluation — not return maximization.** A raw backtest can make almost any signal look
attractive; this framework is built to tell you which signals have a *real, tradeable*
edge once you strip out risk exposures and pay realistic transaction costs.

The project is delivered as a set of **Jupyter notebooks** (one per pipeline stage), each
carrying its own methodology write-up, plus an orchestrator notebook that runs everything
end-to-end and interprets the output.

---

## What it does

1. **Universe** — the **~500 S&P 500 constituents** (ticker + GICS sector pinned in
   `sp500_universe.csv` so re-runs are reproducible). Daily data via `yfinance`, 2018–2026.
   With ~100 names per quintile the long-short book is a portfolio rather than a handful of
   bets. A loud **survivorship-bias warning** prints on every data load: this is the index's
   *current* membership held fixed back to 2018, so breadth improved but the bias did **not**
   go away.
2. **Two raw signals** — Momentum (12–1) and short-term mean-reversion (negative 5-day
   return): a *slow* signal and a *fast* signal, chosen to make the cost story vivid.
3. **Cross-sectional neutralization** — **one OLS regression per day** across the stocks
   (not a time-series regression) of the day's signal on each stock's **market beta**
   (rolling vs SPY), a **log dollar-volume size proxy**, and **sector dummies**. The
   **residual** is the neutralized (idiosyncratic) signal. Beta is used instead of
   book-to-market to avoid look-ahead bias from lagged/restated fundamentals.
4. **Per-day preprocessing** — winsorize (±3σ) then z-score, cross-sectionally, before each
   regression.
5. **Leakage-safe evaluation** — the signal at day *t* is paired with the *t+1* forward
   return (`.shift(-1)`), never the same day.

### Metrics (computed for both raw and neutralized signals)

- **Rank IC (Spearman)** as a daily time series → mean IC, **IC t-stat** `= mean/std × √N`,
  and IC hit-rate.
- **IC decay** across horizons t+1, t+2, t+3, t+5, t+10.
- **Turnover** of the long-short book.
- **Quintile long-short spread**, **gross vs net-of-cost** (cost in bps, config-driven,
  charged via the daily turnover series).
- **Transaction-cost sensitivity** — the same book re-priced at 1/5/10/20 bps, plus each
  signal's **breakeven cost** (the level at which its gross edge is exactly consumed), so the
  verdict's dependence on the cost assumption is explicit rather than hidden.
- **Fundamental Law of Active Management** link: `IR ≈ IC × √BR` (note the square root).

---

## Repository layout

```
alpha_framework/
  config.ipynb      # all parameters (universe, dates, lookbacks, costs) -> a `config` namespace
  data.ipynb        # yfinance ingestion + risk-exposure engineering (beta, size, sectors)
  signals.ipynb     # raw factors: momentum 12-1 and 5-day reversion
  neutralize.ipynb  # per-day winsorize/z-score + one cross-sectional OLS per day (residualization)
  evaluate.ipynb    # IC stats, IC decay, turnover, gross/net quintile P&L, Fundamental-Law IR
  plotting.ipynb    # 2x2 diagnostic report per factor
  main.ipynb        # orchestrator: runs the pipeline end-to-end + methodology & interpretation
  sp500_universe.csv # the ~500 tickers + GICS sectors, pinned for reproducibility
  requirements.txt
output/
  alpha_diagnostic_momentum_12_1.png
  alpha_diagnostic_short_term_reversion.png
README.md
```

`sp500_universe.csv` was generated once from Wikipedia's *List of S&P 500 companies* table
(snapshot taken 2026-08-05) and is committed rather than fetched at run time — index membership
drifts, and a backtest whose universe silently changes between runs is not a repeatable
experiment. It is, to be clear, a **present-day** membership list, not point-in-time history.

The module notebooks contain **only** methodology markdown + function/constant definitions
(no heavy side effects), so `main.ipynb` composes them with IPython's `%run` — e.g.
`%run data.ipynb` executes that notebook's code cells in the shared kernel, exactly like
importing the old package modules.

---

## How to run

Install the dependencies and open the notebooks from **inside `alpha_framework/`** (so the
relative `%run <name>.ipynb` paths and the `../output` save path resolve):

```bash
pip install -r alpha_framework/requirements.txt
cd alpha_framework
jupyter lab            # or: jupyter notebook
# open main.ipynb and Run All
```

Or execute headless without opening Jupyter:

```bash
cd alpha_framework
jupyter nbconvert --to notebook --execute --inplace main.ipynb
```

`main.ipynb` prints the raw-vs-neutralized scorecard for each factor, an IC-decay table, a
data-driven cost verdict, and saves the two diagnostic figures to `../output/`. Each figure
has four panels: cumulative IC, IC decay, gross-vs-net long-short equity, and per-quintile
returns.

---

## What the run shows (2018-01 → 2026-07, 494 names)

Both signals are neutralized; costs are 5 bp per unit of one-way turnover.

| Factor | Neutralized IC (t-stat) | Turnover/day | Gross ann. LS | **Net ann. LS** | Breakeven | Verdict |
|---|---|---|---|---|---|---|
| **Momentum 12–1** | 0.0143 (**3.88**) | 10.0% | +4.51% | **+3.24%** | **17.8 bp** | Survives costs |
| **Reversion 5-day** | 0.0062 (**2.33**) | 67.3% | **+6.29%** | **−2.19%** | **3.7 bp** | Destroyed by costs |

**Costs invert the ranking.** On gross P&L reversion wins (+6.29% vs +4.51%; on the *raw*
signals it is +12.20% vs +5.15%). On net, the ordering flips — momentum earns +3.24% while
reversion loses 2.19%. Ranking these two signals on gross numbers picks the wrong one.

The breakeven column makes the same point sharply: momentum's edge is only exhausted at
**17.8 bp** of one-way cost, reversion's at **3.7 bp**. Since 5 bp is reasonable-to-optimistic
for US large caps, momentum has a real margin of safety and reversion needs frictions below
what any desk actually pays.

Two further findings worth noting:

- **Neutralization *improved* momentum's t-stat** (3.21 → 3.88) even as its IC fell
  (0.0186 → 0.0143): removing beta/size/sector stripped out more variance than signal, leaving
  a smaller but far steadier edge.
- **Reversion's IC peaks at t+5, not t+1** (0.0062 → 0.0077 neutralized). The reversal needs
  several days to play out, so a daily re-sort pays full transaction costs for a signal that
  has not finished working — a 3–5 day holding period is the obvious way to attack the
  8.5%/yr drag.

See §7 of `main.ipynb` for the full interpretation.

> **Reproducibility.** Figures are computed live from `yfinance`, whose adjusted history is
> revised over time (dividends, splits, vendor corrections), and 9 of the 503 tickers failed to
> download on this run. Re-running weeks later moves the last decimal. The qualitative verdict
> is stable; the exact decimals are not.

---

## Known Limitations & Biases

This is a research **skeleton**; every simplification below pushes reported performance in
the *optimistic* direction, which is why the framework leans on the most conservative number
(net-of-cost long-short) for its verdict.

- **Survivorship bias (largest).** The universe is the S&P 500's *current* membership held
  fixed back to 2018. Companies dropped from the index over that period — the weak ones — never
  appear, so historical IC and P&L are structurally inflated. Widening from 30 to ~500 names
  improved *breadth*; it did **not** fix this. A production system needs a **point-in-time**
  universe with proper index add/drop history.
- **Look-ahead in the membership list itself.** Using today's constituents over a 2018–2026
  sample means the universe encodes information (who would still be in the index in 2026) that
  was not available in 2018 — a subtler cousin of the point above.
- **Survivorship cuts against the reported result, not for it.** Worth stating plainly: a
  point-in-time universe would contain more distressed and delisted names, which tend to
  exhibit *stronger* short-term reversal. Reversion's gross edge here may therefore be
  understated, while its cost problem would be worse (those names are the expensive ones to
  trade). The turnover→cost conclusion is robust to this; the exact gross figure is not.
- **Size proxy, not true market cap.** We use `log(price × avg volume)` because we lack
  point-in-time shares outstanding. It correlates with size but is not size.
- **Simplified costs.** A flat 5 bps × turnover ignores bid/ask spread and market impact that
  grow with trade size — and impact is *worse* for the fast (reversion) signal, so its true
  net P&L is likely even worse than shown. No slippage or impact curve.
- **No borrow / short constraints.** The long-short book assumes every short is freely
  available and free to hold. Hard-to-borrow names (often exactly what a reversion signal
  wants to short) carry borrow fees or are un-shortable.
- **Warm-up backfill & static sectors.** Rolling beta is back-filled over its initial 60-day
  window (mild look-ahead), and sector labels are fixed as of today rather than point-in-time.
- **Single assumed breadth.** `BR = 120` independent bets/year is a hand-set constant, not an
  estimate of the signals' true cross-sectional independence, so the Fundamental-Law IR is a
  scale sanity-check, not a tradeable Sharpe.
- **Not bit-reproducible.** The pipeline is deterministic given its inputs, but the inputs are
  not: `yfinance` restates adjusted prices as corporate actions and vendor corrections land, so
  identical code run weeks apart yields slightly different decimals. Production research pins a
  point-in-time data snapshot (and version-controls it) so a backtest can be reproduced exactly.
- **Overlapping windows in the decay table.** The multi-day IC horizons use overlapping forward
  windows, which induces autocorrelation across observations. That is why only the *mean* IC is
  reported there and no t-stat — a naive t-stat would be overstated, and a Newey-West/HAC
  standard error would be the correct fix.
