# Package: `pyeventbt.data_provider`

## Purpose

Top-level data provider package responsible for feeding bar (OHLCV) data into the PyEventBT event loop. Abstracts the data source so that backtest (CSV) and live (MT5 terminal) paths share an identical interface. The package owns the full pipeline from raw data ingestion through resampling, alignment, gap-filling, and event emission.

## Tags

`data-provider`, `bar-data`, `csv`, `mt5`, `backtest`, `live-trading`, `event-source`, `quantdle`

## Modules

| Sub-package / Module | Description |
|---|---|
| `core/` | Domain entities, abstract interface, and configuration models |
| `connectors/` | Concrete data provider implementations (CSV backtest, MT5 live) |
| `services/` | High-level facade (`DataProvider`) and external data utilities (`QuantdleDataUpdater`) |
| `__init__.py` | Package initializer (empty or re-exports) |

## Internal Architecture

```
services/DataProvider (facade)
    |
    +-- delegates to --> connectors/CSVDataProvider   (backtest mode)
    |                    connectors/Mt5LiveDataProvider (live mode)
    |
    +-- selected by --> core/configurations/*DataConfig
    |
    +-- all implement --> core/interfaces/IDataProvider
    |
    +-- emits --> BarEvent --> Queue (shared event bus)

services/QuantdleDataUpdater (standalone utility)
    +-- downloads historical data --> local CSV files
```

**Lifecycle:**
1. `Strategy.backtest()` or `Strategy.run_live()` instantiates `DataProvider` (service layer) with a config object.
2. `DataProvider.__init__` inspects the config type and creates either `CSVDataProvider` or `Mt5LiveDataProvider`.
3. The `TradingDirector` event loop calls `DataProvider.update_bars()` when the queue is empty.
4. The underlying connector returns a `list[BarEvent]`; the service layer puts each event onto the shared `Queue`.

**Data flow for CSV backtest:**
- CSVs are lazy-scanned, parsed, filtered to the backtest window, resampled to M1, aligned across symbols on a common master index, gap-filled (forward-fill close, then derive OHLC from filled close), then resampled to each requested timeframe.
- A per-symbol Python generator yields `BarEvent` objects with integer-scaled prices for zero-allocation iteration.

**Data flow for MT5 live:**
- Each call to `update_bars()` polls `mt5.copy_rates_from_pos()` for every symbol x timeframe combination.
- If a bar datetime is newer than the last seen datetime, a `BarEvent` is emitted.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events.events` | `BarEvent`, `Bar` dataclass |
| `pyeventbt.broker.mt5_broker.mt5_simulator_wrapper` | `Mt5SimulatorWrapper` used as `mt5` in CSV connector for `symbol_info()` |
| `pyeventbt.trading_context.trading_context` | `TypeContext` enum for backtest vs live branching |
| `pyeventbt.utils.utils` | `check_platform_compatibility` for conditional MT5 import |
| `polars` | Primary DataFrame library for CSV loading, resampling, and bar retrieval |
| `pandas` | Legacy/secondary DataFrame usage (type hints, some return types) |
| `pydantic` | `BaseModel` for configuration and entity classes |
| `MetaTrader5` | Conditionally imported for live data feed (Windows only) |
| `quantdle` | Conditionally imported for historical data downloads |

## Gaps & Issues

1. **`core/entities/bar.py` is unused.** The `Bar` Pydantic model defined there is never imported by any connector or service. The actual `Bar` used throughout is the dataclass from `pyeventbt.events.events`. This entity appears to be legacy dead code.
2. **`IDataProvider` is not a true ABC.** It uses `raise NotImplementedError()` instead of `@abstractmethod`, so the compiler cannot enforce implementation at class definition time. It also imports `Protocol` but does not inherit from it.
3. **Missing `plaform_config` field on `MT5LiveDataConfig`.** The knowledge notes mention a `plaform_config` attribute, but the actual source does not include it -- the config only has `tradeable_symbol_list` and `timeframes_list`. If it was intended, it is missing; the typo "plaform" would also need correction.
4. **`BaseDataConfig` has no fields.** It is an empty `BaseModel`, meaning it does not enforce `tradeable_symbol_list` at the base level even though both subclasses define it independently.
5. **`backtest_end_timestamp` defaults to `datetime.now()` at import time**, which is evaluated once when the module is loaded, not when the config is instantiated. This is a well-known Python default-argument pitfall.
6. **`DataProvider.update_bars()` checks `self.trading_context == "BACKTEST"` (string comparison)**, but `trading_context` is set to a `TypeContext` enum value. This comparison will always be `False` unless the enum's `__eq__` handles string comparison, potentially breaking backtest state propagation (`continue_backtest`, `close_positions_end_of_data`).
7. **Multiple deprecated methods in CSV connector** (`get_latest_bar_old_lookahead_bias`, `get_latest_tick_old`, `_base_tf_bar_creates_new_tf_bar_old`, `_base_tf_bar_creates_new_tf_bar_f`, `get_latest_bars_pandas`, `get_latest_bar_old` in live connector) are retained in the codebase without deprecation markers.
8. **`_map_timeframe` in live connector** wraps the dict construction in `try/except KeyError`, but the `KeyError` would occur on the `return` line outside the `try` block, making the handler unreachable.
