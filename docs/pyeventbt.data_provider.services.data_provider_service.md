# File: `pyeventbt/data_provider/services/data_provider_service.py`

## Module

`pyeventbt.data_provider.services.data_provider_service`

## Purpose

Provides the `DataProvider` service facade that the rest of the framework interacts with. Selects the appropriate connector based on the configuration type, delegates all data retrieval calls, and bridges the connector's `update_bars()` output to the shared event queue. This is the class exposed to user callbacks via `Modules.DATA_PROVIDER`.

## Tags

`service`, `facade`, `delegation`, `event-queue`, `factory`, `data-provider`

## Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.data_provider.core.interfaces.data_provider_interface` | `IDataProvider` base class |
| `pyeventbt.data_provider.connectors.csv_data_connector` | `CSVDataProvider` |
| `pyeventbt.data_provider.connectors.mt5_live_data_connector` | `Mt5LiveDataProvider` |
| `pyeventbt.data_provider.core.configurations.data_provider_configurations` | `BaseDataConfig`, `MT5LiveDataConfig`, `CSVBacktestDataConfig` |
| `pyeventbt.trading_context.trading_context` | `TypeContext` enum |
| `pyeventbt.events.events` | `BarEvent` |
| `queue.Queue` | Shared event bus |
| `pandas` | Imported for type hints (`pd.Series`, `pd.DataFrame`, `pd.Timestamp`) |
| `decimal.Decimal` | Return type for bid/ask |

## Classes/Functions

### `DataProvider(IDataProvider)`

**Signature:**
```python
class DataProvider(IDataProvider):
    def __init__(self, events_queue: Queue, data_config: BaseDataConfig,
                 trading_context: trading_context.TypeContext = trading_context.TypeContext.BACKTEST) -> None
```

**Instance Attributes:**

| Attribute | Type | Description |
|---|---|---|
| `trading_context` | `TypeContext` | Backtest or live context |
| `events_queue` | `Queue` | Shared event bus for `BarEvent` emission |
| `DATA_PROVIDER` | `IDataProvider` | Underlying connector instance |
| `continue_backtest` | `bool` | Initially `True`; synced from connector in backtest mode |
| `close_positions_end_of_data` | `bool` | Initially `False`; synced from connector in backtest mode |

#### Methods

---

##### `__init__(self, events_queue, data_config, trading_context) -> None`

Stores context and queue reference. Calls `_get_data_provider(data_config)` to instantiate the appropriate connector.

---

##### `_get_data_provider(self, data_config: BaseDataConfig) -> IDataProvider`

Factory method. Uses `isinstance()` dispatch:
- `MT5LiveDataConfig` -> `Mt5LiveDataProvider(configs=data_config)`
- `CSVBacktestDataConfig` -> `CSVDataProvider(configs=data_config)`
- Otherwise -> raises `Exception`

---

##### `get_latest_bar(self, symbol: str, timeframe: str) -> pd.Series`

Delegates to `self.DATA_PROVIDER.get_latest_bar(symbol, timeframe)`.

**Note:** Return type annotation says `pd.Series` but the underlying connectors now return `BarEvent`. The annotation is stale.

---

##### `get_latest_bars(self, symbol: str, timeframe: str = None, N: int = 2) -> pd.DataFrame`

Delegates to connector. Return type annotation says `pd.DataFrame` but connectors return `pl.DataFrame`.

---

##### `get_latest_tick(self, symbol: str) -> dict`

Delegates to connector.

---

##### `get_latest_bid(self, symbol: str) -> Decimal`

Delegates to connector.

---

##### `get_latest_ask(self, symbol: str) -> Decimal`

Delegates to connector.

---

##### `get_latest_datetime(self, symbol: str, timeframe: str = None) -> pd.Timestamp`

Delegates to connector. Return type annotation says `pd.Timestamp` but connectors return `datetime`.

---

##### `update_bars(self) -> None`

Core event-pumping method called by the `TradingDirector` when the queue is empty:
1. Calls `self.DATA_PROVIDER.update_bars()` which returns `list[BarEvent]`.
2. Iterates the list, putting each `BarEvent` onto `self.events_queue`.
3. If in backtest context (checked via `self.trading_context == "BACKTEST"`), syncs `close_positions_end_of_data` and `continue_backtest` from the connector.

**Returns:** `None`

---

##### `_put_bar_event(self, bar_event: BarEvent) -> None`

Puts a single `BarEvent` onto `self.events_queue`.

## Data Flow

```
TradingDirector (queue is empty)
    --> DataProvider.update_bars()
        --> CSVDataProvider.update_bars() or Mt5LiveDataProvider.update_bars()
            --> returns list[BarEvent]
        --> for each BarEvent:
            --> _put_bar_event(event) --> events_queue.put(event)
        --> if backtest: sync state flags from connector

User signal engine / sizing engine
    --> Modules.DATA_PROVIDER.get_latest_bars("EURUSD", "1H", 50)
        --> DataProvider.get_latest_bars(...)
            --> CSVDataProvider.get_latest_bars(...)
                --> returns pl.DataFrame
```

## Gaps & Issues

1. **String comparison for context check.** Line 66: `if self.trading_context == "BACKTEST"` compares a `TypeContext` enum against a string literal. Unless `TypeContext` implements `__eq__` for strings, this will always be `False`, meaning `continue_backtest` and `close_positions_end_of_data` are never synced from the connector.
2. **Stale return type annotations.** `get_latest_bar` is annotated as `-> pd.Series`, `get_latest_bars` as `-> pd.DataFrame`, and `get_latest_datetime` as `-> pd.Timestamp`. The actual return types from connectors are `BarEvent`, `pl.DataFrame`, and `datetime` respectively.
3. **`pandas` imported but unused at runtime.** Only used in type annotations which are already incorrect.
4. **No error handling in `update_bars()`.** If the connector raises an exception mid-iteration, events already queued are not rolled back, potentially leaving the system in an inconsistent state.
5. **`DATA_PROVIDER` naming convention.** Uses UPPER_CASE for an instance attribute, which by Python convention suggests a constant. Should be `_data_provider` or `data_provider`.

## Requirements Derived

- **REQ-DP-SVC-001:** Fix the backtest context check to compare against the `TypeContext` enum value (e.g., `trading_context.TypeContext.BACKTEST`).
- **REQ-DP-SVC-002:** Update return type annotations to match actual connector return types (`BarEvent`, `pl.DataFrame`, `datetime`).
- **REQ-DP-SVC-003:** Remove unused `pandas` import or keep only if type annotations are corrected.
- **REQ-DP-SVC-004:** Add exception handling in `update_bars()` to handle connector failures gracefully.
- **REQ-DP-SVC-005:** Rename `DATA_PROVIDER` to follow Python naming conventions for instance attributes.
