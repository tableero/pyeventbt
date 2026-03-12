# Package: `pyeventbt.broker.mt5_broker.core.entities`

## Purpose
Collection of Pydantic `BaseModel` classes that mirror the data structures returned by the MetaTrader 5 Python API. These models provide type-safe, validated representations of MT5 account info, symbol info, ticks, trade requests, orders, deals, positions, and related objects. They are used by both the simulator and live broker paths.

## Tags
`entities`, `pydantic`, `data-models`, `mt5`, `trading`, `decimal`

## Modules

| Module | Primary Class | Field Count | Description |
|---|---|---|---|
| `account_info.py` | `AccountInfo` | 27 | Trading account properties (balance, equity, margin, leverage, etc.) |
| `symbol_info.py` | `SymbolInfo` | 97 | Full symbol specification (pricing, volumes, trade rules, session data, Greeks) |
| `terminal_info.py` | `TerminalInfo` | 22 | MT5 terminal state (connection, build, paths, permissions) |
| `tick.py` | `Tick` | 8 | Single market tick (bid, ask, last, volume, timestamp) |
| `init_credentials.py` | `InitCredentials` | 6 | MT5 platform initialization credentials |
| `order_send_result.py` | `OrderSendResult` | 11 | Result of an `order_send` call (includes nested `TradeRequest`) |
| `trade_request.py` | `TradeRequest` | 17 | Parameters for placing/modifying a trade order |
| `trade_deal.py` | `TradeDeal` | 17 | Completed deal record (execution details) |
| `trade_order.py` | `TradeOrder` | 23 | Pending/historical order record |
| `trade_position.py` | `TradePosition` | 20 | Open position record (includes optional `used_margin_acc_ccy`) |
| `mt5_closed_position.py` | `ClosedPosition` | 14 | PyEventBT-specific closed position summary |

## Internal Architecture

All entity classes follow the same pattern:
- Inherit from `pydantic.BaseModel`
- Use `decimal.Decimal` for all monetary and price fields (preserving precision)
- Use `int` for timestamps (Unix epoch seconds) and enum-like fields
- Use `str` for identifiers, symbols, and comments

**Dependency graph between entities**:
```
OrderSendResult
    |
    +---> TradeRequest  (nested as `request` field)
```

All other entities are standalone with no inter-entity dependencies.

## Cross-Package Dependencies

- **External**: `pydantic.BaseModel`
- **External**: `decimal.Decimal`
- **External**: `typing.Optional` (used only in `TradePosition`)
- **External**: `datetime` (imported but unused in `mt5_closed_position.py`)
- **Consumed by**: `SharedData`, all connector classes, `ExecutionEngine`, `Portfolio`

## Gaps & Issues

1. **`ClosedPosition` naming mismatch** -- The file is named `mt5_closed_position.py` but the class is `ClosedPosition`, not `MT5ClosedPosition`. This class has no direct MT5 counterpart.
2. **Unused `datetime` import** -- `mt5_closed_position.py` imports `datetime` but uses `int` for time fields.
3. **No `_asdict()` method** -- The live broker calls `mt5.account_info()._asdict()`, which works with the real MT5 named tuple return. Pydantic models have `.model_dump()` instead. There may be an incompatibility if simulator results are passed to code expecting named-tuple behavior.
4. **No default values** -- All fields are required with no defaults, meaning every field must be provided at construction time.
5. **No validators** -- No Pydantic field validators are defined (e.g., ensuring `volume >= 0` or `type` is within valid enum range).
