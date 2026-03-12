# pyeventbt.portfolio.portfolio

## File
`pyeventbt/portfolio/portfolio.py`

## Module
`pyeventbt.portfolio.portfolio`

## Purpose
Concrete portfolio implementation that tracks account state (balance, equity, PnL), open positions, and pending orders. Delegates to the execution engine for actual position/order data and records historical snapshots during backtests. Provides multiple export formats for historical PnL data.

## Tags
`portfolio`, `state-management`, `pnl`, `backtest-history`, `export`, `parquet`, `csv`, `json`

## Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.trading_context.trading_context.TypeContext` | Enum for BACKTEST vs LIVE context |
| `pyeventbt.portfolio.core.interfaces.portfolio_interface.IPortfolio` | Parent interface |
| `pyeventbt.portfolio.core.entities.open_position.OpenPosition` | Position entity type |
| `pyeventbt.portfolio.core.entities.pending_order.PendingOrder` | Pending order entity type |
| `pyeventbt.execution_engine.core.interfaces.execution_engine_interface.IExecutionEngine` | Execution engine for state queries |
| `pyeventbt.events.events.BarEvent` | Bar event input type |
| `decimal.Decimal` | Precision arithmetic for financial values |
| `pandas` | DataFrame export |
| `polars` | Parquet export |

## Classes/Functions

### `Portfolio(IPortfolio)`

**Signature:** `Portfolio(initial_balance: Decimal, execution_engine: IExecutionEngine, trading_context: TypeContext = TypeContext.BACKTEST, base_timeframe: str = '1min')`

**Description:** Tracks the full state of a trading account -- balance, equity, realised/unrealised PnL, open positions, and pending orders. On each bar, it delegates to the execution engine to check for fills, SL/TP hits, and refreshes all state. In backtest mode, it records historical balance and equity keyed by datetime for post-backtest analysis.

**Attributes:**

| Attribute | Type | Description |
|---|---|---|
| `trading_context` | `TypeContext` | BACKTEST or LIVE mode |
| `EXECUTION` | `IExecutionEngine` | Reference to the execution engine |
| `_initial_balance` | `Decimal` | Starting account balance |
| `_balance` | `Decimal` | Current account balance |
| `_equity` | `Decimal` | Current account equity |
| `_realised_pnl` | `Decimal` | Cumulative realised profit/loss |
| `_unrealised_pnl` | `Decimal` | Current unrealised profit/loss |
| `_strategy_positions` | `tuple[OpenPosition]` | Currently open positions |
| `_strategy_pending_orders` | `tuple[PendingOrder]` | Currently pending orders |
| `_base_timeframe` | `str` | Base timeframe for updates (default `'1min'`) |
| `historical_balance` | `dict[datetime, Decimal]` | Backtest balance history |
| `historical_equity` | `dict[datetime, Decimal]` | Backtest equity history |

**Methods:**

| Method | Signature | Returns | Description |
|---|---|---|---|
| `_update_portfolio` | `(bar_event: BarEvent) -> None` | `None` | Refreshes all state from execution engine; records historical snapshots in backtest mode (base timeframe only) |
| `_update_portfolio_end_of_backtest` | `() -> None` | `None` | Final state update at backtest end; logs realised PnL with color coding |
| `get_account_balance` | `() -> Decimal` | `Decimal` | Returns current balance |
| `get_account_equity` | `() -> Decimal` | `Decimal` | Returns current equity |
| `get_account_unrealised_pnl` | `() -> Decimal` | `Decimal` | Returns unrealised PnL |
| `get_account_realised_pnl` | `() -> Decimal` | `Decimal` | Returns realised PnL |
| `get_positions` | `(symbol: str = '', ticket: int = None) -> tuple[OpenPosition]` | `tuple[OpenPosition]` | Returns open positions, optionally filtered by symbol and/or ticket |
| `get_pending_orders` | `(symbol: str = '', ticket: int = None) -> tuple[PendingOrder]` | `tuple[PendingOrder]` | Returns pending orders, optionally filtered by symbol and/or ticket |
| `get_number_of_strategy_open_positions_by_symbol` | `(symbol: str) -> dict[str, int]` | `dict` | Returns `{"LONG": int, "SHORT": int, "TOTAL": int}` for a given symbol |
| `get_number_of_strategy_pending_orders_by_symbol` | `(symbol: str) -> dict[str, int]` | `dict` | Returns `{"BUY_LIMIT": int, "SELL_LIMIT": int, "BUY_STOP": int, "SELL_STOP": int, "TOTAL": int}` |
| `_export_historical_pnl_dataframe` | `() -> pd.DataFrame` | `pd.DataFrame` | Returns DataFrame with BALANCE and EQUITY columns, datetime index |
| `_export_historical_pnl_to_parquet` | `(file_path: str) -> None` | `None` | Exports historical PnL to Parquet (zstd compression, level 10); rounds to 2 decimal places |
| `_export_historical_pnl_json` | `() -> str` | `str` | Returns JSON string with balance/equity scaled to integers (4 decimal precision) |
| `_export_csv_historical_pnl` | `(file_path: str) -> None` | `None` | Exports historical PnL to CSV via pandas |

## Data Flow

```
BarEvent
  -> _update_portfolio()
      -> ExecutionEngine._update_values_and_check_executions_and_fills(bar_event)
      -> ExecutionEngine._get_strategy_positions() -> _strategy_positions
      -> ExecutionEngine._get_strategy_pending_orders() -> _strategy_pending_orders
      -> ExecutionEngine._get_account_balance() -> _balance
      -> ExecutionEngine._get_account_equity() -> _equity
      -> Compute _realised_pnl = _balance - _initial_balance
      -> Compute _unrealised_pnl = _equity - _balance
      -> [BACKTEST only] Record historical_balance[datetime], historical_equity[datetime]
```

At backtest end:
```
_update_portfolio_end_of_backtest()
  -> Refresh balance/equity from ExecutionEngine
  -> Update last historical entries (equity set equal to balance after closing all positions)
  -> Log realised PnL with color coding (green for profit, red for loss)
```

## Gaps & Issues

1. The `_export_historical_pnl_json` method returns `{}` (an empty dict) on serialization error instead of `""` (empty string), which is inconsistent with the `-> str` return type annotation.
2. Historical data is only recorded for the first-seen symbol to avoid redundancy. This heuristic could produce incorrect results if symbol arrival order is non-deterministic.
3. All export methods (`_export_*`) are prefixed with underscore, suggesting they are private, but they are called externally by `PortfolioHandler`.
4. The Parquet export uses `polars` while CSV/DataFrame exports use `pandas`, creating a dual-library dependency for similar functionality.

## Requirements Derived

- **RQ-PORT-001**: Portfolio must track balance, equity, realised PnL, and unrealised PnL using `Decimal` precision.
- **RQ-PORT-002**: Portfolio must delegate to the execution engine for position/order state and account values on every base-timeframe bar.
- **RQ-PORT-003**: In backtest mode, portfolio must record historical balance and equity at each base-timeframe bar.
- **RQ-PORT-004**: Portfolio must support filtering positions and pending orders by symbol and ticket.
- **RQ-PORT-005**: Portfolio must support exporting historical PnL in Parquet (zstd), CSV, JSON, and DataFrame formats.
- **RQ-PORT-006**: At backtest end, portfolio must update final state and set equity equal to balance (all positions closed).
