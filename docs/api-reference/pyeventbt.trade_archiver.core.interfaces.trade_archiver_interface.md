# pyeventbt.trade_archiver.core.interfaces.trade_archiver_interface

- **File**: `pyeventbt/trade_archiver/core/interfaces/trade_archiver_interface.py`
- **Module**: `pyeventbt.trade_archiver.core.interfaces.trade_archiver_interface`
- **Purpose**: Defines the `ITradeArchiver` protocol specifying the contract for trade archival implementations.
- **Tags**: `interface`, `protocol`, `trade-archiver`, `contract`

## Dependencies

| Dependency | Usage |
|---|---|
| `typing.Protocol` | Base class for structural typing |
| `pyeventbt.events.events.FillEvent` | Event type used in method signatures |

## Classes/Functions

### ITradeArchiver

- **Signature**: `class ITradeArchiver(Protocol)`
- **Description**: Protocol class defining the minimum interface for trade archiver implementations. Uses `typing.Protocol` for structural subtyping, though the concrete `TradeArchiver` explicitly inherits from it (nominal subtyping).

#### Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `archive_trade` | `archive_trade(self, fill_event: FillEvent) -> None` | Store a FillEvent in the archive. Raises `NotImplementedError` if not overridden. | `None` |
| `get_trade_archive` | `get_trade_archive(self) -> dict[int, FillEvent]` | Retrieve the full trade archive as a dict mapping integer IDs to FillEvents. Raises `NotImplementedError` if not overridden. | `dict[int, FillEvent]` |
| `export_csv_trade_archive` | `export_csv_trade_archive(self, file_path: str) -> None` | Export the trade archive to a CSV file at the given path. Raises `NotImplementedError` if not overridden. | `None` |

## Data Flow

1. `ITradeArchiver` is imported by `TradeArchiver` as its base class.
2. The framework's internal components (e.g., `PortfolioHandler`) type-hint against `ITradeArchiver` to allow dependency injection of any conforming archiver implementation.
3. `TradeArchiver` provides the concrete implementation used in both backtest and live modes.

## Gaps & Issues

1. **Incomplete interface**: The protocol only declares 3 of the 6 methods that `TradeArchiver` implements. Missing from the interface: `export_historical_trades_dataframe()`, `export_historical_trades_json()`, `export_historical_trades_parquet()`.
2. **Protocol with NotImplementedError**: Method bodies raise `NotImplementedError`, which is an ABC pattern. With `Protocol`, the standard pattern is to use `...` (ellipsis) as the body. The `NotImplementedError` approach works but is unconventional for protocols.
3. **No `@abstractmethod` or `@runtime_checkable`**: The protocol is not decorated with `@runtime_checkable`, so `isinstance()` checks against `ITradeArchiver` will fail at runtime. Methods are not marked `@abstractmethod` (which is optional for protocols but can add clarity).
4. **Nominal vs structural**: Since `TradeArchiver` explicitly inherits `ITradeArchiver`, the benefit of using `Protocol` (structural/duck typing) over `ABC` is not realized.

## Requirements Derived

- R-TAI-01: Any trade archiver must implement `archive_trade(FillEvent) -> None`.
- R-TAI-02: Any trade archiver must implement `get_trade_archive() -> dict[int, FillEvent]`.
- R-TAI-03: Any trade archiver must implement `export_csv_trade_archive(file_path: str) -> None`.
