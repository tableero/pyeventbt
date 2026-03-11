# pyeventbt.portfolio_handler.core.entities.suggested_order

## File
`pyeventbt/portfolio_handler/core/entities/suggested_order.py`

## Module
`pyeventbt.portfolio_handler.core.entities.suggested_order`

## Purpose
Defines the `SuggestedOrder` Pydantic model representing a sized order produced by the sizing engine, ready for risk assessment. Acts as the intermediate data object between the sizing engine and the risk engine in the order pipeline.

## Tags
`entity`, `pydantic`, `order`, `sizing`, `domain-model`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class |
| `decimal.Decimal` | Precision for volume |
| `typing.Optional` | Optional `buffer_data` field |
| `pyeventbt.events.events.SignalEvent` | Embedded signal event with full signal context |

## Classes/Functions

### `SuggestedOrder(BaseModel)`

**Signature:** Pydantic model (no custom `__init__`)

**Description:** Intermediate data object in the order pipeline. Created by the sizing engine from a `SignalEvent`, it carries the computed volume along with the original signal event. The optional `buffer_data` dict allows passing arbitrary additional data between engines. Consumed by the risk engine's `assess_order()` method.

**Attributes:**

| Attribute | Type | Required | Description |
|---|---|---|---|
| `signal_event` | `SignalEvent` | Yes | The original signal event containing symbol, direction, order type, price, sl, tp, strategy_id, and other signal metadata |
| `volume` | `Decimal` | Yes | Computed position size from the sizing engine |
| `buffer_data` | `Optional[dict]` | No | Arbitrary additional data passed between engines (default `None`) |

**Returns:** N/A (data model)

## Data Flow

```
SignalEvent
  -> SizingEngine.get_suggested_order(signal_event)
      -> Creates SuggestedOrder(signal_event=signal_event, volume=computed_volume)
  -> RiskEngine.assess_order(suggested_order)
      -> Validates order against risk rules
      -> [if approved] Creates OrderEvent and puts it on the event queue
```

## Gaps & Issues

1. The model is very lean compared to the knowledge brief description, which mentioned fields like `symbol`, `strategy_id`, `signal_type`, `order_type`, `order_price`, `sl`, `tp`, and `rollover`. In practice, these fields are accessed via the embedded `signal_event` object rather than being top-level attributes.
2. `buffer_data` is typed as `Optional[dict]` without specifying key/value types (`dict[str, Any]`), reducing type safety.
3. No validation on `volume` (e.g., must be positive, must meet minimum lot size).

## Requirements Derived

- **RQ-SO-001**: A suggested order must carry the original signal event and a computed volume.
- **RQ-SO-002**: A suggested order must support optional arbitrary buffer data for inter-engine communication.
- **RQ-SO-003**: The suggested order is the sole input to the risk engine's `assess_order()` method.
