# Cointegration & VECM Pair Trading Notebook

## What this notebook does

It checks whether a group of stocks move together in a stable, long-run way
(**cointegration**), and if they do, fits a **VECM** (Vector Error Correction
Model) to describe and forecast their prices.

In plain terms: assets can drift apart in the short term but keep snapping
back to a stable relationship in the long term. If that holds, you can trade
the "spread" between them, betting on the snap-back. This notebook is the
statistics step (test → fit → forecast) before building that kind of
strategy.

**This version** uses three Indian bank stocks — **HDFCBANK.NS, ICICIBANK.NS,
AXISBANK.NS** — at 1-minute resolution over the last 5 trading days (yfinance's
max lookback for 1-minute data).

---

## ⚠️ Fix before running: missing imports

The notebook as uploaded is missing some imports. The example outputs saved
inside the file only worked because they were run in a session where
`adfuller`, `VAR`, `coint_johansen`, `VECM`, and `plt` were already imported
from an earlier attempt — but those import lines are **not in the notebook
itself**. Running it fresh, top to bottom, will crash with a `NameError`.

Add this near the top (in the same cell as the `pandas`/`polars`/`yfinance`
imports, before the `TimeSeriesDataLoader` class):

```python
from statsmodels.tsa.stattools import adfuller
from statsmodels.tsa.vector_ar.var_model import VAR
from statsmodels.tsa.vector_ar.vecm import coint_johansen, VECM
import matplotlib.pyplot as plt
```

## Requirements

```bash
pip install pandas polars yfinance statsmodels matplotlib numpy
```

Also needs Jupyter (or VS Code / Colab) and an internet connection — the
data cell downloads live prices from Yahoo Finance.

## How to run

1. Add the missing imports above.
2. Run the cells **in order, top to bottom** — later cells reuse variables
   from earlier ones (`data`, `k_ar_diff`, `vecm_res`).
3. After the Johansen test cell, check its result before running the VECM
   cell — that cell hard-codes `coint_rank = 1`, which you may need to
   change (see Cell 5 below).
4. To use different stocks, edit the `add_from_yfinance(...)` lines with
   different tickers — note 1-minute data is capped at ~5–7 days of history
   by Yahoo Finance.
5. The last cell in the file is empty — nothing to run there.

---

## Cell-by-cell explanation

### Cell 1 — Notes on EC vs VEC vs VAR
Just a comment, no code runs. Explains the theory: an **EC (Error
Correction) model** is for 2 time series; a **VEC (Vector Error Correction)
model** is the same idea for 3+ series; VEC and **VAR** are close cousins —
VEC is a VAR model that also includes an "error correction term" pulling
series back toward their long-run relationship.
**Check:** nothing — no output.

