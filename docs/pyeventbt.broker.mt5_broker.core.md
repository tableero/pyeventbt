# Package: `pyeventbt.broker.mt5_broker.core`

## Purpose
Structural core package for the MT5 broker module. Contains the interface definitions (abstract contracts) and entity models (Pydantic data classes) that define the type system and API contract for the MT5 broker layer.

## Tags
`core`, `interfaces`, `entities`, `models`, `pydantic`, `mt5`

## Modules

| Module / Sub-package | Description |
|---|---|
| `__init__.py` | Empty init with license header. No public exports. |
| `interfaces/` | Protocol-based abstract interface definitions for the MT5 API surface. |
| `entities/` | Pydantic `BaseModel` classes mirroring MT5 C++ data structures. |

## Internal Architecture

```
core/
  __init__.py           # Namespace only
  interfaces/
    __init__.py         # Namespace only
    mt5_broker_interface.py   # 8 Protocol classes defining the full MT5 API contract
  entities/
    __init__.py         # Namespace only
    account_info.py     # AccountInfo model (~27 fields)
    symbol_info.py      # SymbolInfo model (~97 fields)
    terminal_info.py    # TerminalInfo model (~22 fields)
    tick.py             # Tick model (8 fields)
    init_credentials.py # InitCredentials model (6 fields)
    order_send_result.py# OrderSendResult model (11 fields)
    trade_request.py    # TradeRequest model (17 fields)
    trade_deal.py       # TradeDeal model (17 fields)
    trade_order.py      # TradeOrder model (23 fields)
    trade_position.py   # TradePosition model (20 fields)
    mt5_closed_position.py # ClosedPosition model (14 fields) -- PyEventBT-specific, not a direct MT5 mirror
```

The `interfaces` sub-package defines what the MT5 API should look like. The `entities` sub-package provides the typed data structures used by both the simulator and live broker paths.

## Cross-Package Dependencies

- **External**: `pydantic` (all entities inherit from `BaseModel`)
- **External**: `decimal.Decimal` (used for all monetary/price fields)
- **Consumed by**: `connectors/`, `shared/`, `mt5_simulator_wrapper.py`, `execution_engine/`

## Gaps & Issues

1. **Interfaces lack parameter signatures** -- Protocol methods in `mt5_broker_interface.py` accept only `self` with no typed parameters, weakening the contract.
2. **`ClosedPosition` diverges from MT5 naming** -- The class in `mt5_closed_position.py` is named `ClosedPosition` (not `MT5ClosedPosition`) and has fields like `direction` (str) and `time_entry`/`time_exit` that do not map to any MT5 native type. This is a PyEventBT-specific abstraction.
3. **No `__init__.py` re-exports** -- All three init files are empty, requiring consumers to use full dotted import paths.
