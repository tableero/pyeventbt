# pyeventbt.signal_engine

- **Package**: `pyeventbt.signal_engine`
- **Purpose**: Provides the signal generation layer of the PyEventBT framework. Encapsulates configuration, interfaces, service orchestration, and concrete signal engine implementations (MA crossover, passthrough). The package is responsible for consuming `BarEvent`s and producing `SignalEvent`s that feed into the sizing/risk/execution pipeline.
- **Tags**: `signal-generation`, `event-driven`, `trading-signals`, `moving-average`, `crossover`

## Modules

| Module | Path | Role |
|---|---|---|
| `__init__.py` | `signal_engine/__init__.py` | Re-exports `signal_engine_configurations` (all public names) and `SignalMACrossover` |
| `signal_engine_configurations` | `core/configurations/signal_engine_configurations.py` | Pydantic config models: `BaseSignalEngineConfig`, `MAType`, `MACrossoverConfig` |
| `signal_engine_interface` | `core/interfaces/signal_engine_interface.py` | `ISignalEngine` Protocol and `SignalEngineGenerator` factory |
| `signal_engine_service` | `services/signal_engine_service.py` | `SignalEngineService` -- dispatches bar events to the active signal engine and enqueues resulting signals |
| `signal_ma_crossover` | `signal_engines/signal_ma_crossover.py` | `SignalMACrossover` -- concrete MA crossover signal engine |
| `signal_passthrough` | `signal_engines/signal_passthrough.py` | `SignalPassthrough` -- no-op placeholder engine |

## Internal Architecture

1. `SignalEngineService` is instantiated by the strategy wiring layer with an `events_queue`, `Modules`, and an optional `BaseSignalEngineConfig`.
2. If a `MACrossoverConfig` is supplied, the service creates a `SignalMACrossover` internally. Otherwise it falls back to `SignalPassthrough`.
3. Users can override the engine entirely via `set_signal_engine(fn)`, which replaces `generate_signal` with a closure that calls the user-supplied callable.
4. On each `BAR` event dispatched by `TradingDirector`, `generate_signal(bar_event)` is invoked. The engine returns `SignalEvent | list[SignalEvent] | None`; non-None results are placed onto the shared `events_queue`.

## Cross-Package Dependencies

- `pyeventbt.events.events` -- `BarEvent`, `SignalEvent`
- `pyeventbt.strategy.core.modules` -- `Modules` (provides `DATA_PROVIDER`, `PORTFOLIO`, `EXECUTION_ENGINE`, `TRADING_CONTEXT`)
- `pyeventbt.core.entities.hyper_parameter` -- `HyperParameter`
- `pyeventbt.data_provider.core.interfaces.data_provider_interface` -- `IDataProvider` (imported in `signal_ma_crossover`)
- `pyeventbt.portfolio.portfolio` -- `Portfolio`
- `pyeventbt.execution_engine.core.interfaces.execution_engine_interface` -- `IExecutionEngine`
- `pyeventbt.trading_context.trading_context` -- `TypeContext`
- Third-party: `pydantic`, `pandas`, `queue.Queue`, `logging`

## Gaps & Issues

1. `SignalEngineService.generate_signal` does not wrap the engine call in a try/except; if a custom engine raises, the exception propagates uncaught and may halt the event loop.
2. `set_signal_engine` replaces the instance method `generate_signal` with a closure, which means the original `self.signal_engine` object is never updated -- the two code paths (predefined vs custom) diverge silently.
3. `_get_signal_engine` logs a debug message with a typo: `"SINAL"` instead of `"SIGNAL"`.
4. `SignalMACrossover` compares `self.trading_context` against the string `"BACKTEST"` rather than the enum `TypeContext.BACKTEST`, which may break if the enum representation changes.
5. No unit tests exist for any module in this package.
