# File: `pyeventbt/data_provider/connectors/csv_data_connector.py`

## Module

`pyeventbt.data_provider.connectors.csv_data_connector`

## Purpose

Implements the CSV-based backtest data provider. Loads per-symbol CSV files, resamples raw data to M1, aligns all symbols on a common master datetime index, forward-fills gaps, resamples to each requested timeframe, and yields `BarEvent` objects through per-symbol generators. This is the primary data source for all backtesting operations.

## Tags

`connector`, `csv`, `backtest`, `resampling`, `gap-fill`, `generator`, `integer-prices`, `polars`, `lookahead-prevention`

## Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events.events` | `BarEvent`, `Bar` dataclass |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface` | `IDataProvider` base |
| `pyeventbt.data_provider.core.configurations.data_provider_configurations` | `CSVBacktestDataConfig` |
| `pyeventbt.broker.mt5_broker.mt5_simulator_wrapper` | `Mt5SimulatorWrapper as mt5` for `symbol_info()` (digits, currency info) |
| `polars` | CSV scanning, resampling (`group_by_dynamic`), filtering, DataFrame operations |
| `pandas` | `pd.Timestamp` for minute-index calculations; legacy `get_latest_bars_pandas` |
| `decimal` | `Decimal`, `ROUND_DOWN` for tick price reconstruction |
| `functools.lru_cache` | Caching timeframe parsing results |
| `logging` | `pyeventbt` and `backtest_info` loggers |

## Classes/Functions

### `_AGG_MAP` (module-level dict)

Polars aggregation mapping for OHLCV resampling.

```python
_AGG_MAP = {
    "open": pl.first, "high": pl.max, "low": pl.min,
    "close": pl.last, "tickvol": pl.sum, "volume": pl.sum, "spread": pl.first,
}
```

---

### `CSVDataProvider(IDataProvider)`

Main class. ~870 lines.

**Signature:**
```python
class CSVDataProvider(IDataProvider):
    def __init__(self, configs: CSVBacktestDataConfig) -> None
```

**Key Instance Attributes:**

| Attribute | Type | Description |
|---|---|---|
| `backtest_start_timestamp` | `datetime | None` | Start of backtest window |
| `backtest_end_timestamp` | `datetime | None` | End of backtest window |
| `csv_dir` | `str` | Directory containing CSV files |
| `base_timeframe` | `str` | Lowest-resolution TF (must be first in `timeframes_list`) |
| `timeframes_list` | `list` | All requested timeframes |
| `continue_backtest` | `bool` | Set to `True`; flipped when all generators exhaust |
| `close_positions_end_of_data` | `bool` | Signals end-of-data to portfolio handler |
| `symbol_data_generator` | `dict[str, generator]` | Per-symbol bar generators |
| `symbol_list` | `list[str]` | Tradeable + auxiliary symbols |
| `tradeable_symbols` | `list[str]` | Only user-specified tradeable symbols |
| `auxiliary_symbol_list` | `list[str]` | Auto-derived FX crosses for currency conversion |
| `complete_symbol_data_timeframes` | `dict[str, dict[str, pl.DataFrame]]` | `{symbol: {tf: DataFrame}}` |
| `latest_index_timeframes` | `dict[str, dict[str, datetime]]` | Current simulation time per symbol x TF |
| `symbol_digits` | `dict[str, int]` | Decimal precision per symbol |
| `_base_timestamps` | `dict[str, list[datetime]]` | Cached base-TF timestamps per symbol |
| `_base_idx_map` | `dict[str, dict[datetime, int]]` | Timestamp-to-index lookup |
| `_base_minutes_global` | `dict[str, list[int]]` | Global minute-since-epoch per bar |
| `_base_minutes_day` | `dict[str, list[int]]` | Minute-of-day (0-1439) per bar |

#### Methods

---

##### `__init__(self, configs: CSVBacktestDataConfig) -> None`

Validates configuration, creates auxiliary symbol list for FX cross pairs, loads and processes all CSV data, initializes per-symbol generators.

---

##### `_timeframe_to_duration(self, tf: str) -> str` (lru_cached)

Converts MT5-style timeframe string to Polars window duration string.

| Input | Output |
|---|---|
| `"5min"` | `"5m"` |
| `"1H"` | `"1h"` |
| `"1D"` | `"1d"` |
| `"1W"` | `"1w"` |
| `"1M"` | `"1mo"` |

---

##### `_parse_timeframe_to_minutes(self, timeframe: str) -> int` (lru_cached)

