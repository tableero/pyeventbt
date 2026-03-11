# pyeventbt.risk_engine.services.risk_engine_service

## File
`pyeventbt/risk_engine/services/risk_engine_service.py`

## Module
`pyeventbt.risk_engine.services.risk_engine_service`

## Purpose
Orchestrates risk evaluation for suggested orders. Delegates to a concrete `IRiskEngine` implementation, and if the order is approved (volume > 0), creates an `OrderEvent` and enqueues it. Also supports runtime injection of custom risk logic via `set_custom_asses_order`.

## Tags
`service`, `risk-management`, `event-emitter`, `order-pipeline`

## Dependencies

| Dependency | Usage |
|---|---|
| `typing.Callable` | Type hint for custom risk function |
| `queue.Queue` | Shared event queue for emitting `OrderEvent`s |
| `decimal.Decimal` | Volume type in `OrderEvent` construction |
| `pyeventbt.strategy.core.modules.Modules` | Runtime context passed to concrete engines |
| `pyeventbt.risk_engine.core.interfaces.risk_engine_interface.IRiskEngine` | Interface for concrete engines |
| `pyeventbt.risk_engine.risk_engines.passthrough_risk_engine.PassthroughRiskEngine` | Default engine |
| `pyeventbt.risk_engine.core.configurations.risk_engine_configurations.BaseRiskConfig` | Config base |
| `pyeventbt.risk_engine.core.configurations.risk_engine_configurations.PassthroughRiskConfig` | Config for passthrough selection |
| `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder` | Input entity |
| `pyeventbt.events.events.OrderEvent` | Output event |

## Classes/Functions

### `RiskEngineService`

| Field | Value |
|---|---|
| **Signature** | `class RiskEngineService(IRiskEngine)` |
| **Description** | Service-layer orchestrator for risk evaluation. Wraps a concrete `IRiskEngine`, delegates `assess_order` calls, and handles `OrderEvent` creation and queue placement. |
| **Attributes** | `events_queue: Queue` -- shared event queue; `modules: Modules` -- runtime context; `risk_engine: IRiskEngine` -- the selected concrete engine |

#### `__init__`

| Field | Value |
|---|---|
| **Signature** | `def __init__(self, events_queue: Queue, risk_config: BaseRiskConfig, modules: Modules) -> None` |
| **Description** | Initializes the service, selecting the concrete risk engine based on the provided config. |
| **Parameters** | `events_queue` -- shared event queue; `risk_config` -- determines which engine to instantiate; `modules` -- runtime context |

#### `_get_risk_management_method`

| Field | Value |
|---|---|
| **Signature** | `def _get_risk_management_method(self, risk_config: BaseRiskConfig) -> IRiskEngine` |
| **Description** | Factory method. Returns `PassthroughRiskEngine()` for `PassthroughRiskConfig`. Falls back to `PassthroughRiskEngine()` for any unrecognized config. |
| **Returns** | `IRiskEngine` -- the concrete engine instance |

#### `_create_and_put_order_event`

| Field | Value |
|---|---|
| **Signature** | `def _create_and_put_order_event(self, suggested_order: SuggestedOrder, new_volume: Decimal) -> None` |
| **Description** | Constructs an `OrderEvent` from the suggested order with the approved volume and places it on the events queue. Copies symbol, time, strategy_id, signal_type, order_type, order_price, sl, tp, rollover, and buffer_data from the underlying `SignalEvent`. |

#### `assess_order`

| Field | Value |
|---|---|
| **Signature** | `def assess_order(self, suggested_order: SuggestedOrder) -> None` |
| **Description** | Main entry point called by `PortfolioHandler`. Delegates to the concrete engine's `assess_order`, then creates and enqueues an `OrderEvent` if the returned volume > 0. Note: the signature differs from `IRiskEngine.assess_order` (no `modules` param, returns `None` not `float`). |

#### `set_custom_asses_order`

| Field | Value |
|---|---|
| **Signature** | `def set_custom_asses_order(self, custom_asses_order: Callable[[SuggestedOrder], float])` |
| **Description** | Replaces `self.assess_order` with a closure that calls the user-supplied function. The custom function receives `(suggested_order, modules)` and must return a float volume. If volume > 0, the closure creates and enqueues the `OrderEvent`. |
| **Parameters** | `custom_asses_order` -- user-defined callable `(SuggestedOrder, Modules) -> float` |

## Data Flow

```
PortfolioHandler
    --> RiskEngineService.assess_order(suggested_order)
        --> self.risk_engine.assess_order(suggested_order, self.modules) --> volume: float
        --> if volume > 0:
            --> _create_and_put_order_event(suggested_order, volume)
                --> OrderEvent constructed from SignalEvent fields + approved volume
                --> events_queue.put(order_event)
```

## Gaps & Issues

1. **Typo in method name.** `set_custom_asses_order` should be `set_custom_assess_order`. This affects the public API surface.
2. **Signature mismatch.** `RiskEngineService.assess_order` takes `(self, suggested_order)` while `IRiskEngine.assess_order` takes `(self, suggested_order, modules)`. The service is not a true drop-in for the interface.
3. **Type annotation mismatch.** The `custom_asses_order` parameter is typed as `Callable[[SuggestedOrder], float]` but is actually called with `(suggested_order, modules)` -- two arguments.
4. **Else-branch fallback.** `_get_risk_management_method` silently falls back to `PassthroughRiskEngine` for unknown configs instead of raising an error.
5. **Volume comparison.** `if new_volume > 0.0` compares a float to a float literal, but `_create_and_put_order_event` accepts `Decimal`. The Decimal is constructed later from `SignalEvent` fields; the volume itself may be a float from the risk engine.

## Requirements Derived

- R-RISK-SVC-01: `RiskEngineService` shall delegate order evaluation to the configured concrete `IRiskEngine`.
- R-RISK-SVC-02: Orders with approved volume <= 0 shall be silently discarded (no `OrderEvent` emitted).
- R-RISK-SVC-03: The service shall support runtime replacement of risk logic via a callable injection method.
- R-RISK-SVC-04: Created `OrderEvent`s shall faithfully copy all fields from the originating `SignalEvent`, replacing only the volume.
