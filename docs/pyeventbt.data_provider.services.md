# Package: `pyeventbt.data_provider.services`

## Purpose

Contains the high-level service layer for data provisioning. The `DataProvider` facade hides connector selection behind the `IDataProvider` interface and bridges connectors to the shared event queue. The `QuantdleDataUpdater` is an independent utility for downloading and caching historical data from the Quantdle service.

## Tags

`services`, `facade`, `event-queue`, `quantdle`, `data-download`, `service-layer`

## Modules

| Module | Description |
|---|---|
| `data_provider_service.py` | `DataProvider` -- facade that selects CSV or MT5 connector, delegates all calls, and bridges `update_bars()` to the event queue (~72 lines) |
| `quantdle_data_updater.py` | `QuantdleDataUpdater` -- downloads historical OHLCV data from Quantdle API and manages a local CSV cache (~345 lines) |

## Internal Architecture

```
DataProvider (facade, implements IDataProvider)
    |
    +-- _get_data_provider(config)
    |     isinstance(config, MT5LiveDataConfig)  --> Mt5LiveDataProvider
    |     isinstance(config, CSVBacktestDataConfig) --> CSVDataProvider
    |
    +-- delegates get_latest_bar/bars/tick/bid/ask/datetime to connector
    |
    +-- update_bars():
          connector.update_bars() --> list[BarEvent]
          for each event: events_queue.put(event)
          if backtest: sync continue_backtest, close_positions_end_of_data

QuantdleDataUpdater (standalone, no IDataProvider dependency)
    |
    +-- update_data(csv_dir, symbols, start, end, tf)
          for each symbol:
            exists? --> _update_existing_csv (fill gaps before/after)
            new?    --> _create_new_csv (download full range)
```

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.data_provider.core.*` | Interface, configurations |
| `pyeventbt.data_provider.connectors.*` | `CSVDataProvider`, `Mt5LiveDataProvider` |
| `pyeventbt.trading_context.trading_context` | `TypeContext` enum |
| `pyeventbt.events.events` | `BarEvent` |
| `queue.Queue` | Shared event bus |
| `quantdle` | External package for data downloads (conditionally imported) |
| `polars` | CSV reading/writing in `QuantdleDataUpdater` |

## Gaps & Issues

1. **`DataProvider.update_bars()` string comparison bug.** Line 66 checks `self.trading_context == "BACKTEST"` but `trading_context` is a `TypeContext` enum instance, so the comparison may always be `False`.
2. **`QuantdleDataUpdater` uses emoji in log messages** (checkmark, cross), which may not render in all environments.
3. **No error recovery in `DataProvider.update_bars()`** if the connector's `update_bars()` raises an exception mid-iteration; partial events already queued would not be rolled back.
