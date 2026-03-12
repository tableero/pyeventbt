# File: `pyeventbt/broker/mt5_broker/connectors/mt5_simulator_connector.py`

## Module
`pyeventbt.broker.mt5_broker.connectors.mt5_simulator_connector`

## Purpose
Implements the simulated MT5 platform connectors for backtesting. Contains four classes that implement the `IPlatform`, `IAccountInfo`, `ITerminalInfo`, and `ISymbol` interfaces. All methods are static and operate on `SharedData` class-level attributes, providing an in-memory simulation of MT5 platform operations without requiring a running MT5 terminal.

## Tags
`connector`, `simulator`, `backtesting`, `mt5`, `static-methods`, `shared-data`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `IPlatform` | Internal | Interface for platform lifecycle |
| `IAccountInfo` | Internal | Interface for account queries |
| `ITerminalInfo` | Internal | Interface for terminal queries |
| `ISymbol` | Internal | Interface for symbol operations |
| `InitCredentials` | Internal | Credential validation model |
| `AccountInfo` | Internal | Return type for account queries |
| `TerminalInfo` | Internal | Return type for terminal queries |
| `SymbolInfo` | Internal | Return type for symbol queries |
| `Tick` | Internal | Return type for tick queries |
| `SharedData` | Internal | Global mutable state store |
| `typing.Tuple` | Stdlib | Type hint for tuple returns |
| `re` | Stdlib | Regex for symbol group filtering |

## Classes/Functions

### `class PlatformConnector(IPlatform)`

Simulates MT5 platform lifecycle operations.

| Method | Signature | Description | Returns |
|---|---|---|---|
| `initialize` | `(path: str = '', login: int = 0, password: str = '', server: int = 0, timeout: int = 60000, portable: bool = False) -> bool` | Validates credentials via `InitCredentials`, stores in `SharedData`, sets `terminal_info.connected = True`, updates `account_info.login` and `account_info.server`. On failure, sets error code `(-6, 'Terminal: Authorization failed')`. | `bool` |
| `login` | `(login: int = 0, password: str = '', server: int = 0, timeout: int = 60000) -> bool` | Same as `initialize` but without `path` and `portable` parameters (hardcoded to `''` and `False`). | `bool` |
| `shutdown` | `() -> None` | Sets `last_error_code = (1, 'success')` and `terminal_info.connected = False`. | `None` |
| `version` | `() -> tuple` | Returns `(500, build, '20 Oct 2023')` where `build` is from `SharedData.terminal_info.build`. | `tuple` |
| `last_error` | `() -> tuple` | Returns `SharedData.last_error_code`. | `tuple` |

### `class AccountInfoConnector(IAccountInfo)`

Simulates MT5 account information retrieval.

| Method | Signature | Description | Returns |
|---|---|---|---|
| `account_info` | `() -> AccountInfo` | Returns `SharedData.account_info`. | `AccountInfo` |

### `class TerminalInfoConnector(ITerminalInfo)`

Simulates MT5 terminal information retrieval.

| Method | Signature | Description | Returns |
|---|---|---|---|
| `terminal_info` | `() -> TerminalInfo` | Returns `SharedData.terminal_info`. | `TerminalInfo` |

### `class SymbolConnector(ISymbol)`

Simulates MT5 symbol operations with wildcard filtering support.

| Method | Signature | Description | Returns |
|---|---|---|---|
| `symbols_total` | `() -> int` | Returns `len(SharedData.symbol_info)`. | `int` |
| `symbols_get` | `(group: str = "*") -> Tuple[SymbolInfo, ...]` | Filters symbols by comma-separated conditions. Supports `*` wildcards (converted to `.*` regex) and `!` prefix for exclusions. Non-string input returns `False` with error code `(-2, 'Invalid 1st unnamed argument')`. | `Tuple[SymbolInfo, ...]` |
| `symbol_info` | `(symbol: str) -> SymbolInfo` | Looks up symbol in `SharedData.symbol_info` dict. Returns `None` with error code `(-4, 'Terminal: Not found')` if not found, or `(-2, 'Invalid arguments')` if not a string. | `SymbolInfo` or `None` |
| `symbol_info_tick` | `(symbol: str) -> Tick` | **Not fully implemented.** Validates symbol exists and is selected, but does not return a Tick object. Has a TODO comment explaining the DataProvider handles tick access during backtest. | `None` (implicitly) |
| `symbol_select` | `(symbol: str, enable: bool = True) -> bool` | Sets `SharedData.symbol_info[symbol].select` and `.visible` to `enable`. Returns `False` with error `(-1, 'Terminal: Call failed')` if symbol not found. TODO: should not allow removal if charts/positions are open. | `bool` |

## Data Flow

```
Mt5SimulatorWrapper.static_method(args)
    |
    v
[PlatformConnector | AccountInfoConnector | TerminalInfoConnector | SymbolConnector].static_method(args)
    |
    v
Read/Write SharedData class-level attributes
    |
    v
Return entity objects or status values
```

**Symbol filtering algorithm** (`symbols_get`):
1. Split `group` string by comma into conditions
2. Apply inclusion conditions: for each non-`!` condition, convert `*` to `.*` regex and filter with `re.fullmatch`
3. Apply exclusion conditions: for each `!`-prefixed condition, remove matching symbols
4. Return tuple of `SymbolInfo` objects for remaining symbols

## Gaps & Issues

1. **`symbol_info_tick` returns `None`** -- The method has no return statement on the success path. The TODO explains this is by design (DataProvider handles ticks), but the interface contract suggests it should return a `Tick`.
2. **`symbol_select` has incomplete TODO** -- Cannot remove symbols with open charts or positions; currently allows unconditional removal.
3. **`symbols_get` inclusion logic is sequential** -- Multiple inclusion conditions are applied sequentially (AND logic), which means `"EUR*,GBP*"` would return nothing (a symbol cannot match both `EUR*` AND `GBP*`). This may not match the real MT5 behavior where multiple patterns are OR-combined.
4. **`login` duplicates `initialize` logic** -- The `login` method is nearly identical to `initialize` but with hardcoded `path=''` and `portable=False`. This violates DRY.
5. **Error code tuple inconsistency** -- Some error messages use lowercase "success", others use capitalized "Success".
6. **Comment at module end** -- The file ends with a comment acknowledging unimplemented interface classes.

## Requirements Derived

- **REQ-SIM-CONN-001**: `initialize()` must validate credentials using Pydantic model validation and propagate login/server to `AccountInfo`.
- **REQ-SIM-CONN-002**: `symbols_get()` must support wildcard pattern matching with inclusion and exclusion conditions.
- **REQ-SIM-CONN-003**: All connector methods must set `SharedData.last_error_code` to reflect the operation outcome.
- **REQ-SIM-CONN-004**: Symbol selection must update both `select` and `visible` flags on `SymbolInfo`.
