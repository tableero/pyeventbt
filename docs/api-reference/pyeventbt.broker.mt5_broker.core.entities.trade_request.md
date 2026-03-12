# File: `pyeventbt/broker/mt5_broker/core/entities/trade_request.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.trade_request`

## Purpose
Defines the `TradeRequest` Pydantic model representing the parameters for placing or modifying a trade order in MetaTrader 5. This is the input structure for `order_send()` calls and is embedded within `OrderSendResult` as the `request` field.

## Tags
`entity`, `pydantic`, `trade-request`, `order`, `mt5`, `execution`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for price/volume fields |

## Classes/Functions

### `class TradeRequest(BaseModel)`

Pydantic model representing MT5 trade request parameters.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `action` | `int` | Trade action type (1=DEAL, 5=PENDING, 6=SLTP, 7=MODIFY, 8=REMOVE, 10=CLOSE_BY) |
| `magic` | `int` | Expert Advisor / strategy magic number |
| `order` | `int` | Order ticket (for modify/remove operations) |
| `symbol` | `str` | Trading symbol name |
| `volume` | `Decimal` | Requested trade volume in lots |
| `price` | `Decimal` | Order price |
| `stoplimit` | `Decimal` | Stop Limit price (for stop-limit orders) |
| `sl` | `Decimal` | Stop Loss price level |
| `tp` | `Decimal` | Take Profit price level |
| `deviation` | `int` | Maximum allowed deviation from requested price (in points) |
| `type` | `int` | Order type (0=BUY, 1=SELL, 2=BUY_LIMIT, 3=SELL_LIMIT, 4=BUY_STOP, 5=SELL_STOP) |
| `type_filling` | `int` | Order filling type (0=FOK, 1=IOC, 2=RETURN, 3=BOC) |
| `type_time` | `int` | Order time type (0=GTC, 1=DAY, 2=SPECIFIED, 3=SPECIFIED_DAY) |
| `expiration` | `int` | Order expiration time (Unix timestamp, for time-limited orders) |
| `comment` | `str` | Order comment text |
| `position` | `int` | Position ticket (for close/modify operations on existing positions) |
| `position_by` | `int` | Opposite position ticket (for close-by operations) |

## Data Flow

```
SignalEvent -> SizingEngine -> RiskEngine -> SuggestedOrder
    |
    v
ExecutionEngine constructs TradeRequest
    |
    v
order_send(TradeRequest) -> OrderSendResult (contains TradeRequest as .request)
```

## Gaps & Issues

1. **All fields required** -- Fields like `stoplimit`, `expiration`, `position`, `position_by` are often 0 or unused depending on the `action` type, but no defaults are provided.
2. **No conditional validation** -- The required fields differ by `action` type (e.g., `TRADE_ACTION_DEAL` needs `symbol`/`volume`/`price`/`type` but not `order`/`position_by`), but no validator enforces these rules.

## Requirements Derived

- **REQ-ENTITY-REQ-001**: `TradeRequest` must support all MT5 trade action types to enable market orders, pending orders, SL/TP modifications, and close-by operations.
- **REQ-ENTITY-REQ-002**: The `magic` field must be set to the strategy ID to enable position-to-strategy association.
