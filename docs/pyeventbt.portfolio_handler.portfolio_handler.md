# pyeventbt.portfolio_handler.portfolio_handler

## File
`pyeventbt/portfolio_handler/portfolio_handler.py`

## Module
`pyeventbt.portfolio_handler.portfolio_handler`

## Purpose
Implements the `PortfolioHandler` class that orchestrates the full order pipeline: receiving events from the trading director, delegating to sizing/risk engines, updating portfolio state, archiving trades, and exporting backtest results.

## Tags
`portfolio-handler`, `orchestration`, `event-processing`, `sizing`, `risk`, `export`, `backtest`

## Dependencies

| Dependency | Usage |
|---|---|
| `queue.Queue` | Shared event queue for the event-driven architecture |
| `pyeventbt.backtest.core.backtest_results.BacktestResults` | Return type for `process_backtest_end` |
| `pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface.ISizingEngine` | Sizing engine interface |
| `pyeventbt.risk_engine.core.interfaces.risk_engine_interface.IRiskEngine` | Risk engine interface |
| `pyeventbt.trade_archiver.trade_archiver.TradeArchiver` | Trade archival for fill events |
| `pyeventbt.portfolio.core.interfaces.portfolio_interface.IPortfolio` | Portfolio interface |
| `pyeventbt.events.events` | `BarEvent`, `SignalEvent`, `FillEvent` |
| `pandas` | Used indirectly via portfolio exports |
| `pickle`, `zipfile`, `io` | Compressed pickle serialization |
| `os`, `logging`, `datetime` | File operations, logging, timestamps |

## Classes/Functions

### `PortfolioHandler`

**Signature:** `PortfolioHandler(events_queue: Queue, sizing_engine: ISizingEngine, risk_engine: IRiskEngine, portfolio: IPortfolio, base_timeframe: str = '1min', backtest_results_dir: str = None)`

**Description:** Central orchestrator in the event-driven pipeline. Handles three event types (bar, signal, fill) by delegating to the appropriate sub-system. Manages backtest result export at completion.

**Attributes:**

| Attribute | Type | Description |
|---|---|---|
| `event_queue` | `Queue` | Shared event queue |
| `POSITION_SIZER` | `ISizingEngine` | Sizing engine for converting signals to sized orders |
| `RISK_ENGINE` | `IRiskEngine` | Risk engine for validating/filtering orders |
| `PORTFOLIO` | `IPortfolio` | Portfolio for state management |
| `base_timeframe` | `str` | Base timeframe for bar event filtering (default `'1min'`) |
| `backtest_results_dir` | `str` | Optional custom directory for backtest exports |
| `TRADE_ARCHIVER` | `TradeArchiver` | Trade archiver instance (created internally) |

**Methods:**

| Method | Signature | Returns | Description |
|---|---|---|---|
| `process_bar_event` | `(bar_event: BarEvent) -> None` | `None` | Updates portfolio state; skips non-base-timeframe bars |
| `process_signal_event` | `(signal_event: SignalEvent) -> None` | `None` | Runs sizing engine then risk engine on the signal; risk engine emits ORDER event if approved |
| `process_fill_event` | `(fill_event: FillEvent) -> None` | `None` | Archives the filled trade via `TradeArchiver` |
| `process_backtest_end` | `(backtest_name: str, export_backtest_to_csv: bool = False, export_backtest_to_parquet: bool = False) -> BacktestResults` | `BacktestResults` | Finalizes portfolio, optionally exports to CSV/Parquet, returns `BacktestResults` with PnL DataFrame and trades DataFrame |
| `save_compressed_pickle` | `(backtest: tuple[str, str], path: str, password: str = None) -> None` | `None` | Serializes backtest data to a zip-compressed pickle file |
| `_get_default_desktop_path` | `() -> str` | `str` | Returns OS-specific Desktop path (macOS, Windows, Linux) |

## Data Flow

### Signal Processing Pipeline
```
SignalEvent
  -> process_signal_event()
      -> SizingEngine.get_suggested_order(signal_event) -> SuggestedOrder
      -> RiskEngine.assess_order(suggested_order)
          -> [if approved] RiskEngine emits OrderEvent to event_queue
```

### Bar Processing
```
BarEvent
  -> process_bar_event()
      -> [skip if timeframe != base_timeframe]
      -> Portfolio._update_portfolio(bar_event)
```

### Fill Processing
```
FillEvent
  -> process_fill_event()
      -> TradeArchiver.archive_trade(fill_event)
```

### Backtest End
```
process_backtest_end()
  -> Portfolio._update_portfolio_end_of_backtest()
  -> [if CSV] Export trades CSV + PnL CSV to Desktop/PyEventBT/backtest_results_csv/
  -> [if Parquet] Export trades Parquet + PnL Parquet to Desktop/PyEventBT/backtest_results_parquet/
  -> Return BacktestResults(backtest_pnl=DataFrame, trades=DataFrame)
```

## Gaps & Issues

1. `TradeArchiver` is instantiated directly in `__init__` rather than injected, making it difficult to mock or replace in tests.
2. `save_compressed_pickle` calls `zip_file.setpassword(password.encode())`, but Python's `zipfile` module does not support writing password-protected zip files -- this call has no effect on the output archive.
3. `save_compressed_pickle` does not guard against `password=None`, which would cause `AttributeError` when calling `.encode()` on `None`.
4. The CSV and Parquet export branches in `process_backtest_end` duplicate the directory path construction logic. This could be refactored into a shared helper.
5. The default export path (`Desktop/PyEventBT/...`) assumes a desktop environment and will fail or produce unexpected results in headless/server environments.
6. `BacktestResults` is always constructed (even when no export is requested), but the Parquet export happens after `BacktestResults` creation, meaning any export errors do not affect the return value.

## Requirements Derived

- **RQ-PH-001**: PortfolioHandler must process bar events only for the base timeframe, ignoring higher timeframes.
- **RQ-PH-002**: Signal events must pass through the sizing engine and then the risk engine before order placement.
- **RQ-PH-003**: Fill events must be archived for historical record-keeping.
- **RQ-PH-004**: At backtest end, the handler must finalize portfolio state and return a `BacktestResults` object containing PnL and trade DataFrames.
- **RQ-PH-005**: Backtest results must be optionally exportable to CSV and/or Parquet formats.
- **RQ-PH-006**: Export directory must default to the OS-specific Desktop path when no custom directory is configured.
