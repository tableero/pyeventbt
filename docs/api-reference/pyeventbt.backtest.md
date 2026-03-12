# pyeventbt.backtest

- **Package**: `pyeventbt.backtest`
- **Purpose**: Contains backtesting result handling and visualization. Provides the `BacktestResults` class returned by `Strategy.backtest()`.
- **Tags**: `backtest`, `results`, `visualization`, `package-init`

## Modules

| Module | Description |
|---|---|
| `backtest.core.backtest_results` | `BacktestResults` class -- holds PnL data, trades, and provides plotting |

## Internal Architecture

The package has a minimal structure:

```
backtest/
  __init__.py              # Re-exports BacktestResults
  core/
    backtest_results.py    # BacktestResults implementation
```

`__init__.py` re-exports `BacktestResults` from `core.backtest_results`, making it available as `pyeventbt.backtest.BacktestResults`. It is also re-exported at the top-level `pyeventbt` package.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pandas` | `pd.DataFrame` for PnL and trades data; `pct_change()` for returns |
| `matplotlib.pyplot` | Plotting equity/balance curves |

The `BacktestResults` object is constructed by the `Strategy` class after a backtest run completes, receiving the PnL DataFrame and trades DataFrame from the portfolio and trade archiver components.

## Gaps & Issues

1. **No performance metrics**: No Sharpe ratio, max drawdown, CAGR, win rate, profit factor, or any other standard backtest statistics are computed. Users must calculate these themselves from the raw DataFrames.
2. **No benchmark comparison**: No ability to compare strategy returns against a benchmark.
3. **Deprecated `plot_old()` method**: A legacy plotting method is kept in the codebase with no deprecation warning.
4. **Commented-out imports**: Several imports (numpy, os, sklearn, scipy, pydantic, enum, utils) are commented out, suggesting planned features that were never implemented.
