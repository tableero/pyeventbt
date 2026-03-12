# File: `pyeventbt/data_provider/core/entities/bar.py`

## Module

`pyeventbt.data_provider.core.entities.bar`

## Purpose

Defines a Pydantic `Bar` model representing a single OHLCV financial bar with optional fields for adjusted close, volume, spread, and open interest. This entity uses `float` prices (not integer-scaled) and appears to be a legacy/unused definition superseded by the `Bar` dataclass in `pyeventbt.events.events`.

## Tags

`entity`, `bar`, `pydantic`, `legacy`, `unused`, `ohlcv`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class for the model |
| `datetime.datetime` | Type for the `datetime` field |
| `typing.Optional` | Optional field annotations |

## Classes/Functions

### `Bar(BaseModel)`

A Pydantic model representing a financial OHLCV bar.

**Signature:**
```python
class Bar(BaseModel):
    datetime: datetime
    open: float
    high: float
    low: float
    close: float
    adj_close: Optional[float]
    volume: Optional[int]
    spread: Optional[int]
    open_interest: Optional[int]
```

**Attributes:**

| Attribute | Type | Required | Description |
|---|---|---|---|
| `datetime` | `datetime` | Yes | Timestamp of the bar |
| `open` | `float` | Yes | Opening price |
| `high` | `float` | Yes | Highest price during the period |
| `low` | `float` | Yes | Lowest price during the period |
| `close` | `float` | Yes | Closing price |
| `adj_close` | `Optional[float]` | No | Adjusted closing price (for equities with dividends/splits) |
| `volume` | `Optional[int]` | No | Trading volume |
| `spread` | `Optional[int]` | No | Bid/ask spread in points |
| `open_interest` | `Optional[int]` | No | Open interest (for futures) |

**Returns:** N/A (data model)

## Data Flow

This entity is not referenced by any other module in the codebase. The import in `data_provider_interface.py` is commented out:
```python
#from ..entities.bar import Bar
```

All runtime bar handling uses `Bar` from `pyeventbt.events.events`, which is a `dataclass` (not Pydantic) with integer-scaled prices and a `digits` field.

## Gaps & Issues

1. **Completely unused.** No module imports or instantiates this class. It is dead code.
2. **Different schema from the runtime `Bar`.** The events `Bar` uses integer prices (`open`, `high`, `low`, `close` as `int`) plus a `digits` field for decimal reconstruction, plus `tickvol`. This Pydantic `Bar` uses `float` prices, has no `digits` or `tickvol`, and includes `adj_close` and `open_interest` which the events `Bar` lacks.
3. **No default values for optional fields.** `adj_close`, `volume`, `spread`, and `open_interest` are typed as `Optional` but have no `= None` default, so Pydantic will require them to be explicitly passed (even as `None`).

## Requirements Derived

- **REQ-DP-ENT-001:** If this entity is retained, it should either be brought into alignment with the runtime `Bar` or explicitly marked as deprecated.
- **REQ-DP-ENT-002:** Optional fields should have `= None` defaults to allow partial construction.
- **REQ-DP-ENT-003:** Consider removing this file entirely if no consumer exists.
