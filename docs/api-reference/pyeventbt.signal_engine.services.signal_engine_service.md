# pyeventbt.signal_engine.services.signal_engine_service

- **File**: `pyeventbt/signal_engine/services/signal_engine_service.py`
- **Module**: `pyeventbt.signal_engine.services.signal_engine_service`
- **Purpose**: Orchestrates signal generation by dispatching `BarEvent`s to the active signal engine and placing resulting `SignalEvent`s onto the shared events queue. Acts as the bridge between the `TradingDirector` event loop and concrete signal engine implementations.
- **Tags**: `service`, `signal-engine`, `event-queue`, `orchestration`

## Dependencies

| Dependency | Type |
|---|---|
| `pyeventbt.strategy.core.modules.Modules` | Internal |
| `pyeventbt.signal_engine.core.interfaces.signal_engine_interface.ISignalEngine` | Internal |
| `pyeventbt.signal_engine.signal_engines.signal_passthrough.SignalPassthrough` | Internal |
| `pyeventbt.signal_engine.signal_engines.signal_ma_crossover.SignalMACrossover` | Internal |
| `pyeventbt.signal_engine.core.configurations.signal_engine_configurations.BaseSignalEngineConfig` | Internal |
| `pyeventbt.signal_engine.core.configurations.signal_engine_configurations.MACrossoverConfig` | Internal |
| `pyeventbt.events.events.BarEvent` | Internal |
| `queue.Queue` | Standard library |
| `logging` | Standard library |

## Classes/Functions

### `SignalEngineService`

- **Signature**: `class SignalEngineService`
- **Description**: Service layer that owns a signal engine instance, receives bar events from the trading director, and enqueues any generated signal events. Supports both predefined configs and user-supplied custom engine functions.

#### `__init__`

- **Signature**: `def __init__(self, events_queue: Queue, modules: Modules, signal_config: BaseSignalEngineConfig = None) -> None`
- **Description**: Initialises the service with the shared event queue, modules context, and an optional signal configuration. Calls `_get_signal_engine` to resolve the concrete engine.
- **Attributes set**:
  - `self.events_queue: Queue` -- Shared event queue for inter-component communication.
  - `self.modules: Modules` -- Provides access to `DATA_PROVIDER`, `PORTFOLIO`, `EXECUTION_ENGINE`, `TRADING_CONTEXT`.
  - `self.signal_engine: ISignalEngine` -- The resolved signal engine instance.

#### `_get_signal_engine`

- **Signature**: `def _get_signal_engine(self, signal_config: BaseSignalEngineConfig) -> ISignalEngine`
- **Description**: Factory method. Returns `SignalMACrossover` if `signal_config` is `MACrossoverConfig`, otherwise falls back to `SignalPassthrough`.
- **Returns**: `ISignalEngine`

#### `set_signal_engine`

- **Signature**: `def set_signal_engine(self, new_signal_engine) -> None`
- **Description**: Replaces the instance's `generate_signal` method with a closure that calls `new_signal_engine(bar_event, self.modules)` and enqueues resulting `SignalEvent`(s). Used by `@strategy.custom_signal_engine` decorator wiring.
- **Returns**: `None`

#### `generate_signal`

- **Signature**: `def generate_signal(self, bar_event: BarEvent) -> None`
- **Description**: Invokes the current signal engine's `generate_signal` method. If the result is a single `SignalEvent`, it is placed on the queue. If it is a list, each element is enqueued individually. `None` results are silently ignored.
- **Returns**: `None`

## Data Flow

```
TradingDirector (BAR event)
  |
  v
SignalEngineService.generate_signal(bar_event)
  |
  v
self.signal_engine.generate_signal(bar_event, modules)
  |                                           |
  |  (predefined engine, e.g. SignalMACrossover)
  |  -- OR --
  |  (custom engine set via set_signal_engine)
  |
  v
SignalEvent | list[SignalEvent] | None
  |
  v  (if not None)
events_queue.put(signal_event)
  |
  v
TradingDirector picks up SIGNAL event --> PortfolioHandler
```

## Gaps & Issues

1. `generate_signal` does not catch exceptions from the underlying engine. A failing engine will propagate the exception up to `TradingDirector` and halt the event loop.
2. `set_signal_engine` replaces `self.generate_signal` with a closure, leaving `self.signal_engine` stale. This means inspecting `self.signal_engine` after a custom engine is set will show the old (passthrough/predefined) engine, which is misleading.
3. The duplicate enqueue logic (check for list, put items) exists in both `generate_signal` and the closure created by `set_signal_engine`. This violates DRY.
4. `_get_signal_engine` contains a typo in its debug log message: `"SINAL"` instead of `"SIGNAL"`.
5. The `new_signal_engine` parameter of `set_signal_engine` lacks a type annotation.

## Requirements Derived

- REQ-SIGSVC-01: The service must support both predefined signal engine configs and user-supplied custom engine callables.
- REQ-SIGSVC-02: Generated `SignalEvent`(s) must be placed onto the shared events queue for downstream consumption.
- REQ-SIGSVC-03: The service must handle single `SignalEvent` returns and `list[SignalEvent]` returns uniformly.
- REQ-SIGSVC-04: `None` return values from signal engines must be silently discarded without enqueuing.
