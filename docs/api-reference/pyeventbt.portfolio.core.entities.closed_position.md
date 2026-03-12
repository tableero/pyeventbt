# pyeventbt.portfolio.core.entities.closed_position

## File
`pyeventbt/portfolio/core/entities/closed_position.py`

## Module
`pyeventbt.portfolio.core.entities.closed_position`

## Purpose
Defines the `ClosedPosition` Pydantic model representing a completed trade with both entry and exit data, along with realised PnL and commission.

## Tags
`entity`, `pydantic`, `position`, `closed-trade`, `domain-model`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class |
| `datetime.datetime` | Entry and exit timestamps |
| `decimal.Decimal` | Precision for prices, volume, commission, pnl, sl, tp, swap |
| `typing.Optional` | Optional fields (sl, tp, swap, comment) |

## Classes/Functions

### `ClosedPosition(BaseModel)`

**Signature:** Pydantic model (no custom `__init__`)

**Description:** Represents a trade that has been fully closed. Contains both entry and exit data, the realised PnL, and commission. Used by the `TradeArchiver` for historical trade records and backtest results export.

**Attributes:**

| Attribute | Type | Required | Description |
|---|---|---|---|
| `time_entry` | `datetime` | Yes | Position entry timestamp |
| `price_entry` | `Decimal` | Yes | Entry price |
| `time_exit` | `datetime` | Yes | Position exit timestamp |
| `price_exit` | `Decimal` | Yes | Exit price |
| `strategy_id` | `str` | Yes | Maps to MT5 Magic Number |
| `ticket` | `int` | Yes | Unique position identifier |
| `symbol` | `str` | Yes | Trading instrument ticker |
| `direction` | `str` | Yes | Trade direction (e.g., `"BUY"`, `"SELL"`) |
| `volume` | `Decimal` | Yes | Position size in lots |
| `commission` | `Decimal` | Yes | Trading commission charged |
| `pnl` | `Decimal` | Yes | Realised profit/loss for this trade |
| `sl` | `Optional[Decimal]` | No | Stop-loss price level |
| `tp` | `Optional[Decimal]` | No | Take-profit price level |
| `swap` | `Optional[Decimal]` | No | Accumulated swap charges |
| `comment` | `Optional[str]` | No | Free-text comment |

**Returns:** N/A (data model)

## Data Flow

```
ExecutionEngine closes a position
  -> FillEvent emitted
  -> PortfolioHandler.process_fill_event() -> TradeArchiver.archive_trade()
  -> TradeArchiver creates ClosedPosition from FillEvent data
  -> Stored in TradeArchiver for export (CSV, Parquet, DataFrame)
```

## Gaps & Issues

1. The `direction` field uses a different name than `OpenPosition.type` for the same concept, creating inconsistency across entity models.
2. `direction` is a plain `str` with no validation constraint.
3. No computed fields (e.g., holding duration, return percentage) that could be derived from the existing data.
4. `ClosedPosition` is imported in `IPortfolio` but `get_closed_positions()` is commented out, so the entity is not used directly by the `Portfolio` class.

## Requirements Derived

- **RQ-CPOS-001**: A closed position must record both entry and exit timestamps and prices.
- **RQ-CPOS-002**: A closed position must include the realised PnL and commission.
- **RQ-CPOS-003**: A closed position must be identifiable by ticket and strategy_id.