Converts timeframe string to total minutes.

| Input | Output |
|---|---|
| `"5min"` | `5` |
| `"1H"` | `60` |
| `"1D"` | `1440` |

---

##### `_merge_sorted_unique(self, list1, list2) -> list[datetime]`

O(n) merge of two sorted datetime lists into a sorted unique list. Used to build the master datetime index across all symbols.

---

##### `_check_first_element_in_tf_list_is_base_tf(self, base_timeframe, timeframes_list) -> None`

Validates that `timeframes_list[0] == base_timeframe`. Raises `Exception` on failure.

---

##### `_check_account_currency_is_supported(self, account_currency: str) -> None`

Validates account currency is one of `{"USD", "EUR", "GBP"}`. Raises `Exception` on failure.

---

##### `_create_auxiliary_symbol_list(self, tradeable_symbols, account_currency) -> list[str]`

Determines which FX cross pairs are needed to convert margin/profit currencies to the account currency. Uses `mt5.symbol_info()` to inspect `currency_margin` and `currency_profit` for each tradeable symbol.

---

##### `_open_convert_csv_files(self) -> None`

Core data loading pipeline. Three stages:

1. **Stage 1 -- Scan & Resample to M1:** For each symbol, lazy-scans CSV (headerless, 9 columns: date, time, OHLC, tickvol, volume, spread), parses datetime, filters to backtest window, collects eagerly, resamples to 1-minute bars via `group_by_dynamic`.
2. **Stage 2 -- Build master index:** Takes the latest start date across all symbols, creates a unified datetime index.
3. **Stage 3 -- Align, fill, resample:** Joins each symbol's M1 data onto the master index, forward-fills close, derives open/high/low from filled close for gap bars, marks gap bars with `tickvol=volume=spread=1`, filters to common start, then resamples to each requested timeframe via `group_by_dynamic`. Weekly bars get a 7-day datetime offset.

After stage 3, populates timestamp and minute-index caches for the base timeframe.

---

##### `_get_new_bar_generator(self, symbol: str) -> Generator[BarEvent]`

Creates a generator that yields `BarEvent` objects for the base timeframe. Scales float prices to integers (`price * 10^digits`), creates zero-copy `memoryview` objects over Polars->NumPy arrays for maximum iteration performance. Validates no nulls exist in price columns before iteration.

---

##### `_populate_symbol_data_with_generators(self) -> None`

Calls `_get_new_bar_generator` for each symbol and stores in `self.symbol_data_generator`.

---

##### `get_latest_tick(self, symbol: str) -> dict`

Returns a tick dictionary. Looks up the *next unfinished bar* (first bar after current simulation time) and uses its **opening price** as bid to avoid lookahead bias. Reconstructs bid/ask using `Decimal` arithmetic with the bar's spread. At end of backtest, falls back to last bar's close.

**Returns:** `dict` with keys: `time`, `bid`, `ask`, `last`, `volume`, `time_msc`, `flags`, `volume_real`.

---

##### `get_latest_bar(self, symbol: str, timeframe: str = None) -> BarEvent`

Returns the most recent closed bar. For the base timeframe, returns the last bar at or before the current simulation time. For higher timeframes, returns the **second-to-last** bar to avoid returning the currently-forming (incomplete) bar. Prices are integer-scaled.

---

##### `get_latest_bars(self, symbol: str, timeframe: str = None, N: int = 2) -> pl.DataFrame`

Returns the N most recent bars as a Polars DataFrame with **float** prices (not integer-scaled). For higher timeframes, excludes the most recent forming bar.

---

##### `get_latest_bid(self, symbol: str) -> Decimal`

Delegates to `get_latest_tick(symbol)["bid"]`.

---

##### `get_latest_ask(self, symbol: str) -> Decimal`

Delegates to `get_latest_tick(symbol)["ask"]`.

---

##### `get_latest_datetime(self, symbol: str, timeframe: str = None) -> datetime`

Returns `get_latest_bar(symbol, timeframe).datetime`.

---

##### `_base_tf_bar_creates_new_tf_bar(self, latest_base_tf_time, timeframe, symbol) -> bool`

Determines if the current base-TF bar crosses into a new higher-TF bucket. Uses direct datetime field comparisons (year, month, day, hour, minute) against the last recorded timeframe datetime. Handles intraday (sub-hourly, hourly), daily, weekly (ISO week number), and monthly boundaries.

---

##### `update_bars(self) -> list[BarEvent]`

