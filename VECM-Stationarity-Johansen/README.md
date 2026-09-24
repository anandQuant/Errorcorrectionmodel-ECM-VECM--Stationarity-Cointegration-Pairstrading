# Cointegration & VECM Pair Trading Notebook

## What this notebook does

It checks whether a group of currency pairs move together in a stable, long-run
way (this is called **cointegration**), and if they do, it fits a **VECM**
(Vector Error Correction Model) to describe and forecast how they move.

In plain terms: some assets drift apart in the short term but keep coming back
to a stable relationship in the long term. If that's true, you can trade the
"spread" between them, betting it will snap back to normal. This notebook is
the statistical test-and-forecast step before building that kind of strategy.

The example in the notebook uses three FX pairs: **AUDUSD, NZDUSD, CADUSD**
(daily data, last 5 years).

---

## Requirements

```bash
pip install pandas polars yfinance statsmodels matplotlib
```

You also need Jupyter (or VS Code / Colab) to run the `.ipynb` file, and an
internet connection (Cell 3 downloads live price data from Yahoo Finance).

## How to run

1. Open the notebook and run the cells **in order, top to bottom**. Later
   cells reuse variables created earlier (`data`, `k_ar_diff`, `vecm_res`), so
   skipping around will break things.
2. After Cell 6 (the Johansen test), **look at the result before running
   Cell 7** — Cell 7 has a hard-coded line `coint_rank = 1` that you may need
   to change based on what Cell 6 tells you (see Cell 6 notes below).
3. To use your own assets, edit the `add_from_yfinance(...)` lines in Cell 3
   (ticker, name, period, interval), or use `add_from_csv(...)` if you have
   your own price file instead of pulling from Yahoo Finance.

---

## Cell-by-cell explanation

### Cell 1 — `pip install polars`
Installs the `polars` library (a fast alternative to pandas), used later for
quickly reading CSV files.
**Check:** the output should end with "Successfully installed..." and show no
red error text.

### Cell 2 — Notes on EC vs VEC vs VAR
Just a comment, no code runs. It explains the theory:
- An **EC (Error Correction) model** is for 2 time series.
- A **VEC (Vector Error Correction) model** is the same idea but for 3+ time
  series.
- VEC and **VAR** models are close cousins — VEC is a VAR model that also
  includes an "error correction term" (a term that pulls the series back
  toward their long-run relationship).
**Check:** nothing — no output.

### Cell 3 — Load and combine price data
Defines a `TimeSeriesDataLoader` class with three jobs:
- `add_from_yfinance()` — downloads a price series from Yahoo Finance.
- `add_from_csv()` — loads a price series from your own CSV file instead.
- `build_dataset()` — lines up all the series on the same dates, fills any
  gaps, and drops rows that still have missing data.

Then it actually loads AUDUSD, NZDUSD, and USDCAD (5 years, daily), flips
USDCAD into CADUSD so all three pairs are quoted the same way (against USD),
and prints the shape and first few rows.

**Check in the output:**
- `data.shape` should show 3 columns and roughly 1,200–1,300 rows (5 years of
  trading days).
- `data.head()` should show 3 sensible-looking price columns with no `NaN`
  values, dates in order.
- If the shape looks tiny or empty, the Yahoo Finance download likely failed
  (check your internet connection or the ticker symbols).

### Cell 4 — Stationarity check (ADF test)
Runs the **Augmented Dickey-Fuller (ADF) test**, which checks whether a
series is "stationary" (roughly: does it wander off with a trend, or does it
hover around a stable average?). It runs the test twice: once on the raw
prices ("Levels"), and once on the day-to-day price changes ("First
Differences").

**Check in the output:**
- **Levels section:** you generally *want* to see `Stationary: False`
  (p-value > 0.05) here — raw FX prices normally wander, which is expected.
- **First Differences section:** you generally *want* to see
  `Stationary: True` (p-value < 0.05) here — daily price changes should be
  stable.
- Together, this pattern (non-stationary in levels, stationary in
  differences) means each series is "integrated of order 1," which is the
  required condition before testing for cointegration.

### Cell 5 — Choose the lag length (VAR lag selection)
Fits a plain VAR model just to find the best number of lags (how many past
days to look back) using the AIC criterion, testing up to 15 lags. That lag
count is then converted into `k_ar_diff`, a setting the VECM model needs
later.

**Check in the output:**
- The summary table compares different lag lengths — the row picked by AIC
  is what gets used.
- `Selected VAR Lag (p)` — should be a small, reasonable number (not stuck at
  the max of 15, which would suggest the model is struggling to find a clean
  lag length).

### Cell 6 — Test for cointegration (Johansen test)
Runs the **Johansen test**, the standard way to check cointegration when you
have 3+ series at once. It prints two versions of the test (trace test and
max-eigenvalue test), each comparing a test statistic to a 5% critical value.

**Check in the output:**
- Look at the `r <= 0` row first: if `Stat > 5% CV` (so `Cointegrated: True`),
  that means at least one stable long-run relationship exists among the three
  currencies — the core result you need for this whole approach to work.
- Count how many rows say `Cointegrated: True` — that count is your suggested
  **cointegration rank** (how many independent stable relationships exist).
  This number is what you should plug into Cell 7's `coint_rank`.
- If every row says `Cointegrated: False`, the series aren't cointegrated,
  and a VECM/pair-trading approach isn't statistically justified for this
  particular basket.

### Cell 7 — Fit the VECM model
Fits the actual VECM using the lag setting from Cell 5 and the rank you
decide from Cell 6 (currently hard-coded to `coint_rank = 1` — **change this
number if Cell 6 suggested a different rank**). Prints the full model summary
and the **beta** vector.

**Check in the output:**
- **Beta (cointegrating vector):** these are the weights that define the
  stable "spread" between the assets — effectively your hedge ratios if you
  were to trade this spread.
- **Alpha (in the summary, "loading coefficients"):** shows how fast each
  asset corrects back toward the long-run equilibrium after a shock. Look at
  which ones are statistically significant (small p-value) — those are the
  assets that actively "do the correcting."

### Cell 8 — Forecast and plot
Uses the fitted VECM to forecast the next 15 steps ahead for all three
series, then plots the last 100 historical points (solid lines) together
with the forecast (dashed lines).

**Check in the output:**
- The dashed forecast lines should connect smoothly from where the solid
  historical lines end — no sudden jump.
- Watch for forecasts that trend off in a straight, aggressive line — VECM
  forecasts can extrapolate a trend fairly strongly over longer horizons, so
  treat far-out forecasts with caution.
- This is a visual sanity check, not a trading signal by itself.

---

## Things to keep in mind

- `coint_rank` in Cell 7 is **manually set**, not automatically pulled from
  the Cell 6 result — update it yourself based on what you see.
- The `add_from_csv()` loader method is written but not used in the example
  — it's there if you want to swap Yahoo Finance data for your own tick or
  intraday CSV file.
- Everything here is a statistical/forecasting workflow, not a ready-made
  trading strategy — turning the beta spread into actual trade signals
  (entry/exit thresholds, position sizing, etc.) is a separate step not
  covered in this notebook.
