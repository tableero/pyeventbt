# File: `pyeventbt/data_provider/core/interfaces/data_provider_interface.py`

## Module

`pyeventbt.data_provider.core.interfaces.data_provider_interface`

## Purpose

Defines the `IDataProvider` interface that all data providers (backtest CSV, live MT5, and the service facade) must implement. Establishes the contract for bar retrieval, tick data access, and bar update/emission. This is the central abstraction enabling backtest and live trading to share the same engine pipeline.

## Tags

`interface`, `abstract`, `data-provider`, `contract`, `bar-retrieval`, `tick-data`

## Dependencies

| Dependency | Usage |
|---|---|
| `typing.Protocol` | Imported but **unused** -- the class does not inherit from `Protocol` |
| `pyeventbt.events.events.BarEvent` | Return type annotation for `update_bars()` |
| `decimal.Decimal` | Return type for `get_latest_bid()` and `get_latest_ask()` |

## Classes/Functions

### `IDataProvider`

Base class defining the data provider contract. Implementors: `CSVDataProvider`, `Mt5LiveDataProvider`, `DataProvider` (service facade).

**Signature:**
```python
class IDataProvider:
```

**Note:** Does not inherit from `abc.ABC` or `typing.Protocol`. Methods raise `NotImplementedError()` at runtime rather than being enforced at class-definition time.

#### Methods

---

##### `get_latest_bar(symbol: str, timeframe: str)`

Returns the most recent closed bar for the given symbol and timeframe.

| Parameter | Type | Description |
|---|---|---|
| `symbol` | `str` | Trading symbol (e.g., `"EURUSD"`) |
| `timeframe` | `str` | Timeframe string (e.g., `"1min"`, `"1H"`) |

**Returns:** Implementation-dependent. CSV connector returns `BarEvent`; live connector returns `BarEvent | None`.

---

##### `get_latest_bars(symbol: str, timeframe: str, N: int)`

Returns the N most recent bars.

| Parameter | Type | Description |
|---|---|---|
| `symbol` | `str` | Trading symbol |
| `timeframe` | `str` | Timeframe string |
| `N` | `int` | Number of bars to retrieve |

**Returns:** `pl.DataFrame` (CSV connector) or `pl.DataFrame | None` (live connector).

---

##### `get_latest_tick(symbol: str) -> dict`

Returns the latest tick data as a dictionary with MT5 tick structure keys: `time`, `bid`, `ask`, `last`, `volume`, `time_msc`, `flags`, `volume_real`.

| Parameter | Type | Description |
|---|---|---|
| `symbol` | `str` | Trading symbol |

**Returns:** `dict` with `Decimal` values for `bid`, `ask`, `last`, `volume_real`.

---

##### `get_latest_bid(symbol: str) -> Decimal`

Returns the latest bid price.

| Parameter | Type | Description |
|---|---|---|
| `symbol` | `str` | Trading symbol |

**Returns:** `Decimal`

---

##### `get_latest_ask(symbol: str) -> Decimal`

Returns the latest ask price.

| Parameter | Type | Description |
|---|---|---|
| `symbol` | `str` | Trading symbol |

**Returns:** `Decimal`

---

##### `get_latest_datetime(symbol: str, timeframe: str)`

Returns the timestamp of the last bar.

| Parameter | Type | Description |
|---|---|---|
| `symbol` | `str` | Trading symbol |
| `timeframe` | `str` | Timeframe string |

**Returns:** `datetime` (implementation-dependent; some return `pd.Timestamp`).

---

##### `update_bars() -> list[BarEvent] | None`

Advances the data feed. In connectors, returns a `list[BarEvent]`. In the service layer, returns `None` (events are placed directly onto the queue).

**Returns:** `list[BarEvent]` (connectors) or `None` (service).

## Data Flow

```
TradingDirector (event loop)
    --> DataProvider.update_bars()  [service, returns None]
        --> CSVDataProvider.update_bars()  [connector, returns list[BarEvent]]
            --> generator yields BarEvent per base-TF bar
            --> checks higher-TF boundaries, emits additional BarEvents
        --> events put onto Queue by service layer

User signal engine callbacks
    --> Modules.DATA_PROVIDER.get_latest_bars(symbol, tf, N)
    --> Modules.DATA_PROVIDER.get_latest_tick(symbol)
```

## Gaps & Issues

1. **Not a true abstract class.** `Protocol` is imported but not used as a base class. Methods raise `NotImplementedError()` at runtime rather than using `@abstractmethod`, so missing implementations are only caught when called.
2. **Undeclared instance attributes.** The service layer accesses `continue_backtest` and `close_positions_end_of_data` on the interface, but these are not declared in `IDataProvider`.
3. **Inconsistent return types across implementations.** `get_latest_bar` returns `BarEvent` in CSV connector but `BarEvent | None` in live connector. `get_latest_datetime` returns `datetime` in CSV but `pd.Timestamp` in live.
4. **Commented-out code.** Line 71-72 contains a commented-out `get_latest_bar_datetime` method and a Spanish comment ("Implementar como servicio").
5. **`timeframe` parameter is not optional in the interface** but is declared as `Optional` with a default in the concrete implementations, creating a signature mismatch.

## Requirements Derived

- **REQ-DP-IF-001:** Convert `IDataProvider` to a proper `abc.ABC` with `@abstractmethod` decorators to enforce implementation at class-definition time.
- **REQ-DP-IF-002:** Declare `continue_backtest` and `close_positions_end_of_data` as attributes (or properties) in the interface.
- **REQ-DP-IF-003:** Standardize return types across all implementations (e.g., `get_latest_bar` should consistently return `BarEvent | None`).
- **REQ-DP-IF-004:** Add `timeframe: str | None = None` default to the interface signature to match implementations.
- **REQ-DP-IF-005:** Remove unused `Protocol` import.
