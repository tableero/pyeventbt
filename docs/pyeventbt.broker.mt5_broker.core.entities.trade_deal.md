# File: `pyeventbt/broker/mt5_broker/core/entities/trade_deal.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.trade_deal`

## Purpose
Defines the `TradeDeal` Pydantic model representing a completed deal (execution) in MetaTrader 5. A deal is the actual execution of a trade -- every order that gets filled produces one or more deals. Mirrors the MT5 deal properties documented at `mql5.com/en/docs/constants/tradingconstants/dealproperties`.

## Tags
`entity`, `pydantic`, `deal`, `execution`, `mt5`, `history`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for price/volume/financial fields |

## Classes/Functions

### `class TradeDeal(BaseModel)`

Pydantic model representing a single MT5 deal record.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `ticket` | `int` | Unique deal ticket identifier |
| `order` | `int` | Order ticket that triggered this deal |
| `time` | `int` | Deal execution time (Unix timestamp) |
| `time_msc` | `int` | Deal execution time in milliseconds |
| `type` | `int` | Deal type: 0=buy, 1=sell, 2=balance, 3=credit (see `DEAL_TYPE_*` constants) |
| `entry` | `int` | Deal entry direction: 0=in (open), 1=out (close) |
| `magic` | `int` | Expert Advisor ID / strategy magic number |
| `position_id` | `int` | Position identifier that originated the deal |
| `reason` | `int` | Deal reason (client, mobile, web, expert, SL, TP, etc.) |
| `volume` | `Decimal` | Deal volume in lots |
| `price` | `Decimal` | Deal execution price |
| `commission` | `Decimal` | Commission charged |
| `swap` | `Decimal` | Accumulated swap |
| `profit` | `Decimal` | Profit in deposit currency (0 for entry deals, calculated for exit deals) |
| `fee` | `Decimal` | Fee charged |
| `symbol` | `str` | Trading symbol name |
| `comment` | `str` | Deal comment |
| `external_id` | `str` | External system deal identifier |

## Data Flow

```
Order execution (simulator or live)
    |
    v
TradeDeal created with entry=0 (DEAL_ENTRY_IN) for position open
    |
    v
TradeDeal stored in SharedData.history_deals
    |
    v
On position close: TradeDeal created with entry=1 (DEAL_ENTRY_OUT), profit calculated
```

## Gaps & Issues

1. **`profit` is 0 for entry deals** -- As noted in the source comment, profit is only meaningful for exit deals (`entry=1`). No validator enforces this invariant.
2. **No enum types** -- `type`, `entry`, and `reason` are plain `int` fields rather than Python enums, reducing code readability.

## Requirements Derived

- **REQ-ENTITY-DEAL-001**: Each deal must be linked to its originating position via `position_id` for P&L attribution.
- **REQ-ENTITY-DEAL-002**: The `magic` field must match the `strategy_id` to associate deals with specific strategies.
