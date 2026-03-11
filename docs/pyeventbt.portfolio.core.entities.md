# pyeventbt.portfolio.core.entities

## Package
`pyeventbt.portfolio.core.entities`

## Purpose
Domain entity models for the portfolio module. Contains Pydantic `BaseModel` data classes representing open positions, closed positions, and pending orders. These are pure data containers with validation but no business logic.

## Tags
`entities`, `domain-model`, `pydantic`, `positions`, `orders`

## Modules

| Module | Path | Description |
|---|---|---|
| `open_position` | `portfolio/core/entities/open_position.py` | `OpenPosition` model representing an active trade position |
| `closed_position` | `portfolio/core/entities/closed_position.py` | `ClosedPosition` model representing a completed trade with entry/exit data |
| `pending_order` | `portfolio/core/entities/pending_order.py` | `PendingOrder` model representing an unfilled order awaiting execution |

## Internal Architecture

All three entities are Pydantic `BaseModel` subclasses. They share a common set of fields (symbol, ticket, volume, strategy_id, sl, tp, comment) with entity-specific additions:

- **`OpenPosition`** adds `time_entry`, `price_entry`, `type` (BUY/SELL), `unrealized_profit`, `swap`.
- **`ClosedPosition`** adds `time_entry`, `price_entry`, `time_exit`, `price_exit`, `direction`, `commission`, `pnl`, `swap`.
- **`PendingOrder`** adds `price`, `type` (BUY_LIMIT/SELL_LIMIT/BUY_STOP/SELL_STOP).

These entities are created by the execution engine connectors (simulated or live MT5) and consumed by the `Portfolio` class and user-facing `Modules` API.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class for all entities |
| `decimal.Decimal` | Precision type for all monetary/price fields |
| `datetime.datetime` | Timestamp fields for entry/exit times |

## Gaps & Issues

1. `OpenPosition.type` and `PendingOrder.type` are plain `str` rather than an enum, allowing invalid values without Pydantic validation errors.
2. Field naming is inconsistent across entities: `OpenPosition` uses `price_entry` and `type`, while `ClosedPosition` uses `price_entry`/`price_exit` and `direction` instead of `type`.
3. `ClosedPosition` uses `pnl` while `OpenPosition` uses `unrealized_profit` -- different naming conventions for the same concept (profit/loss).
4. No shared base class for common fields (symbol, ticket, volume, strategy_id, sl, tp, comment) leading to field duplication across all three entities.
