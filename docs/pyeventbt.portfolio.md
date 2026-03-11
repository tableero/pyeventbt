# pyeventbt.portfolio

## Package
`pyeventbt.portfolio`

## Purpose
Top-level package for portfolio state management. Tracks account balance, equity, realised/unrealised PnL, open positions, and pending orders. Provides historical data recording for backtests and export capabilities (Parquet, CSV, JSON, DataFrame).

## Tags
`portfolio`, `state-management`, `positions`, `pnl`, `backtest-history`, `export`

## Modules

| Module | Path | Description |
|---|---|---|
| `portfolio` | `portfolio/portfolio.py` | Concrete `Portfolio` implementation that tracks all account and position state |
| `core.interfaces.portfolio_interface` | `portfolio/core/interfaces/portfolio_interface.py` | `IPortfolio` interface defining the portfolio contract |
| `core.entities.open_position` | `portfolio/core/entities/open_position.py` | `OpenPosition` Pydantic model for open trade positions |
| `core.entities.closed_position` | `portfolio/core/entities/closed_position.py` | `ClosedPosition` Pydantic model for completed trades |
| `core.entities.pending_order` | `portfolio/core/entities/pending_order.py` | `PendingOrder` Pydantic model for unfilled orders |

## Internal Architecture

The package follows a layered structure:

- **`core/interfaces/`** -- Contains `IPortfolio`, the abstract interface that defines the portfolio contract. Uses `raise NotImplementedError` rather than ABC abstract methods.
- **`core/entities/`** -- Pydantic `BaseModel` data classes representing domain objects: `OpenPosition`, `ClosedPosition`, `PendingOrder`. These are pure data containers with no business logic.
- **`portfolio.py`** -- The concrete `Portfolio(IPortfolio)` implementation. Delegates to the `ExecutionEngine` for position/order state and account values. Maintains `historical_balance` and `historical_equity` dictionaries keyed by `datetime` for backtest recording.

Data flows inward: `Portfolio` receives `BarEvent` objects via `_update_portfolio()`, calls the execution engine to refresh fills and state, then updates its internal balance/equity/PnL fields. Historical snapshots are only recorded during backtests and only for the base timeframe.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.execution_engine.core.interfaces.execution_engine_interface.IExecutionEngine` | Portfolio delegates to execution engine for position queries, balance, and equity |
| `pyeventbt.trading_context.trading_context.TypeContext` | Enum distinguishing BACKTEST vs LIVE context |
| `pyeventbt.events.events.BarEvent` | Input event for portfolio updates |
| `pydantic` | BaseModel for all entity classes |
| `polars` | Parquet export via `pl.DataFrame` |
| `pandas` | DataFrame export and CSV export |

## Gaps & Issues

1. `IPortfolio` is not a true ABC (does not inherit from `abc.ABC` or use `@abstractmethod`); it uses `raise NotImplementedError` as a convention, which means the interface is not enforced at instantiation time.
2. `_export_historical_pnl_dataframe` in `IPortfolio` is missing the `self` parameter in its signature.
3. `ClosedPosition` entity exists and is imported in the interface module but is not actively used by the `Portfolio` class -- closed position tracking is handled by the `TradeArchiver` in the `portfolio_handler` package instead.
4. `get_closed_positions` is commented out in the interface, suggesting incomplete or deferred functionality.
5. Historical recording uses the first-seen symbol heuristic (`_first_seen_historical_symbol`) to avoid duplicate entries, which may produce incorrect results if symbols arrive in non-deterministic order.