Advances each symbol's generator by one bar. Skips filled/synthetic bars (where `tickvol == volume == spread == 1`). For each real bar:
1. Updates `latest_index_timeframes` for base TF.
2. Appends base-TF `BarEvent` (only for tradeable symbols, not auxiliary).
3. Checks each higher TF for boundary crossing; if crossed, updates tracking and appends higher-TF `BarEvent`.
4. Sets `close_positions_end_of_data = True` when any generator is exhausted.

**Returns:** `list[BarEvent]`

---

##### Deprecated Methods (still present)

| Method | Notes |
|---|---|
| `get_latest_tick_old(symbol)` | Uses `BarEvent.data` integer fields directly instead of DataFrame lookup |
| `get_latest_bar_old_lookahead_bias(symbol, timeframe)` | Returns last bar without second-to-last protection for higher TFs |
| `get_latest_bars_pandas(symbol, timeframe, N)` | Returns `pd.DataFrame` instead of `pl.DataFrame` |
| `_base_tf_bar_creates_new_tf_bar_old(...)` | Uses integer minute-index caches for bucket comparison |
| `_base_tf_bar_creates_new_tf_bar_f(...)` | Optimized variant with cached TF categorization |

## Data Flow

```
__init__
  --> _check_first_element_in_tf_list_is_base_tf()
  --> _check_account_currency_is_supported()
  --> _create_auxiliary_symbol_list()
  --> _open_convert_csv_files()
      --> Stage 1: pl.scan_csv -> parse -> filter -> collect -> resample M1
      --> Stage 2: build master index from max(first_dates)
      --> Stage 3: join/align -> forward_fill -> resample per TF
      --> populate caches (_base_timestamps, _base_minutes_global, etc.)
  --> _populate_symbol_data_with_generators()
      --> _get_new_bar_generator(symbol) for each symbol

update_bars() [called by event loop]
  --> next(generator) for each symbol
  --> skip filled bars (tickvol==volume==spread==1)
  --> emit base-TF BarEvent
  --> _base_tf_bar_creates_new_tf_bar() for each higher TF
      --> if boundary crossed: emit higher-TF BarEvent via get_latest_bar()
  --> return list[BarEvent]
```

## Gaps & Issues

1. **Five deprecated methods remain in the file** without `@deprecated` decorators or `DeprecationWarning`. They add ~200 lines of dead code.
2. **Unused cache fields:** `_base_minutes` and `_base_idx_map_int` are initialized in `__init__` but never populated or read.
3. **`_create_auxiliary_symbol_list` hardcodes a static FX pair list** (33 pairs). New instruments require code changes.
4. **`_check_account_currency_is_supported` only allows USD/EUR/GBP.** Other major currencies (JPY, CHF, AUD, CAD) are unsupported.
5. **`update_bars` does not set `continue_backtest = False`** when all generators are exhausted. It only sets `close_positions_end_of_data = True` per-symbol. The service layer reads `continue_backtest` but the CSV connector never flips it to `False`; the TradingDirector likely relies on an empty event list to exit.
6. **`get_latest_tick` accesses the next bar's opening price** by filtering `datetime > current_time`. At end-of-data, it falls back to `df.tail(1)["close"]`, which may return stale data.
7. **Weekly bar datetime adjustment** (`- pl.duration(days=7)`) in `_open_convert_csv_files` shifts the bar timestamp backward by exactly 7 days, which may not align correctly with all trading calendars.
8. **`get_latest_bars` for higher TFs** uses `filtered_df.slice(df_len - N - 1, N)` which may return unexpected results if `df_len` is exactly `N + 1`.
9. **Logging uses ANSI color codes directly** (green `\x1b[92;20m`, yellow `\x1b[93;20m`) which may render as garbage in non-terminal outputs (files, CI logs).

## Requirements Derived

- **REQ-DP-CSV-001:** Remove or explicitly deprecate the five legacy methods.
- **REQ-DP-CSV-002:** Remove unused `_base_minutes` and `_base_idx_map_int` cache fields.
- **REQ-DP-CSV-003:** Consider making the auxiliary FX pair list configurable or dynamic.
- **REQ-DP-CSV-004:** Add a mechanism to set `continue_backtest = False` when all symbol generators are exhausted.
- **REQ-DP-CSV-005:** Support additional account currencies beyond USD/EUR/GBP.
- **REQ-DP-CSV-006:** Validate that higher-TF `get_latest_bars` slice logic is correct for edge cases (N=1, df_len=2).
- **REQ-DP-CSV-007:** Use a structured logging formatter rather than inline ANSI escape codes.
