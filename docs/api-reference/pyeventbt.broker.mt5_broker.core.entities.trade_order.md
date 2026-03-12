# File: `pyeventbt/broker/mt5_broker/core/entities/trade_order.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.trade_order`

## Purpose
Defines the `TradeOrder` Pydantic model representing a pending or historical order in MetaTrader 5. Contains full order details including setup/completion timestamps, order type, state, filling mode, and associated position information.

## Tags
`entity`, `pydantic`, `order`, `pending-order`, `mt5`, `history`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for price/volume fields |

## Classes/Functions

### `class TradeOrder(BaseModel)`

Pydantic model representing an MT5 order record.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `ticket` | `int` | Unique order ticket |
| `time_setup` | `int` | Order setup time (Unix timestamp) |
| `time_setup_msc` | `int` | Order setup time in milliseconds |
| `time_done` | `int` | Order execution/cancellation time |
| `time_done_msc` | `int` | Order execution/cancellation time in milliseconds |
| `time_expiration` | `int` | Order expiration time |
| `type` | `int` | Order type (0=BUY through 8=CLOSE_BY, see `ORDER_TYPE_*` constants) |
| `type_time` | `int` | Order time type (GTC, DAY, SPECIFIED, etc.) |
| `type_filling` | `int` | Order filling type (FOK, IOC, RETURN, BOC). See [MT5 docs](https://www.mql5.com/en/docs/constants/tradingconstants/orderproperties#enum_order_type_filling). |
| `state` | `int` | Order state (STARTED, PLACED, CANCELED, PARTIAL, FILLED, REJECTED, EXPIRED, etc.). See [MT5 docs](https://www.mql5.com/en/docs/constants/tradingconstants/orderproperties#enum_order_state). |
| `magic` | `int` | Expert Advisor / strategy magic number |
| `position_id` | `int` | Position identifier associated with this order |
| `position_by_id` | `int` | Opposite position identifier (for close-by orders) |
| `reason` | `int` | Order placement reason (client, mobile, web, expert, SL, TP, SO). See [MT5 docs](https://www.mql5.com/en/docs/constants/tradingconstants/orderproperties#enum_order_reason). |
| `volume_initial` | `Decimal` | Initial order volume |
| `volume_current` | `Decimal` | Remaining unfilled volume |
| `price_open` | `Decimal` | Order price |
| `sl` | `Decimal` | Stop Loss level |
| `tp` | `Decimal` | Take Profit level |
| `price_current` | `Decimal` | Current price of the order symbol |
| `price_stoplimit` | `Decimal` | Stop Limit order price |
| `symbol` | `str` | Trading symbol name |
| `comment` | `str` | Order comment |
| `external_id` | `str` | External system order identifier |

## Data Flow

```
TradeRequest submitted to broker
    |
    v
TradeOrder created (state=STARTED -> PLACED -> FILLED/CANCELED/EXPIRED)
    |
    v
Stored in SharedData.pending_orders (while active)
    |
    v
Moved to order history on completion
```

## Gaps & Issues

1. **`time_done_msc` extra field** -- The model includes `time_done_msc` which is present in the source but not always documented in older MT5 API versions.
2. **No state machine** -- Order state transitions (STARTED -> PLACED -> FILLED) are not enforced by the model.

## Requirements Derived

- **REQ-ENTITY-ORD-001**: `TradeOrder` must track both `volume_initial` and `volume_current` to support partial fills.
- **REQ-ENTITY-ORD-002**: The `state` field must accurately reflect the order lifecycle for history queries.
