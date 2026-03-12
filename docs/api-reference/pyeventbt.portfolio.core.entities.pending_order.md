# pyeventbt.portfolio.core.entities.pending_order

## File
`pyeventbt/portfolio/core/entities/pending_order.py`

## Module
`pyeventbt.portfolio.core.entities.pending_order`

## Purpose
Defines the `PendingOrder` Pydantic model representing an unfilled order awaiting execution at a specified price level. Covers limit and stop order types used in MT5.

## Tags
`entity`, `pydantic`, `order`, `pending-order`, `domain-model`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class |
| `decimal.Decimal` | Precision for price, volume, sl, tp |
| `typing.Optional` | Optional fields (sl, tp, comment) |

## Classes/Functions

### `PendingOrder(BaseModel)`

**Signature:** Pydantic model (no custom `__init__`)

**Description:** Represents an unfilled order waiting in the market to be triggered. Supports MT5 pending order types: BUY_LIMIT, SELL_LIMIT, BUY_STOP, SELL_STOP. Also supports BUY and SELL types per the inline comment, though those are typically market orders.

**Attributes:**

| Attribute | Type | Required | Description |
|---|---|---|---|
| `price` | `Decimal` | Yes | Order trigger price |
| `type` | `str` | Yes | Order type: `"BUY_LIMIT"`, `"SELL_LIMIT"`, `"BUY_STOP"`, `"SELL_STOP"` (also `"BUY"`, `"SELL"`) |
| `symbol` | `str` | Yes | Trading instrument ticker |
| `ticket` | `int` | Yes | Unique order identifier in the trading platform |
| `volume` | `Decimal` | Yes | Order size in lots |
| `strategy_id` | `str` | Yes | Maps to MT5 Magic Number |
| `sl` | `Optional[Decimal]` | No | Stop-loss price level |
| `tp` | `Optional[Decimal]` | No | Take-profit price level |
| `comment` | `Optional[str]` | No | Free-text comment |

**Returns:** N/A (data model)

## Data Flow

```
ExecutionEngine (simulated or live MT5)
  -> Creates PendingOrder instances from broker/simulator state
  -> Portfolio._update_portfolio() retrieves via ExecutionEngine._get_strategy_pending_orders()
  -> Portfolio.get_pending_orders() exposes to user code via Modules.PORTFOLIO
```

## Gaps & Issues

1. `type` field is a plain `str` rather than a constrained enum or `Literal`. The inline comment lists numeric codes (0-5) alongside string names but validation does not enforce either format.
2. The comment mentions BUY(0) and SELL(1) as possible types, but these are market orders, not pending orders -- it is unclear whether these values are actually used.
3. Unlike `OpenPosition`, `PendingOrder` has no `time_entry` field, so the order creation timestamp is not tracked.
4. No `swap` field, unlike `OpenPosition`.

## Requirements Derived

- **RQ-PORD-001**: A pending order must specify a trigger price, order type, symbol, unique ticket, volume, and strategy ID.
- **RQ-PORD-002**: Supported pending order types must include BUY_LIMIT, SELL_LIMIT, BUY_STOP, and SELL_STOP.
- **RQ-PORD-003**: Stop-loss, take-profit, and comment are optional metadata fields.
