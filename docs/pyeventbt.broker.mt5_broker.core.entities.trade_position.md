# File: `pyeventbt/broker/mt5_broker/core/entities/trade_position.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.trade_position`

## Purpose
Defines the `TradePosition` Pydantic model representing an open trading position in MetaTrader 5. Includes all standard MT5 position fields plus a PyEventBT-specific optional field for tracking used margin in account currency.

## Tags
`entity`, `pydantic`, `position`, `mt5`, `trading`, `portfolio`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `typing.Optional` | Stdlib | Optional type for PyEventBT-specific field |
| `decimal.Decimal` | Stdlib | Precision type for price/volume/financial fields |

## Classes/Functions

### `class TradePosition(BaseModel)`

Pydantic model representing an open MT5 position.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `ticket` | `int` | Unique position ticket |
| `time` | `int` | Position open time (Unix timestamp) |
| `time_msc` | `int` | Position open time in milliseconds |
| `time_update` | `int` | Last update time |
| `time_update_msc` | `int` | Last update time in milliseconds |
| `type` | `int` | Position direction: 0=buy, 1=sell |
| `magic` | `int` | Strategy ID (Expert Advisor magic number) |
| `identifier` | `int` | Position identifier |
| `reason` | `int` | Position open reason (see `DEAL_REASON_*` / `POSITION_REASON_*` constants) |
| `volume` | `Decimal` | Position volume in lots |
| `price_open` | `Decimal` | Position open price |
| `sl` | `Decimal` | Stop Loss level |
| `tp` | `Decimal` | Take Profit level |
| `price_current` | `Decimal` | Current market price |
| `swap` | `Decimal` | Accumulated swap |
| `profit` | `Decimal` | Current floating P&L in deposit currency |
| `symbol` | `str` | Trading symbol name |
| `comment` | `str` | Position comment |
| `external_id` | `str` | External system identifier |
| `used_margin_acc_ccy` | `Optional[Decimal]` | **PyEventBT extension**: Used margin in account currency. Not present in standard MT5 API. |

## Data Flow

```
ExecutionEngine fills an order
    |
    v
TradePosition created and added to SharedData.open_positions
    |
    v
Portfolio tracks position, updates profit on each bar
    |
    v
On close: position removed from open_positions, converted to ClosedPosition
```

## Gaps & Issues

1. **PyEventBT extension field** -- `used_margin_acc_ccy` is not part of the standard MT5 API. This could cause issues if position objects are serialized or compared with real MT5 data.
2. **`reason` field references** -- The source comment references `DEAL_REASON_*` constants for the reason enum, but positions use `POSITION_REASON_*` constants. The reference URL in the code points to deal properties, not position properties.
3. **Mutable fields** -- `price_current`, `profit`, `swap`, `sl`, `tp` need to be updated during the position lifetime, requiring Pydantic model mutability.

## Requirements Derived

- **REQ-ENTITY-POS-001**: Open positions must track `magic` number to filter positions by strategy ID.
- **REQ-ENTITY-POS-002**: `profit` must be recalculated on each bar event using current market price.
- **REQ-ENTITY-POS-003**: `used_margin_acc_ccy` must be available for portfolio margin calculations during backtesting.
