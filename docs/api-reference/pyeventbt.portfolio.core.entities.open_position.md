# pyeventbt.portfolio.core.entities.open_position

## File
`pyeventbt/portfolio/core/entities/open_position.py`

## Module
`pyeventbt.portfolio.core.entities.open_position`

## Purpose
Defines the `OpenPosition` Pydantic model representing an active (open) trade position in the portfolio. Used throughout the system to represent positions retrieved from the execution engine.

## Tags
`entity`, `pydantic`, `position`, `domain-model`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class |
| `datetime.datetime` | Entry timestamp |
| `decimal.Decimal` | Precision for price, volume, profit, sl, tp, swap |
| `typing.Optional` | Optional fields (sl, tp, swap, comment) |

## Classes/Functions

### `OpenPosition(BaseModel)`

**Signature:** Pydantic model (no custom `__init__`)

**Description:** Represents a currently open trade position. Supports MT5 hedging accounts where multiple positions per symbol are allowed. Created by the execution engine connector and consumed by `Portfolio.get_positions()`.

**Attributes:**

| Attribute | Type | Required | Description |
|---|---|---|---|
| `time_entry` | `datetime` | Yes | Position entry timestamp |
| `price_entry` | `Decimal` | Yes | Entry price |
| `type` | `str` | Yes | Position direction: `"BUY"` or `"SELL"` |
| `symbol` | `str` | Yes | Trading instrument ticker |
| `ticket` | `int` | Yes | Unique position identifier in the trading platform |
| `volume` | `Decimal` | Yes | Position size in lots |
| `unrealized_profit` | `Decimal` | Yes | Current unrealised profit/loss |
| `strategy_id` | `str` | Yes | Maps to MT5 Magic Number; string of digits |
| `sl` | `Optional[Decimal]` | No | Stop-loss price level |
| `tp` | `Optional[Decimal]` | No | Take-profit price level |
| `swap` | `Optional[Decimal]` | No | Accumulated swap charges |
| `comment` | `Optional[str]` | No | Free-text comment |

**Returns:** N/A (data model)

## Data Flow

```
ExecutionEngine (simulated or live MT5)
  -> Creates OpenPosition instances from broker/simulator state
  -> Portfolio._update_portfolio() retrieves via ExecutionEngine._get_strategy_positions()
  -> Portfolio.get_positions() exposes to user code via Modules.PORTFOLIO
```

## Gaps & Issues

1. `type` field is a plain `str` rather than a constrained enum or `Literal["BUY", "SELL"]`, so invalid values (e.g., `"LONG"`, `"SHORT"`) would be silently accepted.
2. No Pydantic field validators for business constraints (e.g., `volume > 0`, `ticket > 0`).
3. The code comment says `time_entry` is "in milliseconds" but the type is `datetime`, not `int` -- the comment appears outdated.

## Requirements Derived

- **RQ-OPOS-001**: An open position must have a defined entry time, entry price, direction, symbol, unique ticket, volume, and strategy ID.
- **RQ-OPOS-002**: Stop-loss, take-profit, swap, and comment are optional metadata fields.
- **RQ-OPOS-003**: The model must support multiple positions per symbol (MT5 hedging account model).
