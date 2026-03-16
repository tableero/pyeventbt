# File: `pyeventbt/trading_director/trading_director.py`

## Module

`pyeventbt.trading_director.trading_director`

## Purpose

Implements the `TradingDirector`, the central event loop of the PyEventBT framework. It dequeues events from a shared `queue.Queue`, dispatches each event to its registered handler, and manages the lifecycle of both backtesting and live trading sessions.

## Tags

`event-loop`, `core`, `orchestration`, `backtest`, `live-trading`, `dispatcher`

## Dependencies

| Dependency | Import |
|---|---|
| `pyeventbt.signal_engine.services.signal_engine_service` | `SignalEngineService` |
| `pyeventbt.hooks.hook_service` | `HookService`, `Hooks` |
| `pyeventbt.schedule_service.schedule_service` | `ScheduleService` |
| `pyeventbt.strategy.core.modules` | `Modules` |
| `pyeventbt.events.events` | `BarEvent`, `ScheduledEvent`, `SignalEvent`, `OrderEvent`, `FillEvent`, `EventType` |
| `pyeventbt.portfolio_handler.portfolio_handler` | `PortfolioHandler` |
| `pyeventbt.broker.mt5_broker.connectors.live_mt5_broker` | `LiveMT5Broker` |
| `pyeventbt.strategy.core.strategy_timeframes` | `StrategyTimeframes` |
| `.core.configurations.trading_session_configurations` | `BaseTradingSessionConfig`, `MT5BacktestSessionConfig`, `MT5LiveSessionConfig` |
| `pyeventbt.utils.utils` | `Utils` |
| `pyeventbt.trading_context.trading_context` | Module-level import (aliased as `trading_context`) |
| `queue` | Standard library queue |
| `time` | `time.sleep` for live heartbeat |
| `logging` | Logger named `"pyeventbt"` |

## Classes/Functions

### `TradingDirector`

```python
class TradingDirector:
    def __init__(
        self,
        events_queue: queue.Queue,
        signal_engine_service: SignalEngineService,
        portfolio_handler: PortfolioHandler,
        trading_session_config: BaseTradingSessionConfig,
        modules: Modules,
        run_schedules: bool = False,
        export_backtest: bool = False,
        export_backtest_parquet: bool = False,
        backtest_results_dir: str = None,
        hook_service: HookService = HookService()
    ) -> None
```

**Description**: Orchestrates the entire event-driven trading loop. On construction, it builds the event handler dispatch table, creates a `ScheduleService`, and configures the session type (backtest or live) based on the provided configuration object.

**Constructor Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `events_queue` | `queue.Queue` | Shared event bus |
| `event_handlers_dict` | `dict[EventType, Callable]` | Maps `EventType` to handler methods |
| `is_live_trading` | `bool` | `False` by default; set to `True` for live sessions |
| `MODULES` | `Modules` | Dependency injection container |
| `DATA_PROVIDER` | `DataProvider` | From `modules.DATA_PROVIDER` |
| `SIGNAL_GENERATOR` | `SignalEngineService` | User signal engine wrapper |
| `EXECUTION_ENGINE` | `ExecutionEngine` | From `modules.EXECUTION_ENGINE` |
| `PORTFOLIO_HANDLER` | `PortfolioHandler` | Processes bar/signal/fill events |
| `SCHEDULE_SERVICE` | `ScheduleService` | Manages time-based callbacks |
| `HOOK_SERVICE` | `HookService` | Lifecycle hook manager |

**Instance attributes set during session configuration**:

| Attribute | Set By | Type | Description |
|---|---|---|---|
| `initial_capital` | `_configure_mt5_backtest_session` | `float` | Starting capital for backtest |
| `start_date` | `_configure_mt5_backtest_session` | `datetime` | Backtest start date |
| `backtest_name` | `_configure_mt5_backtest_session` | `str` | Name identifier for the backtest |
| `heartbeat` | `_configure_mt5_live_session` | `float` | Sleep interval (seconds) between live loop iterations |
| `LIVE_MT5_BROKER` | `_configure_mt5_live_session` | `LiveMT5Broker` | Live MT5 connection |

---

#### `_configure_session(trading_session_config: BaseTradingSessionConfig) -> None`

Dispatches to `_configure_mt5_backtest_session` or `_configure_mt5_live_session` based on config type using `isinstance` checks.

---

#### `_configure_mt5_backtest_session(configuration: MT5BacktestSessionConfig) -> None`

Sets `initial_capital`, `start_date`, and `backtest_name` from the configuration.

---

#### `_configure_mt5_live_session(configuration: MT5LiveSessionConfig) -> None`

Sets `heartbeat`, `is_live_trading = True`, and creates a `LiveMT5Broker` instance.

---

#### `_handle_bar_event(event: BarEvent) -> None`

1. Calls `PORTFOLIO_HANDLER.process_bar_event(event)` to update portfolio values
2. Calls `SCHEDULE_SERVICE.run_scheduled_callbacks(event)` to fire time-based callbacks
3. Calls `SIGNAL_GENERATOR.generate_signal(event)` to potentially emit a `SignalEvent`

---

#### `_handle_signal_event(event: SignalEvent) -> None`

1. Calls `HOOK_SERVICE.call_callbacks(Hooks.ON_SIGNAL_EVENT, MODULES)`
2. Calls `PORTFOLIO_HANDLER.process_signal_event(event)` which runs sizing and risk engines, potentially emitting an `OrderEvent`

---

#### `_handle_order_event(event: OrderEvent) -> None`

1. Calls `EXECUTION_ENGINE._process_order_event(event)` to execute the trade, emitting a `FillEvent`
2. Calls `HOOK_SERVICE.call_callbacks(Hooks.ON_ORDER_EVENT, MODULES)`

