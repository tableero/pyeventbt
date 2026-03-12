# pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface

- **File**: `pyeventbt/sizing_engine/core/interfaces/sizing_engine_interface.py`
- **Module**: `pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface`
- **Purpose**: Defines the `ISizingEngine` Protocol that all position sizing engines must satisfy. Establishes the contract for converting a `SignalEvent` into a `SuggestedOrder`.
- **Tags**: `interface`, `protocol`, `sizing`, `contract`

## Dependencies

| Dependency | Purpose |
|---|---|
| `typing.Protocol` | Structural subtyping base |
| `pyeventbt.strategy.core.modules.Modules` | Passed to engines for access to data provider, portfolio, etc. |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Return type |
| `pyeventbt.events.events.SignalEvent` | Input event type |

## Classes/Functions

### `ISizingEngine`

```python
class ISizingEngine(Protocol)
```

- **Description**: Protocol (structural interface) that all sizing engines must implement. Uses `typing.Protocol` so concrete classes do not need to explicitly inherit from it, though in practice they do.

#### `get_suggested_order`

```python
def get_suggested_order(self, signal_event: SignalEvent, modules: Modules) -> SuggestedOrder
```

- **Description**: Accepts a trading signal and produces a `SuggestedOrder` with a calculated position size (volume). Each concrete implementation defines its own sizing logic.
- **Parameters**:
  - `signal_event` (`SignalEvent`): The signal containing symbol, direction, order type, stop-loss, take-profit, and other trade parameters.
  - `modules` (`Modules`): Access to `DATA_PROVIDER`, `EXECUTION_ENGINE`, `PORTFOLIO`, `TRADING_CONTEXT`.
- **Returns**: `SuggestedOrder` -- wraps the original signal event plus a computed `volume`.
- **Raises**: `NotImplementedError` in the default Protocol body.

## Data Flow

```
SignalEvent + Modules --> ISizingEngine.get_suggested_order() --> SuggestedOrder
```

The `SizingEngineService` holds a reference to an `ISizingEngine` instance and delegates `get_suggested_order` calls through it. The resulting `SuggestedOrder` is then forwarded to the `RiskEngineService`.

## Gaps & Issues

1. **Concrete engines use `*args, **kwargs`**: `MT5MinSizing` and `MT5FixedSizing` accept `*args, **kwargs` instead of the explicit `modules: Modules` parameter, which weakens the Protocol guarantee at the call site.
2. **Protocol vs ABC**: `ISizingEngine` uses `Protocol` but raises `NotImplementedError` in its body. Since `Protocol` relies on structural subtyping, this fallback is only hit if someone instantiates the Protocol directly; it could be replaced with `...` for clarity.

## Requirements Derived

- R-SIZING-IF-01: Every sizing engine must implement `get_suggested_order(signal_event, modules) -> SuggestedOrder`.
- R-SIZING-IF-02: Engines must not modify the incoming `SignalEvent`; they should treat it as read-only input.
