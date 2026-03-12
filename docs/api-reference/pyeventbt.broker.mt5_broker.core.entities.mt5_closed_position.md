# File: `pyeventbt/broker/mt5_broker/core/entities/mt5_closed_position.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.mt5_closed_position`

## Purpose
Defines the `ClosedPosition` Pydantic model representing a closed (completed) trading position. This is a PyEventBT-specific abstraction that does not directly mirror any single MT5 data structure. It aggregates data from the position open and close lifecycle into a single summary record used for portfolio tracking and performance reporting.

## Tags
`entity`, `pydantic`, `closed-position`, `portfolio`, `pyeventbt-specific`, `performance`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `datetime` | Stdlib | **Imported but unused** |
| `decimal.Decimal` | Stdlib | Precision type for price/financial fields |

## Classes/Functions

### `class ClosedPosition(BaseModel)`

Pydantic model representing a closed position summary. **PyEventBT-specific** -- not a direct MT5 type.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `time_entry` | `int` | Position open time (Unix timestamp) |
| `price_entry` | `Decimal` | Position entry price |
| `magic` | `int` | Strategy ID (Expert Advisor magic number) |
| `ticket` | `int` | Position ticket |
| `symbol` | `str` | Trading symbol name |
| `direction` | `str` | Trade direction as string (e.g., "BUY", "SELL") -- differs from MT5's integer `type` field |
| `volume` | `Decimal` | Position volume in lots |
| `sl` | `Decimal` | Stop Loss level at close time |
| `tp` | `Decimal` | Take Profit level at close time |
| `commission` | `Decimal` | Total roundtrip commission (entry + exit) |
| `swap` | `Decimal` | Accumulated swap charges |
| `time_exit` | `int` | Position close time (Unix timestamp) |
| `price_exit` | `Decimal` | Position exit price |
| `comment` | `str` | Position comment |
| `profit` | `Decimal` | Net profit/loss in deposit currency |

## Data Flow

```
TradePosition (open) + closing FillEvent
    |
    v
Portfolio constructs ClosedPosition with aggregated entry/exit data
    |
    v
Stored in SharedData.closed_positions / Portfolio closed position list
    |
    v
BacktestResults uses closed positions for performance metrics and .plot()
```

## Gaps & Issues

1. **Unused `datetime` import** -- The module imports `datetime` but all time fields use `int` (Unix timestamps).
2. **`direction` is `str` instead of `int`** -- Unlike all other MT5 entity models that use `int` for type/direction enums, `ClosedPosition` uses a `str` field. This breaks consistency and requires string-to-int conversion when correlating with other entities.
3. **File/class naming mismatch** -- File is `mt5_closed_position.py` but the class is `ClosedPosition`, not `MT5ClosedPosition` or `Mt5ClosedPosition`.
4. **No link to constituent deals** -- The closed position does not reference the deal tickets that opened/closed it, making it difficult to reconstruct the full audit trail.

## Requirements Derived

- **REQ-ENTITY-CLPOS-001**: Closed positions must capture both entry and exit prices/times for P&L calculation.
- **REQ-ENTITY-CLPOS-002**: `commission` must reflect the total roundtrip cost (entry + exit commissions combined).
- **REQ-ENTITY-CLPOS-003**: Closed positions must be attributable to a specific strategy via the `magic` field.