---

#### `_handle_fill_event(event: FillEvent) -> None`

Calls `PORTFOLIO_HANDLER.process_fill_event(event)` to update portfolio state (positions, balance).

---

#### `_handle_none_event(event) -> None`

Logs a warning for `None` events received from the queue.

---

#### `_handle_backtest_end() -> BacktestResults`

Logs backtest completion and delegates to `PORTFOLIO_HANDLER.process_backtest_end(backtest_name, export_csv, export_parquet)`. Returns the result.

---

#### `add_schedule(timeframe: StrategyTimeframes, fn: Callable[[ScheduledEvent, Modules], None]) -> None`

Registers a scheduled callback via `SCHEDULE_SERVICE.add_schedule`.

---

#### `run() -> None | BacktestResults`

**Description**: Main entry point. Calls `ON_START` hook, dispatches to `_run_backtest()` or `_run_live_trading()`, then calls `ON_END` hook. Returns the result (backtest results or `None`).

---

#### `_run_backtest() -> BacktestResults`

**Description**: Backtest event loop.

**Algorithm**:
1. If `run_schedules` is `False`, deactivates all schedules
2. While `DATA_PROVIDER.continue_backtest` is `True`:
   - Try to dequeue an event (non-blocking)
   - On `queue.Empty`, call `DATA_PROVIDER.update_bars()` to feed the next bar
   - On valid event, dispatch via `event_handlers_dict[event.type](event)`
   - On `None` event, call `_handle_none_event`
   - If `close_positions_end_of_data` is set and there are open positions, close them and continue loop to process resulting fill events
   - If no open positions and queue is empty, set `continue_backtest = False`
3. Call `_handle_backtest_end()` and return result

---

#### `_run_live_trading() -> None`

**Description**: Live trading event loop. Detects new closed candles by **polling** MT5 at a configurable interval — the system does not receive push notifications when a bar closes.

**Algorithm**:
1. If `run_schedules` is `False`, deactivates all schedules
2. Infinite loop:
   - Try to dequeue an event (non-blocking)
   - On `queue.Empty`, call `DATA_PROVIDER.update_bars()` — this polls MT5 for the last closed bar (`from_pos=1`) per symbol/timeframe and emits `BarEvent`s only when a new bar datetime is detected (see `Mt5LiveDataProvider.update_bars` for details)
   - On valid event, dispatch via `event_handlers_dict[event.type](event)`
   - Sleep for `heartbeat` seconds — this controls the polling frequency and determines the worst-case latency between a candle closing and the system reacting to it

## Data Flow

```
run()
  |
  +--> ON_START hook
  |
  +--> _run_backtest() / _run_live_trading()
  |       |
  |       +--> [queue empty] --> DATA_PROVIDER.update_bars() --> BarEvent into queue
  |       |
  |       +--> [BAR event]
  |       |       +--> PortfolioHandler.process_bar_event
  |       |       +--> ScheduleService.run_scheduled_callbacks
  |       |       +--> SignalEngineService.generate_signal --> SignalEvent into queue
  |       |
  |       +--> [SIGNAL event]
  |       |       +--> ON_SIGNAL_EVENT hook
  |       |       +--> PortfolioHandler.process_signal_event --> OrderEvent into queue
  |       |
  |       +--> [ORDER event]
  |       |       +--> ExecutionEngine._process_order_event --> FillEvent into queue
  |       |       +--> ON_ORDER_EVENT hook
  |       |
  |       +--> [FILL event]
  |               +--> PortfolioHandler.process_fill_event
  |
  +--> ON_END hook
```

## Gaps & Issues

1. **Mutable default argument**: `hook_service: HookService = HookService()` in the constructor creates a single shared instance across all calls that omit this parameter. This is a known Python anti-pattern.
2. **Private method cross-call**: `_handle_order_event` calls `self.EXECUTION_ENGINE._process_order_event(event)`, accessing a private method on another object, violating encapsulation.
3. **No `SCHEDULED_EVENT` handler**: The `event_handlers_dict` does not map `EventType.SCHEDULED_EVENT`. If such an event were queued, a `KeyError` would be raised.
4. **No graceful shutdown for live trading**: `_run_live_trading` uses `while True` with no break condition, signal handler, or exception handling for clean shutdown.
5. **`_run_backtest` returns a value but `_run_live_trading` does not**: The `run()` method returns `res` from both paths, but the live path returns `None` implicitly.
6. **Catalan comment on line 77**: `# interessant posar una flag que digui si el environment global es backtest o live` is a developer note left in the source.

## Requirements Derived

| ID | Requirement | Source |
|---|---|---|
| TD-01 | The event loop must dispatch events based on `EventType` to registered handler methods | `event_handlers_dict` pattern |
| TD-02 | When the event queue is empty, the data provider must be polled for new bars | `queue.Empty` handling in both loops |
| TD-03 | At the end of a backtest, all open positions must be closed before finalizing results | `close_positions_end_of_data` logic |
| TD-04 | Live trading must sleep for a configurable heartbeat between iterations | `time.sleep(self.heartbeat)` |
| TD-05 | Lifecycle hooks (ON_START, ON_END, ON_SIGNAL_EVENT, ON_ORDER_EVENT) must fire at appropriate points | Hook calls in `run()` and handlers |
| TD-06 | Scheduled callbacks must be evaluated on every bar event | `_handle_bar_event` calls `run_scheduled_callbacks` |
| TD-07 | Schedules can be disabled globally via `run_schedules=False` | `deactivate_schedules()` call at loop start |
