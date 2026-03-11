# Package: `pyeventbt.data_provider.core`

## Purpose

Contains the foundational domain layer for the data provider module: entity models, the abstract data provider interface, and configuration schemas. This sub-package defines *what* a data provider is and how it is configured, without any implementation details.

## Tags

`core`, `interface`, `configuration`, `entity`, `domain-model`, `abstract`

## Modules

| Module | Description |
|---|---|
| `entities/bar.py` | Pydantic `Bar` model -- legacy/unused entity representing a financial OHLCV bar |
| `interfaces/data_provider_interface.py` | `IDataProvider` base class defining the contract for all data providers |
| `configurations/data_provider_configurations.py` | Pydantic configuration models (`BaseDataConfig`, `MT5LiveDataConfig`, `CSVBacktestDataConfig`) |

## Internal Architecture

```
core/
  entities/
    bar.py              --> Bar(BaseModel)  [UNUSED]
  interfaces/
    data_provider_interface.py  --> IDataProvider (pseudo-ABC)
  configurations/
    data_provider_configurations.py --> BaseDataConfig
                                       +-- MT5LiveDataConfig
                                       +-- CSVBacktestDataConfig
```

The `IDataProvider` interface is implemented by:
- `connectors/CSVDataProvider` (backtest)
- `connectors/Mt5LiveDataProvider` (live)
- `services/DataProvider` (facade that delegates to a connector)

Configuration objects are passed into the service-layer `DataProvider.__init__`, which uses `isinstance()` checks to select the appropriate connector.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base for `Bar` entity and all configuration models |
| `pyeventbt.events.events.BarEvent` | Return type annotation in `IDataProvider.update_bars()` |
| `decimal.Decimal` | Return type for `get_latest_bid` / `get_latest_ask` |

## Gaps & Issues

1. **`entities/bar.py` is dead code.** The import in `data_provider_interface.py` is commented out (`#from ..entities.bar import Bar`). All runtime code uses `Bar` from `pyeventbt.events.events` instead.
2. **`IDataProvider` is not enforced as abstract.** Methods raise `NotImplementedError()` at runtime rather than using `abc.ABC` + `@abstractmethod`. The `Protocol` import is unused.
3. **`BaseDataConfig` is an empty model** with no shared fields, even though both subclasses define `tradeable_symbol_list` independently.
4. **`CSVBacktestDataConfig.backtest_end_timestamp`** defaults to `datetime.now()` evaluated at module import time, not at instantiation time.
5. **`IDataProvider` has undeclared instance attributes** (`continue_backtest`, `close_positions_end_of_data`) that are accessed by the service layer but not defined in the interface.
