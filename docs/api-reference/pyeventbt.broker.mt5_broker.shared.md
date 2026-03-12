# Package: `pyeventbt.broker.mt5_broker.shared`

## Purpose
Provides global mutable state management for the MT5 simulator. Contains the `SharedData` class (pseudo-singleton with class-level attributes) and YAML configuration files defining default account info, terminal info, and symbol definitions for approximately 30 FX pairs.

## Tags
`shared-state`, `singleton`, `yaml`, `configuration`, `defaults`, `simulator`

## Modules

| Module / File | Description |
|---|---|
| `__init__.py` | Empty init with license header. No public exports. |
| `shared_data.py` | `SharedData` class -- global mutable state store with YAML loader methods. |
| `default_account_info.yaml` | Default account configuration (balance: 10000, leverage: 30, currency: USD, etc.). |
| `default_terminal_info.yaml` | Default terminal configuration (connected: false, trade_allowed: true, build number, paths, etc.). |
| `default_symbols_info.yaml` | Default symbol definitions for ~30 FX pairs with full `SymbolInfo` field values. |

## Internal Architecture

```
shared/
  shared_data.py
    |
    +-- __init__() called at class body level in Mt5SimulatorWrapper
    |     |
    |     +-- _load_default_terminal_info() --> default_terminal_info.yaml --> SharedData.terminal_info
    |     +-- _load_default_account_info()  --> default_account_info.yaml  --> SharedData.account_info
    |     +-- _load_default_symbols_info()  --> default_symbols_info.yaml  --> SharedData.symbol_info dict
    |
    +-- Class-level attributes mutated by connector classes:
          SharedData.credentials        (InitCredentials or None)
          SharedData.terminal_info      (TerminalInfo)
          SharedData.account_info       (AccountInfo)
          SharedData.symbol_info        (dict[str, SymbolInfo])
          SharedData.last_error_code    (tuple)
          SharedData.open_positions     (list)
          SharedData.pending_orders     (list)
          SharedData.closed_positions   (list)
          SharedData.history_deals      (list)
          SharedData.next_order_ticket  (counter)
```

The YAML files are loaded relative to the `shared/` directory using `os.path`. The `Decimal` constructor is registered with `yaml.SafeLoader` to preserve financial precision from YAML float values.

## Cross-Package Dependencies

- **Internal**: `core/entities/init_credentials.py`, `core/entities/account_info.py`, `core/entities/terminal_info.py`, `core/entities/symbol_info.py`
- **External**: `pyyaml` (YAML parsing with custom Decimal constructor)
- **External**: `decimal.Decimal` (numeric precision)
- **Consumed by**: All simulator connector classes, `Mt5SimulatorWrapper`, `ExecutionEngine` (for positions/orders lists)

## Gaps & Issues

1. **Not thread-safe** -- Class-level mutable attributes can be concurrently read/written without synchronization. Problematic if multiple strategies or tests share the same process.
2. **Side effect on import** -- `SharedData()` is called in the `Mt5SimulatorWrapper` class body, triggering YAML file loading at module import time.
3. **`SharedData()` reinitializes everything** -- Each `SharedData()` call reloads all YAML files and resets all state, including any previous credentials or positions. No guard against multiple instantiations.
4. **YAML Decimal constructor globally registered** -- `yaml.add_constructor` modifies the global `yaml.SafeLoader`, which could affect other YAML parsing in the same process.
5. **Error handling in `_load_yaml_file`** -- Errors are caught, printed, and `False` is returned, but callers do not check for `False` (they pass the result directly to Pydantic constructors, which would fail with a confusing error).
