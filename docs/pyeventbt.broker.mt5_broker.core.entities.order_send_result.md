# File: `pyeventbt/broker/mt5_broker/core/entities/order_send_result.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.order_send_result`

## Purpose
Defines the `OrderSendResult` Pydantic model representing the result returned by an MT5 `order_send()` call. Contains the return code, deal/order tickets, execution prices, and the original `TradeRequest` that triggered the order.

## Tags
`entity`, `pydantic`, `order`, `execution`, `mt5`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for price/volume fields |
| `TradeRequest` | Internal | Nested model for the original trade request |

## Classes/Functions

### `class OrderSendResult(BaseModel)`

Pydantic model representing the result of an `order_send` operation.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `retcode` | `int` | Return code (e.g., 10009 = TRADE_RETCODE_DONE) |
| `deal` | `int` | Deal ticket -- unique identifier of the executed deal |
| `order` | `int` | Order ticket -- unique identifier of the order (appears as Ticket in MT5 position view) |
| `volume` | `Decimal` | Executed volume |
| `price` | `Decimal` | Execution price |
| `bid` | `Decimal` | Current bid price at time of execution |
| `ask` | `Decimal` | Current ask price at time of execution |
| `comment` | `str` | Broker comment on the result |
| `request_id` | `int` | Request identifier |
| `retcode_external` | `int` | External system return code |
| `request` | `TradeRequest` | The original trade request that produced this result |

## Data Flow

```
ExecutionEngine creates TradeRequest
    |
    v
order_send(TradeRequest) -> OrderSendResult
    |
    v
OrderSendResult.retcode checked for success/failure
    |
    v
OrderSendResult.deal / .order used to track the position
```

## Gaps & Issues

1. **Nested `TradeRequest` coupling** -- `OrderSendResult` depends on `TradeRequest`, creating the only inter-entity dependency in the entities package. Changes to `TradeRequest` fields will break `OrderSendResult` construction.
2. **No default values** -- All fields are required, including `retcode_external` which is often 0.

## Requirements Derived

- **REQ-ENTITY-OSR-001**: `OrderSendResult` must include the original `TradeRequest` for auditability and debugging of trade execution.
- **REQ-ENTITY-OSR-002**: The `retcode` field must be checked against `TRADE_RETCODE_DONE` (10009) to determine execution success.
