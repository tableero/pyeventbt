# File: `pyeventbt/broker/mt5_broker/shared/shared_data.py`

## Module
`pyeventbt.broker.mt5_broker.shared.shared_data`

## Purpose
Defines the `SharedData` class, which acts as a global mutable state store for the MT5 simulator. Uses class-level attributes (not instance attributes) to hold platform state including credentials, account info, terminal info, symbol info, positions, orders, deals, and error codes. The `__init__` method loads default values from YAML configuration files.

## Tags
`shared-state`, `pseudo-singleton`, `yaml-loader`, `global-state`, `simulator`, `mutable`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `InitCredentials` | Internal | Type for `credentials` attribute |
| `AccountInfo` | Internal | Type for `account_info` attribute |
| `TerminalInfo` | Internal | Type for `terminal_info` attribute |
| `SymbolInfo` | Internal | Type for symbol dict values |
| `os.path` | Stdlib | Resolve YAML file paths relative to module location |
| `decimal.Decimal` | Stdlib | Custom YAML constructor for decimal precision |
| `yaml` | External | YAML file parsing |

## Classes/Functions

### `class SharedData`

Global mutable state store using class-level attributes.

#### Class-Level Attributes

| Attribute | Type | Default | Description |
|---|---|---|---|
| `last_error_code` | `tuple` | `(-1, 'generic fail')` | Last operation error code and message |
| `credentials` | `InitCredentials` | `None` | Current platform credentials |
| `terminal_info` | `TerminalInfo` | `None` (loaded from YAML) | Terminal state and properties |
| `account_info` | `AccountInfo` | `None` (loaded from YAML) | Account state and balances |
| `symbol_info` | `dict[str, SymbolInfo]` | `None` (loaded from YAML) | Symbol name -> SymbolInfo mapping |

**Note**: Additional attributes (`open_positions`, `pending_orders`, `closed_positions`, `history_deals`, `next_order_ticket`) are referenced by other modules but not explicitly declared in this file's class body. They are set by the execution engine during runtime.

#### `__init__(self)`

Initializes all default state by loading YAML files.

```python
def __init__(self):
```

Execution sequence:
1. Sets `SharedData.last_error_code = (-1, 'generic fail')`
2. Calls `_load_default_terminal_info()`
3. Calls `_load_default_account_info()`
4. Calls `_load_default_symbols_info()`

#### `decimal_constructor(loader, node) -> Decimal` (staticmethod)

Custom YAML constructor that converts YAML float nodes to `decimal.Decimal` for financial precision.

```python
@staticmethod
def decimal_constructor(loader, node) -> Decimal
```

**Parameters**:
- `loader`: YAML loader instance
- `node`: YAML scalar node

**Returns**: `Decimal` -- the parsed decimal value

#### `_load_yaml_file(self, filepath: str) -> dict | bool`

Loads and parses a YAML file with the custom Decimal constructor registered.

```python
def _load_yaml_file(self, filepath: str)
```

**Parameters**:
- `filepath` (`str`): Absolute path to the YAML file

**Returns**: Parsed YAML data as a dict, or `False` on error

**Side effect**: Registers `decimal_constructor` globally on `yaml.SafeLoader` via `yaml.add_constructor`.

#### `_load_default_account_info(self) -> None`

Loads `default_account_info.yaml` from the same directory and creates `SharedData.account_info = AccountInfo(**yaml_data)`.

#### `_load_default_terminal_info(self) -> None`

Loads `default_terminal_info.yaml` from the same directory and creates `SharedData.terminal_info = TerminalInfo(**yaml_data)`.

#### `_load_default_symbols_info(self) -> None`

Loads `default_symbols_info.yaml` from the same directory. Iterates the resulting dict (keyed by symbol name) and converts each inner dict to a `SymbolInfo` object. Stores the result in `SharedData.symbol_info`.

## Data Flow

```
Mt5SimulatorWrapper class body
    |
    v
SharedData()  [__init__ called]
    |
    +---> _load_default_terminal_info()
    |         |
    |         v
    |     default_terminal_info.yaml --> TerminalInfo(**data) --> SharedData.terminal_info
    |
    +---> _load_default_account_info()
    |         |
    |         v
    |     default_account_info.yaml --> AccountInfo(**data) --> SharedData.account_info
    |
    +---> _load_default_symbols_info()
              |
              v
          default_symbols_info.yaml --> {symbol: SymbolInfo(**data)} --> SharedData.symbol_info

--- Runtime mutations ---

PlatformConnector.initialize()    --> SharedData.credentials, .terminal_info.connected, .account_info.login/server, .last_error_code
PlatformConnector.shutdown()      --> SharedData.terminal_info.connected, .last_error_code
SymbolConnector.symbol_select()   --> SharedData.symbol_info[symbol].select, .visible
ExecutionEngine (external)        --> SharedData.open_positions, .pending_orders, .closed_positions, .history_deals, .next_order_ticket
```

## Gaps & Issues

1. **Pseudo-singleton anti-pattern** -- State is stored as class-level attributes, but `__init__` can be called multiple times (and is, every time `Mt5SimulatorWrapper` is imported or `SharedData()` is called). Each call resets all state including any accumulated positions or modified account info.
2. **Global YAML loader mutation** -- `yaml.add_constructor('tag:yaml.org,2002:float', ...)` modifies the global `yaml.SafeLoader`. This affects all subsequent YAML parsing in the process, not just SharedData's loads.
3. **`_load_yaml_file` error handling** -- Returns `False` on error, but callers pass the result directly to `AccountInfo(**yaml_data)` / `SymbolInfo(**yaml_data)`, which would raise a `TypeError` if `yaml_data is False`.
4. **No explicit declaration of runtime attributes** -- `open_positions`, `pending_orders`, `closed_positions`, `history_deals`, and `next_order_ticket` are used by external modules but not declared in the class body. This makes the full state surface non-discoverable from this file alone.
5. **Not thread-safe** -- No locking or synchronization on any class-level attribute mutations.
6. **File I/O at import time** -- YAML files are read during module import (when `Mt5SimulatorWrapper` class body evaluates `SharedData()`), which can cause unexpected import-time failures.
7. **Relative path resolution** -- Uses `path.join(path.dirname(__file__), ...)` which works correctly but is fragile if the module is loaded from an unexpected location (e.g., zip imports).

## Requirements Derived

- **REQ-SHARED-001**: Default state must be loaded from YAML files to provide a working simulator environment without user configuration.
- **REQ-SHARED-002**: All financial values in YAML files must be loaded as `Decimal` to preserve precision.
- **REQ-SHARED-003**: `SharedData` must be the single source of truth for simulator state, readable and writable by all connector classes.
- **REQ-SHARED-004**: Default account configuration must provide a reasonable starting state (balance: 10000, leverage: 30, currency: USD) for backtesting.
