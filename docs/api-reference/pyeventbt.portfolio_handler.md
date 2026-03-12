# pyeventbt.portfolio_handler

## Package
`pyeventbt.portfolio_handler`

## Purpose
Orchestration layer between the event loop and the portfolio/execution pipeline. Receives bar, signal, and fill events from the `TradingDirector` and coordinates the sizing engine, risk engine, execution engine, and trade archiver to process them.

## Tags
`portfolio-handler`, `orchestration`, `event-processing`, `pipeline`, `backtest-export`

## Modules

| Module | Path | Description |
|---|---|---|
| `portfolio_handler` | `portfolio_handler/portfolio_handler.py` | Main `PortfolioHandler` class that orchestrates the sizing -> risk -> execution pipeline |
| `core.entities.suggested_order` | `portfolio_handler/core/entities/suggested_order.py` | `SuggestedOrder` Pydantic model representing a sized order before risk assessment |

## Internal Architecture

The `PortfolioHandler` is the central orchestrator in the event-driven pipeline. It sits between the `TradingDirector` (event dispatcher) and the individual engines:

1. **Bar events** -> `process_bar_event()` -> delegates to `Portfolio._update_portfolio()` (base timeframe only)
2. **Signal events** -> `process_signal_event()` -> `SizingEngine.get_suggested_order()` -> `RiskEngine.assess_order()` -> (risk engine emits ORDER event to queue)
3. **Fill events** -> `process_fill_event()` -> `TradeArchiver.archive_trade()`

At backtest end, `process_backtest_end()` finalizes the portfolio state, optionally exports results to CSV/Parquet, and returns a `BacktestResults` object.

The `core/entities/` subfolder contains `SuggestedOrder`, which is the intermediate data object passed from the sizing engine to the risk engine.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.portfolio.core.interfaces.portfolio_interface.IPortfolio` | Portfolio for state updates |
| `pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface.ISizingEngine` | Sizing engine for order sizing |
| `pyeventbt.risk_engine.core.interfaces.risk_engine_interface.IRiskEngine` | Risk engine for order validation |
| `pyeventbt.trade_archiver.trade_archiver.TradeArchiver` | Archives fill events for historical records |
| `pyeventbt.backtest.core.backtest_results.BacktestResults` | Return type for backtest completion |
| `pyeventbt.events.events` | `BarEvent`, `SignalEvent`, `FillEvent` event types |
| `queue.Queue` | Shared event queue |

## Gaps & Issues

1. `PortfolioHandler` directly instantiates `TradeArchiver()` in `__init__` rather than receiving it via dependency injection, reducing testability.
2. The `save_compressed_pickle` method sets a zip password via `setpassword()`, but Python's `zipfile` module does not support password-protected writing -- `setpassword()` only affects reading. The method would not actually password-protect the output.
3. Export directory naming includes a timestamp (`datetime.now()`) which makes outputs non-deterministic and harder to test.
4. The CSV and Parquet export branches duplicate the directory setup logic (determining base path, creating timestamp, building export path).
