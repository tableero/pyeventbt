# pyeventbt.signal_engine.signal_engines.signal_passthrough

- **File**: `pyeventbt/signal_engine/signal_engines/signal_passthrough.py`
- **Module**: `pyeventbt.signal_engine.signal_engines.signal_passthrough`
- **Purpose**: Provides a minimal no-op signal engine implementation used as the default placeholder when no predefined signal engine configuration is supplied. Always returns `None`, producing no signals.
- **Tags**: `signal-engine`, `placeholder`, `no-op`, `default`

## Dependencies

| Dependency | Type |
|---|---|
| `pyeventbt.strategy.core.modules.Modules` | Internal |
| `pyeventbt.events.events.BarEvent` | Internal |
| `pyeventbt.events.events.SignalEvent` | Internal |
| `pyeventbt.signal_engine.core.interfaces.signal_engine_interface.ISignalEngine` | Internal |

## Classes/Functions

### `SignalPassthrough(ISignalEngine)`

- **Signature**: `class SignalPassthrough(ISignalEngine)`
- **Description**: No-op signal engine. Implements the `ISignalEngine` protocol but generates no signals. Used as the default engine in `SignalEngineService` when the user provides a custom signal engine via `@strategy.custom_signal_engine` (the passthrough holds the slot until `set_signal_engine` replaces `generate_signal`).

#### `generate_signal`

- **Signature**: `def generate_signal(self, bar_event: BarEvent, modules: Modules) -> SignalEvent`
- **Description**: Does nothing. Returns `None` implicitly (via `pass`).
- **Returns**: `None` (implicit)

## Data Flow

```
BarEvent --> SignalPassthrough.generate_signal(bar_event, modules) --> None
```

No `SignalEvent` is produced. `SignalEngineService` checks for `None` and does not enqueue anything.

## Gaps & Issues

1. The return type annotation is `SignalEvent` but the method always returns `None`. The annotation should be `SignalEvent | None` or `Optional[SignalEvent]` for accuracy.
2. The class has no docstring explaining its role as a default placeholder.

## Requirements Derived

- REQ-SIGPT-01: The passthrough engine must satisfy the `ISignalEngine` protocol without producing any signals.
- REQ-SIGPT-02: The passthrough engine must be safe to call repeatedly with no side effects.
