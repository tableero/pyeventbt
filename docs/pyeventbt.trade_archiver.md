# pyeventbt.trade_archiver

- **Package**: `pyeventbt.trade_archiver`
- **Purpose**: Provides trade archival and export capabilities. Stores executed trades (FillEvents) and supports export to multiple formats (DataFrame, JSON, Parquet, CSV).
- **Tags**: `trade-archiver`, `storage`, `export`, `serialization`, `package-init`

## Modules

| Module | Description |
|---|---|
| `trade_archiver.trade_archiver` | `TradeArchiver` implementation -- stores and exports FillEvents |
| `trade_archiver.core.interfaces.trade_archiver_interface` | `ITradeArchiver` protocol defining the archiver contract |

## Internal Architecture

```
trade_archiver/
  __init__.py                               # Empty (no re-exports)
  trade_archiver.py                         # TradeArchiver implementation
  core/
    interfaces/
      trade_archiver_interface.py           # ITradeArchiver protocol
```

`TradeArchiver` implements `ITradeArchiver` and is used internally by the framework's execution pipeline. When a `FillEvent` is produced by the execution engine, `PortfolioHandler` calls `archive_trade()` to store it.

Note: `__init__.py` is empty -- it does not re-export `TradeArchiver` or `ITradeArchiver`.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events.events.FillEvent` | The event type stored in the archive |
| `pandas` | DataFrame construction for `export_historical_trades_dataframe()` and CSV export |
| `polars` | DataFrame construction for Parquet export with schema enforcement |
| `decimal.Decimal` | Precision formatting of financial values during JSON/Parquet export |
| `json` | JSON serialization |
| `os` | Directory creation for file exports |
| `logging` | Error and debug logging |

## Gaps & Issues

1. **Empty `__init__.py`**: The package init does not re-export `TradeArchiver`, requiring users/internal code to import directly from `trade_archiver.trade_archiver`.
2. **FIXME on deprecated pandas pattern**: Line 65 of `trade_archiver.py` has a `FIXME` comment noting that the row-accumulation approach was historically using deprecated pandas `append`. While the code now uses list accumulation (which is correct), the FIXME comment remains.
3. **Interface incompleteness**: `ITradeArchiver` only declares `archive_trade`, `get_trade_archive`, and `export_csv_trade_archive`. The JSON, Parquet, and DataFrame export methods are not part of the interface.
4. **Protocol vs ABC**: `ITradeArchiver` uses `typing.Protocol` rather than `abc.ABC`, but raises `NotImplementedError` in method bodies (a pattern more typical of ABCs). Since `TradeArchiver` explicitly inherits from `ITradeArchiver`, the Protocol approach provides no structural typing benefit.
