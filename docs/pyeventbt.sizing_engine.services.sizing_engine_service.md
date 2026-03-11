# pyeventbt.sizing_engine.services.sizing_engine_service

- **File**: `pyeventbt/sizing_engine/services/sizing_engine_service.py`
- **Module**: `pyeventbt.sizing_engine.services.sizing_engine_service`
- **Purpose**: Factory and delegation service that selects the appropriate sizing engine based on configuration and exposes a unified `get_suggested_order` interface to the `PortfolioHandler`. Also supports runtime replacement of the sizing function for custom user-defined logic.
- **Tags**: `service`, `factory`, `sizing`, `delegation`, `portfolio-pipeline`

## Dependencies

| Dependency | Purpose |
|---|---|
| `typing.Callable` | Type hint for custom sizing functions |
| `queue.Queue` | Events queue (stored but not directly used by this service) |
| `pyeventbt.strategy.core.modules.Modules` | Provides `TRADING_CONTEXT` and other module references |
| `pyeventbt.sizing_engine.core.interfaces.sizing_engine_interface.ISizingEngine` | Protocol type for the sizing engine |
| `pyeventbt.sizing_engine.sizing_engines.mt5_min_sizing.MT5MinSizing` | Minimum-lot sizing implementation |
| `pyeventbt.sizing_engine.sizing_engines.mt5_fixed_sizing.MT5FixedSizing` | Fixed-volume sizing implementation |
| `pyeventbt.sizing_engine.sizing_engines.mt5_risk_pct_sizing.MT5RiskPctSizing` | Risk-percentage sizing implementation |
| `pyeventbt.sizing_engine.core.configurations.sizing_engine_configurations.*` | All config classes for dispatch |
| `pyeventbt.events.events.SignalEvent` | Input event type |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Output entity |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider` | Imported but not directly used in this file |

## Classes/Functions

### `SizingEngineService`

```python
class SizingEngineService
```

- **Description**: Manages the lifecycle and invocation of position sizing engines. Acts as both a factory (selecting engine from config) and a facade (delegating `get_suggested_order` calls).

#### `__init__`

```python
def __init__(self, events_queue: Queue, modules: Modules, sizing_config: BaseSizingConfig = BaseSizingConfig()) -> None
```

- **Description**: Initialises the service, stores the modules and events queue, and resolves the sizing engine via `_get_position_sizing_method`.
- **Parameters**:
  - `events_queue` (`Queue`): Shared event queue. Stored as `self.events_queue` but not consumed by this service.
  - `modules` (`Modules`): Trading modules providing `TRADING_CONTEXT` and runtime dependencies.
  - `sizing_config` (`BaseSizingConfig`): Configuration selecting the sizing strategy. Defaults to `BaseSizingConfig()` which resolves to `MT5MinSizing`.

#### `_get_position_sizing_method`

```python
def _get_position_sizing_method(self, sizing_config: BaseSizingConfig) -> ISizingEngine
```

- **Description**: Factory method. Matches the config type to a concrete engine instance.
- **Dispatch table**:
  - `MinSizingConfig` -> `MT5MinSizing(trading_context)`
  - `FixedSizingConfig` -> `MT5FixedSizing(configs)`
  - `RiskPctSizingConfig` -> `MT5RiskPctSizing(configs, trading_context)`
  - Anything else -> `MT5MinSizing(trading_context)` (default fallback)
- **Returns**: `ISizingEngine`

#### `get_suggested_order`

```python
def get_suggested_order(self, signal_event: SignalEvent) -> SuggestedOrder
```

- **Description**: Delegates to the selected engine's `get_suggested_order`, passing both the signal event and `self.modules`.
- **Parameters**:
  - `signal_event` (`SignalEvent`): The trading signal to size.
- **Returns**: `SuggestedOrder`

#### `set_suggested_order_function`

```python
def set_suggested_order_function(self, fn: Callable[[SignalEvent, Modules], SuggestedOrder]) -> None
```

- **Description**: Replaces the engine's `get_suggested_order` method with a user-provided function. Used when a strategy registers a custom sizing engine via `@strategy.custom_sizing_engine`.
- **Parameters**:
  - `fn` (`Callable[[SignalEvent, Modules], SuggestedOrder]`): Custom sizing function.
- **Returns**: None

## Data Flow

```
PortfolioHandler
  |
  v
SizingEngineService.get_suggested_order(signal_event)
  |
  v
ISizingEngine.get_suggested_order(signal_event, modules)
  |
  v
SuggestedOrder  -->  forwarded to RiskEngineService
```

## Gaps & Issues

1. **Duplicate import**: `SuggestedOrder` is imported twice from the same path.
2. **Unused `events_queue`**: The queue is stored on `self` but never read or written to within this service.
3. **Unused `IDataProvider` import**: Imported at the top of the file but not referenced.
4. **Monkey-patching risk**: `set_suggested_order_function` replaces a bound method on the engine instance. If the engine is later replaced (e.g., by re-initialisation), the custom function is lost.
5. **No validation of custom function signature**: The replacement function is accepted without any runtime check that it matches the expected `(SignalEvent, Modules) -> SuggestedOrder` signature.

## Requirements Derived

- R-SIZING-SVC-01: The service must default to minimum-lot sizing when no explicit config is provided.
- R-SIZING-SVC-02: Custom sizing functions set via `set_suggested_order_function` must accept `(SignalEvent, Modules)` and return `SuggestedOrder`.
- R-SIZING-SVC-03: The service must pass `modules` to the engine so that engines requiring runtime data (e.g., risk-pct) can access the data provider and portfolio.
