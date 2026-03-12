# File: `pyeventbt/broker/mt5_broker/core/entities/tick.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.tick`

## Purpose
Defines the `Tick` Pydantic model representing a single market tick in MetaTrader 5. Mirrors the `MqlTick` structure documented at `mql5.com/en/docs/constants/structures/mqltick`. Contains bid/ask/last prices, volume, and timestamp information.

## Tags
`entity`, `pydantic`, `tick`, `market-data`, `mt5`, `price`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for price fields |

## Classes/Functions

### `class Tick(BaseModel)`

Pydantic model representing a single MT5 market tick.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `time` | `int` | Tick time (Unix timestamp in seconds) |
| `bid` | `Decimal` | Current bid price |
| `ask` | `Decimal` | Current ask price |
| `last` | `Decimal` | Last deal price |
| `volume` | `int` | Volume for the last deal (integer) |
| `time_msc` | `int` | Tick time in milliseconds |
| `flags` | `int` | Tick flags (bitmask of `TICK_FLAG_*` constants indicating which fields changed) |
| `volume_real` | `Decimal` | Volume for the last deal with greater accuracy |

## Data Flow

```
Market data feed (live or simulated)
    |
    v
Tick object created
    |
    v
SymbolConnector.symbol_info_tick(symbol) -> Tick  (NOT FULLY IMPLEMENTED in simulator)
    |
    v
DataProvider.get_latest_tick() (primary access path during backtest)
```

## Gaps & Issues

1. **Simulator `symbol_info_tick` incomplete** -- The `SymbolConnector.symbol_info_tick()` method has a TODO and does not return a `Tick` object. The comment explains that the DataProvider's `get_latest_tick()` method is the intended access path during backtesting, not direct MT5 API calls.
2. **`volume` is `int` while `volume_real` is `Decimal`** -- This mirrors the real MT5 API where `volume` is a legacy integer field and `volume_real` provides higher precision.

## Requirements Derived

- **REQ-ENTITY-TICK-001**: `Tick` must include both `time` (seconds) and `time_msc` (milliseconds) for timestamp precision compatibility with MT5.
- **REQ-ENTITY-TICK-002**: `flags` field must use the same bitmask values as MT5 `TICK_FLAG_*` constants.
