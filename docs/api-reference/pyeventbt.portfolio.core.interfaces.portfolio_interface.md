# pyeventbt.portfolio.core.interfaces.portfolio_interface

## File
`pyeventbt/portfolio/core/interfaces/portfolio_interface.py`

## Module
`pyeventbt.portfolio.core.interfaces.portfolio_interface`

## Purpose
Defines the `IPortfolio` interface class that establishes the contract for all portfolio implementations. Provides method stubs that raise `NotImplementedError` for unimplemented methods.

## Tags
`interface`, `portfolio`, `contract`, `abstract`

## Dependencies

| Dependency | Usage |
|---|---|
| `decimal.Decimal` | Return type for balance/equity/PnL methods |
| `typing.Protocol` | Imported but not used as base class |
| `pandas` | Return type for `_export_historical_pnl_dataframe` |
| `pyeventbt.events.events.BarEvent` | Parameter type for `_update_portfolio` |
| `pyeventbt.portfolio.core.entities.open_position.OpenPosition` | Return type for `get_positions` |
| `pyeventbt.portfolio.core.entities.closed_position.ClosedPosition` | Imported but currently unused (commented-out method) |
| `pyeventbt.portfolio.core.entities.pending_order.PendingOrder` | Return type for `get_pending_orders` |

## Classes/Functions

### `IPortfolio`

**Signature:** `class IPortfolio` (plain class, not ABC)

**Description:** Interface defining the portfolio contract. All methods raise `NotImplementedError`. Concrete implementations (e.g., `Portfolio`) override these methods. Despite importing `Protocol`, it does not use structural subtyping -- `Portfolio` inherits from `IPortfolio` explicitly.

**Methods:**

| Method | Signature | Returns | Description |
|---|---|---|---|
| `_update_portfolio` | `(bar_event: BarEvent) -> None` | `None` | Update portfolio state from a new bar event |
| `_update_portfolio_end_of_backtest` | `() -> None` | `None` | Final update at backtest completion |
| `get_positions` | `(symbol: str = '', ticket: int = None) -> tuple[OpenPosition]` | `tuple[OpenPosition]` | Get open positions, optionally filtered |
| `get_pending_orders` | `(symbol: str = '', ticket: int = None) -> tuple[PendingOrder]` | `tuple[PendingOrder]` | Get pending orders, optionally filtered |
| `get_number_of_strategy_open_positions_by_symbol` | `(symbol: str) -> dict[str, int]` | `dict[str, int]` | Count open positions by direction for a symbol |
| `get_number_of_strategy_pending_orders_by_symbol` | `(symbol: str) -> dict[str, int]` | `dict[str, int]` | Count pending orders by type for a symbol |
| `get_account_balance` | `() -> Decimal` | `Decimal` | Get current account balance |
| `get_account_equity` | `() -> Decimal` | `Decimal` | Get current account equity |
| `get_account_unrealised_pnl` | `() -> Decimal` | `Decimal` | Get current unrealised PnL |
| `get_account_realised_pnl` | `() -> Decimal` | `Decimal` | Get cumulative realised PnL |
| `_export_historical_pnl_dataframe` | `() -> pd.DataFrame` | `pd.DataFrame` | Export historical PnL as DataFrame |
| `_export_historical_pnl_json` | `() -> str` | `str` | Export historical PnL as JSON string |
| `_export_csv_historical_pnl` | `(file_path: str) -> None` | `None` | Export historical PnL to CSV file |

## Data Flow

`IPortfolio` does not implement data flow -- it defines the contract. See `pyeventbt.portfolio.portfolio` for the concrete implementation.

## Gaps & Issues

1. `IPortfolio` does not inherit from `abc.ABC` and does not use `@abstractmethod`, so the interface is not enforced at class instantiation. `Protocol` is imported but unused as a base class.
2. `_export_historical_pnl_dataframe` is missing the `self` parameter in its signature.
3. `get_closed_positions` method is commented out, leaving an incomplete interface for closed trade queries.
4. Private methods (`_update_portfolio`, `_export_*`) are part of the interface, blurring the line between public API and internal implementation detail.
5. The docstring references `SignalEvents` and `DataHandler` concepts that are not part of the portfolio's responsibility, suggesting it was copied from another context.

## Requirements Derived

- **RQ-IPRT-001**: Any portfolio implementation must provide methods for querying balance, equity, realised PnL, and unrealised PnL.
- **RQ-IPRT-002**: Any portfolio implementation must support position and pending order queries with optional symbol and ticket filters.
- **RQ-IPRT-003**: Any portfolio implementation must support portfolio state updates from bar events.
- **RQ-IPRT-004**: Any portfolio implementation must support historical PnL export in DataFrame, JSON, and CSV formats.
