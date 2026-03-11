# pyeventbt.backtest.core.backtest_results

- **File**: `pyeventbt/backtest/core/backtest_results.py`
- **Module**: `pyeventbt.backtest.core.backtest_results`
- **Purpose**: Encapsulates backtest output data (PnL, returns, trades) and provides visualization via matplotlib.
- **Tags**: `backtest`, `results`, `plotting`, `pandas`, `matplotlib`

## Dependencies

| Dependency | Usage |
|---|---|
| `pandas` | `pd.DataFrame` for PnL and trades storage; `.pct_change()` for returns; `.astype(float)` for type conversion; `.plot()` for charting |
| `matplotlib.pyplot` | `plt.show()` to render plots |
| `warnings` | Silences `FutureWarning` from matplotlib |

## Classes/Functions

### BacktestResults

- **Signature**: `BacktestResults(backtest_pnl: pd.DataFrame, trades: pd.DataFrame)`
- **Description**: Data container for backtest results. Constructed by the `Strategy` class after a backtest completes. Stores raw PnL DataFrame, a float-cast version, percentage returns, and the trades DataFrame.

#### Constructor

| Parameter | Type | Description |
|---|---|---|
| `backtest_pnl` | `pd.DataFrame` | DataFrame with at least `EQUITY` and `BALANCE` columns (one row per bar) |
| `trades` | `pd.DataFrame` | DataFrame of executed trades |

#### Properties

| Property | Type | Description |
|---|---|---|
| `pnl` | `pd.DataFrame` | The PnL DataFrame cast to float |
| `returns` | `pd.Series` | Percentage change of the `EQUITY` column (`pct_change()`) |
| `trades` | `pd.DataFrame` | The trades DataFrame as provided |
| `backtest_pnl` | `pd.DataFrame` | The raw PnL DataFrame (uncast) |

#### Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `plot` | `plot() -> None` | Renders a matplotlib line chart of EQUITY and BALANCE columns with title "Backtest", tight margins, and a legend. Calls `plt.show()`. | `None` |
| `plot_old` | `plot_old() -> None` | Legacy plotting method; renders EQUITY and BALANCE without title or legend customization. Calls `plt.show()`. | `None` |

## Data Flow

1. `Strategy.backtest()` runs the event loop to completion.
2. The portfolio accumulates PnL data (EQUITY, BALANCE) per bar into a DataFrame.
3. The trade archiver collects all `FillEvent`s into a trades DataFrame.
4. `BacktestResults` is instantiated with both DataFrames and returned to the user.
5. The user can access `.pnl`, `.returns`, `.trades` for analysis or call `.plot()` for visualization.

## Gaps & Issues

1. **No computed statistics**: No Sharpe ratio, max drawdown, CAGR, Sortino ratio, Calmar ratio, win rate, average trade, profit factor, or other standard metrics. Users must compute these manually.
2. **`plot_old()` not deprecated**: The legacy method remains without any `@deprecated` decorator or deprecation warning.
3. **Commented-out imports suggest planned features**: `numpy`, `os`, `sklearn.linear_model.LinearRegression`, `scipy.stats.norm`, `pydantic.BaseModel`, `enum.Enum`, and a `utils` module are all commented out, indicating features that were planned but not implemented (e.g., regression analysis, statistical tests).
4. **`returns` computed eagerly**: The returns series is computed in `__init__` regardless of whether the user needs it.
5. **No serialization**: No built-in method to export results to CSV, JSON, or Parquet (unlike `TradeArchiver` which has multiple export methods).

## Requirements Derived

- R-BT-01: `BacktestResults` must store PnL and trades DataFrames from a completed backtest.
- R-BT-02: Equity returns must be computed as percentage change of the EQUITY column.
- R-BT-03: A `plot()` method must render EQUITY and BALANCE curves via matplotlib.