### Cell 2 — Load and combine price data
Defines `TimeSeriesDataLoader`, which downloads a Close-price series per
ticker from Yahoo Finance (`add_from_yfinance`), warns if a ticker returns
nothing, strips timezone info so series line up cleanly, and combines them
all onto one time grid in `build_dataset()` (resample → forward/back-fill →
drop anything still missing). Then it loads the three bank stocks and builds
the combined dataset. (The leftover "invert USDCAD" block is dead code from
an earlier FX version — harmless here since there's no `USDCAD` column.)

**Check in the output:**
- `Loaded '<ticker>': N rows` for each stock — should be non-trivial numbers
  (in the saved run: ~1,800 rows each before alignment). If a ticker shows
  the ⚠️ warning, that download failed — check the ticker symbol.
- `Final cleaned dataset shape` — in the saved run this was **(9001, 3)**,
  i.e. 9,001 aligned 1-minute timestamps across 3 stocks. A shape with 0
  rows means the series didn't overlap at all after alignment.

### Cell 3 — Stationarity check (ADF test)
Runs the **Augmented Dickey-Fuller (ADF) test** on raw prices ("Levels") and
on day-to-day price changes ("First Differences"), skipping any column that
turns out constant (avoids a crash on zero-variance data).

**Check in the output:**
- **First Differences:** all three stocks came back `Stationary: True` in
  the saved run — expected, price changes should be stable.
- **Levels:** the "textbook" expectation is `Stationary: False` for raw
  prices. In the saved run, **AXISBANK.NS** showed that expected pattern
  (p = 0.99, not stationary), but **HDFCBANK.NS** and **ICICIBANK.NS**
  came back `Stationary: True` even in levels. With very large intraday
  samples (9,000+ points) the ADF test gains a lot of statistical power and
  can flag mild mean-reversion or noise as "stationary" even in a price
  series — worth treating that particular result with some skepticism
  rather than taking it at face value.

### Cell 4 — Choose the lag length (VAR lag selection)
First works out a safe maximum lag from the data size (rule of thumb:
`(rows − 1) / (variables + 1)`, capped at 15), then fits a VAR model and
picks the best lag count by AIC. That count becomes `k_ar_diff`, needed by
the VECM later.

**Check in the output:**
- In the saved run, AIC kept improving all the way to the cap
  (`Selected VAR Lag (p): 15`, the max allowed) instead of leveling off
  at some smaller number. That's a sign the true optimum might sit beyond
  15 — if you want a firmer answer, try raising the cap and re-running.
- `k_ar_diff` is just `p − 1` (here, 14) — this feeds directly into Cells 5
  and 6.

### Cell 5 — Test for cointegration (Johansen test)
Runs the **Johansen test**, the standard way to test cointegration with 3+
series at once — prints a trace test and a max-eigenvalue test, each
comparing a statistic to its 5% critical value.

**Check in the output (saved run):**
- `r <= 0`: Stat 68.03 vs 5% CV 29.80 → **Cointegrated: True**
- `r <= 1`: Stat 8.10 vs 5% CV 15.49 → **Cointegrated: False**
- Both tests agree: there's exactly **one** cointegrating relationship
  among the three stocks — this is what justifies using `coint_rank = 1`
  in the next cell. If your own run shows a different number of `True`
  rows, that count is your rank instead.

### Cell 6 — Fit the VECM model
Fits the VECM using `k_ar_diff` from Cell 4 and `coint_rank` (hard-coded to
`1` — **update this if Cell 5 pointed to a different rank**). Prints the
full model summary and the **beta** (cointegrating) vector.

**Check in the output (saved run):**
- **Beta = [1, −0.72, 0.19]** for HDFCBANK / ICICIBANK / AXISBANK — this
  defines the stable spread: roughly `HDFCBANK − 0.72·ICICIBANK + 0.19·AXISBANK`
  is the combination that tends to stay put. These are your hedge ratios.
- **Alpha (loading coefficients)** — how fast each stock corrects back
  toward that spread: HDFCBANK (p = 0.001) and ICICIBANK (p < 0.001) were
  both statistically significant correctors; **AXISBANK was not**
  (p = 0.337) — in this run, AXISBANK mostly sits in the relationship
  without actively adjusting back toward it.

### Cell 7 — Forecast and summarize
Forecasts the next 15 one-minute steps with the fitted VECM, then builds
three outputs: a step-by-step forecast table, a before/after summary table
(current price, forecast price, expected change, % change), and a chart of
the last 100 historical minutes plus the forecast (dashed).

**Check in the output:**
- The forecast table's dates should continue right after your data's last
  timestamp with no gaps.
- The summary table's `% Change` column gives a quick read on direction and
  size — in the saved run all three moves were small (well under 0.2% over
  15 minutes), which is normal for a short intraday horizon.
- On the chart, the dashed forecast lines should connect smoothly from
  where the solid historical lines stop, with no sudden jump.
- This is a visual/numeric sanity check, not a trading signal by itself.

---

## Things to keep in mind

- **Fix the missing imports first** (see the warning above) — the notebook
  won't run standalone otherwise.
- The "invert USDCAD" block in Cell 2 is leftover code from an earlier FX
  version of this notebook — it does nothing here and can be deleted.
- `coint_rank` in Cell 6 is **manually typed in**, not pulled automatically
  from Cell 5's result — always re-check it after changing tickers or the
  time window.
- 1-minute Yahoo Finance data only goes back a handful of days, so this
  notebook is naturally limited to short-horizon, intraday analysis — it
  isn't testing a long-run relationship the way daily or weekly data would.
- This notebook stops at fitting/forecasting the statistical relationship —
  turning the beta spread into actual entry/exit trading rules is a
  separate step not covered here.
