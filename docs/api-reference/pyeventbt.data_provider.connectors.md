# Package: `pyeventbt.data_provider.connectors`

## Purpose

Contains the concrete data provider implementations that connect to external data sources. Each connector implements the `IDataProvider` interface and handles the mechanics of data ingestion, transformation, and bar event emission for its specific data source.

## Tags

`connectors`, `csv`, `mt5`, `backtest`, `live`, `data-ingestion`, `bar-generation`

## Modules

| Module | Description |
|---|---|
| `csv_data_connector.py` | `CSVDataProvider` -- reads CSV files, resamples, aligns, gap-fills, and yields `BarEvent`s via generators for backtesting (~910 lines) |
| `mt5_live_data_connector.py` | `Mt5LiveDataProvider` -- polls the MT5 terminal for real-time bar and tick data (~423 lines) |

## Internal Architecture

```
IDataProvider (interface)
    |
    +-- CSVDataProvider
    |     - Loads CSVs at init time (lazy scan -> collect -> resample -> align -> gap-fill)
    |     - Per-symbol Python generators yield BarEvent with integer-scaled prices
    |     - update_bars() advances generators, detects higher-TF boundary crossings
    |     - Lookahead protection: higher-TF bars return second-to-last (fully formed) bar
    |
    +-- Mt5LiveDataProvider
          - Polls mt5.copy_rates_from_pos() on each update_bars() call
          - Tracks last-seen datetime per symbol x timeframe
          - Emits BarEvent when a newer bar is detected
```

Both connectors produce `BarEvent` objects with `Bar` dataclass payloads containing integer-scaled prices (price * 10^digits) for performance and precision.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events.events` | `BarEvent`, `Bar` dataclass |
| `pyeventbt.broker.mt5_broker.mt5_simulator_wrapper` | CSV connector uses `Mt5SimulatorWrapper as mt5` for `symbol_info()` |
| `pyeventbt.utils.utils` | Live connector uses `check_platform_compatibility()` |
| `polars` | DataFrame operations, resampling, filtering |
| `pandas` | Legacy return types, `pd.Timestamp` for minute-index calculations |
| `MetaTrader5` | Live connector only; conditionally imported |

## Gaps & Issues

1. **CSV connector contains multiple deprecated methods** that are still present: `get_latest_bar_old_lookahead_bias`, `get_latest_tick_old`, `_base_tf_bar_creates_new_tf_bar_old`, `_base_tf_bar_creates_new_tf_bar_f`, `get_latest_bars_pandas`. These should be removed or explicitly marked as deprecated.
2. **Live connector's `_map_timeframe` has a dead `except KeyError`** block -- the `KeyError` would occur outside the `try` scope (on the `return` statement).
3. **Live connector's `update_bars()` calls `get_latest_bar()` twice** per new bar detection (once to check datetime, once to append), doubling the MT5 API calls unnecessarily.
4. **CSV connector has unused cache fields** (`_base_minutes`, `_base_idx_map_int`) that are initialized but never populated or read.
5. **No `__init__.py`** visible in the connectors directory listing (may be implicit).
