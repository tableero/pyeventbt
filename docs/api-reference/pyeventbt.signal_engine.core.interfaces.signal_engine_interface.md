# pyeventbt.signal_engine.core.interfaces.signal_engine_interface

- **File**: `pyeventbt/signal_engine/core/interfaces/signal_engine_interface.py`
- **Module**: `pyeventbt.signal_engine.core.interfaces.signal_engine_interface`
- **Purpose**: Defines the `ISignalEngine` Protocol that all signal engine implementations must satisfy, and provides `SignalEngineGenerator`, a factory that wraps arbitrary callables into an `ISignalEngine`-compatible object.
- **Tags**: `interface`, `protocol`, `signal-engine`, `factory`

## Dependencies

| Dependency | Type |
|---|---|
| `pyeventbt.strategy.core.modules.Modules` | Internal |
| `pyeventbt.portfolio.portfolio.Portfolio` | Internal |
| `pyeventbt.events.events.BarEvent` | Internal |
| `pyeventbt.events.events.SignalEvent` | Internal |
| `typing.Protocol` | Standard library |
| `typing.Callable` | Standard library |

## Classes/Functions

### `ISignalEngine(Protocol)`

- **Signature**: `class ISignalEngine(Protocol)`
- **Description**: Protocol defining the interface for all signal engine implementations. Designed to be data-source agnostic -- works with both historic and live data since it receives events from the shared queue.
- **Attributes**: None
- **Methods**:

#### `ISignalEngine.generate_signal`

- **Signature**: `def generate_signal(self, bar_event: BarEvent, modules: Modules) -> SignalEvent | list[SignalEvent]`
- **Description**: Calculates and returns trading signal(s) based on the incoming bar event and available modules (data provider, portfolio, execution engine).
- **Returns**: `SignalEvent | list[SignalEvent]` -- A single signal event, a list of signal events, or implicitly `None` when no signal is generated.

---

### `SignalEngineGenerator(ISignalEngine)`

- **Signature**: `class SignalEngineGenerator(ISignalEngine)`
- **Description**: Factory/adapter class that wraps an arbitrary callable into an `ISignalEngine`-compatible object. Used internally to convert user-decorated functions into signal engines.
- **Attributes**:
  - `signal_generator: Callable` -- The wrapped callable; defaults to `lambda x: None`.
- **Methods**:

#### `SignalEngineGenerator.__init__`

- **Signature**: `def __init__(self) -> None`
- **Description**: Initialises the generator with a no-op lambda as the default signal generator.

#### `SignalEngineGenerator.generate_signal`

- **Signature**: `def generate_signal(self, bar_event: BarEvent) -> SignalEvent | list[SignalEvent]`
- **Description**: Delegates to `self.signal_generator(bar_event)`.
- **Returns**: `SignalEvent | list[SignalEvent]`

#### `SignalEngineGenerator.generate_signal_engine` (staticmethod)

- **Signature**: `@staticmethod def generate_signal_engine(signal_generator: Callable[[BarEvent, Portfolio], SignalEvent] | list[SignalEvent]) -> SignalEngineGenerator`
- **Description**: Factory method. Creates a `SignalEngineGenerator` instance and assigns the provided callable as its `signal_generator`.
- **Returns**: `SignalEngineGenerator`

## Data Flow

```
User-defined callable (decorated with @strategy.custom_signal_engine)
  |
  v
SignalEngineGenerator.generate_signal_engine(fn)
  |
  v
SignalEngineGenerator instance (implements ISignalEngine)
  |
  v
SignalEngineService.generate_signal(bar_event)  -->  generator.generate_signal(bar_event)
```

## Gaps & Issues

1. `SignalEngineGenerator.generate_signal` accepts only `bar_event` (1 argument) while `ISignalEngine.generate_signal` expects `(bar_event, modules)` (2 arguments). This signature mismatch means `SignalEngineGenerator` does not actually satisfy the `ISignalEngine` protocol at runtime.
2. The `generate_signal_engine` type hint `Callable[[BarEvent, Portfolio], SignalEvent] | list[SignalEvent]` is malformed -- the `| list[SignalEvent]` is outside the `Callable` and reads as the parameter itself being a list, not the return type.
3. `SignalEngineGenerator` appears unused in practice -- `SignalEngineService.set_signal_engine` replaces the method directly via a closure rather than using this factory.
4. The docstring describes `ISignalEngine` as an "abstract base class" but it is actually a `Protocol`, not an ABC.

## Requirements Derived

- REQ-SIGIF-01: All signal engine implementations must expose a `generate_signal(bar_event, modules)` method returning `SignalEvent | list[SignalEvent] | None`.
- REQ-SIGIF-02: The framework must support wrapping arbitrary user callables into signal engine objects.
