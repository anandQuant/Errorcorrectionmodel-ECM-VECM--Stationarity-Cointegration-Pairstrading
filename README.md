# Stationarity-Cointegration-PairsTrading

A simple Python tool that automatically checks time series data (like stock prices) and selects the right statistical model to analyze them.

---

## 💡 What Does This Project Do?

When working with financial data or stock prices, you can't just run a standard regression—doing so often leads to false signals (called *spurious correlation*). 

This project solves that by automatically running a **3-step decision tree**:

1. **Stationarity Check (ADF Test):** Checks if your two datasets bounce around a constant mean or if they trend over time.
2. **Cointegration Test:** If both datasets trend over time, it checks if they move together in a long-term relationship (useful for **Pairs Trading**).
3. **Model Selection:** Automatically fits the best model based on the results:
   - **Both stationary:** Runs OLS Regression or VAR model.
   - **Both non-stationary & cointegrated:** Runs an Error Correction Model (VECM) to track the mean-reverting spread.
   - **Not cointegrated / Mixed:** Takes the first difference (price changes) to make data stationary before running OLS.

---

## ✨ Features

- **Fetch Data Easily:** Download stock prices directly using `yfinance` or load your own local `CSV` file.
- **Automated Workflow:** No need to manually check ADF tables—the code routes everything for you.
- **Visual Plots:** Automatically plots raw price charts, differenced charts, and trading spreads at every step.
- **Jupyter Notebook Ready:** Formatted cleanly into step-by-step cells.

---

## ⚡ Quick Start

### 1. Installation
Install the required Python packages:

```bash
pip install numpy pandas matplotlib statsmodels yfinance

### Decission flow

[ Input 2 Time Series ]
                                |
                   Are both series stationary?
                     /                      \
               ( YES )                      ( NO )
                 /                            \
      [ Run OLS / VAR ]               Are they Cointegrated?
                                       /                  \
                                 ( YES )                  ( NO )
                                   /                        \
                           [ Run VECM Model ]       [ Take First Difference ]
                          (Pairs Trading Spread)                |
                                                        [ Run OLS Regression ]
