# Package: `pyeventbt.trading_director`

## Purpose

Contains the central event loop that drives both backtesting and live trading. The `TradingDirector` dequeues events from the shared `queue.Queue`, dispatches them to the appropriate handler, and coordinates the overall lifecycle of a trading session (start, run, end).

## Tags

`event-loop`, `core`, `orchestration`, `backtest`, `live-trading`, `dispatcher`

## Modules

| Module | Path | Description |
|---|---|---|
| `trading_director` | `trading_director.py` | The `TradingDirector` class -- main event loop and event dispatch |
| `core.configurations.trading_session_configurations` | `core/configurations/trading_session_configurations.py` | Session config models: `BaseTradingSessionConfig`, `MT5BacktestSessionConfig`, `MT5LiveSessionConfig` |
| `services.hook_service` | `services/hook_service.py` | Empty placeholder file (actual hook service lives in `pyeventbt.hooks`) |

## Internal Architecture

```
TradingDirector
    |
    +-- events_queue (queue.Queue) -- shared event bus
    |
    +-- event_handlers_dict
    |       BAR    --> _handle_bar_event
    |       SIGNAL --> _handle_signal_event
    |       ORDER  --> _handle_order_event
    |       FILL   --> _handle_fill_event
    |
    +-- DATA_PROVIDER       -- feeds BarEvents when queue is empty
    +-- SIGNAL_GENERATOR    -- SignalEngineService
    +-- EXECUTION_ENGINE    -- processes OrderEvents
    +-- PORTFOLIO_HANDLER   -- processes Bar/Signal/Fill events
    +-- SCHEDULE_SERVICE    -- time-based callback scheduling
    +-- HOOK_SERVICE        -- lifecycle hooks (ON_START, ON_END, etc.)
```

### Backtest vs Live

- **Backtest** (`_run_backtest`): Loops while `DATA_PROVIDER.continue_backtest` is `True`. On queue empty, calls `update_bars()`. At end-of-data, closes all open positions then processes backtest results.
- **Live** (`_run_live_trading`): Infinite loop with `time.sleep(heartbeat)` between iterations. On queue empty, calls `update_bars()`.

### Subpackages

- `core/configurations/` -- Pydantic configuration models for session setup
- `services/` -- Contains an empty `hook_service.py` (legacy/placeholder; the real hook service is in `pyeventbt.hooks`)

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events` | All event types (`BarEvent`, `SignalEvent`, `OrderEvent`, `FillEvent`, `EventType`) |
| `pyeventbt.signal_engine.services.signal_engine_service` | `SignalEngineService` for signal generation |
| `pyeventbt.portfolio_handler.portfolio_handler` | `PortfolioHandler` for processing bar/signal/fill events |
| `pyeventbt.hooks.hook_service` | `HookService` and `Hooks` enum |
| `pyeventbt.schedule_service.schedule_service` | `ScheduleService` for scheduled callbacks |
| `pyeventbt.strategy.core.modules` | `Modules` (dependency injection container) |
| `pyeventbt.strategy.core.strategy_timeframes` | `StrategyTimeframes` enum |
| `pyeventbt.broker.mt5_broker.connectors.live_mt5_broker` | `LiveMT5Broker` for live session setup |
| `pyeventbt.trading_context.trading_context` | Trading context (backtest vs live) global state |
| `pyeventbt.utils.utils` | `Utils` utility class |
| `pyeventbt.config.configs` | `Mt5PlatformConfig` (via session configurations) |

## Gaps & Issues

1. **`services/hook_service.py` is empty**: The real `HookService` lives in `pyeventbt.hooks.hook_service`. This empty file is either a leftover or a planned local override that was never implemented.
2. **`SCHEDULED_EVENT` not in `event_handlers_dict`**: `ScheduledEvent` has a defined `EventType` but the director has no handler mapping for it. If a `ScheduledEvent` were placed on the queue, it would raise a `KeyError`.
3. **No graceful shutdown for live trading**: `_run_live_trading` runs `while True` with no break condition or signal handling.
4. **`_process_order_event` is a private method called externally**: `TradingDirector._handle_order_event` calls `self.EXECUTION_ENGINE._process_order_event(event)`, accessing a private method on another object.
5. **Default mutable `HookService()` in constructor**: The `hook_service` parameter defaults to `HookService()`, a mutable default that is shared across calls if not overridden.
