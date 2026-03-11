# Package: `pyeventbt.broker.mt5_broker.connectors`

## Purpose
Contains concrete connector implementations for the MT5 broker layer. Two connectors serve the two execution paths: `mt5_simulator_connector.py` provides an in-memory simulated MT5 environment for backtesting, while `live_mt5_broker.py` wraps the real MetaTrader 5 Python package for live trading.

## Tags
`connectors`, `simulator`, `live-trading`, `mt5`, `execution`

## Modules

| Module | Description |
|---|---|
| `__init__.py` | Empty init with license header. No public exports. |
| `mt5_simulator_connector.py` | Four connector classes (`PlatformConnector`, `AccountInfoConnector`, `TerminalInfoConnector`, `SymbolConnector`) implementing `IPlatform`, `IAccountInfo`, `ITerminalInfo`, `ISymbol` for simulated backtesting. |
| `live_mt5_broker.py` | `LiveMT5Broker` class that initializes and manages a real MT5 terminal connection for live trading. |

## Internal Architecture

```
connectors/
  mt5_simulator_connector.py
    |
    +-- PlatformConnector(IPlatform)       --> SharedData.credentials, .terminal_info, .account_info, .last_error_code
    +-- AccountInfoConnector(IAccountInfo)  --> SharedData.account_info
    +-- TerminalInfoConnector(ITerminalInfo)--> SharedData.terminal_info
    +-- SymbolConnector(ISymbol)            --> SharedData.symbol_info

  live_mt5_broker.py
    |
    +-- LiveMT5Broker                      --> import MetaTrader5 as mt5 (real terminal)
```

The simulator connectors are stateless (all static methods) and operate on `SharedData` class-level attributes. The live broker is stateful, holding a reference to its `Mt5PlatformConfig`.

## Cross-Package Dependencies

- **Internal**: `core/interfaces/mt5_broker_interface.py` (interface contracts)
- **Internal**: `core/entities/` (entity models for parameters and return types)
- **Internal**: `shared/shared_data.py` (global mutable state, simulator only)
- **Internal**: `pyeventbt.utils.utils` (`check_platform_compatibility`, `Utils.dateprint`)
- **Internal**: `pyeventbt.config` (`Mt5PlatformConfig`)
- **External**: `MetaTrader5` (optional, live broker only)
- **External**: `re` (regex for symbol filtering in simulator)
- **External**: `python-dotenv` (imported in live broker, legacy usage)

## Gaps & Issues

1. **Asymmetric implementations** -- The simulator has 4 fine-grained connector classes following the interface contracts, while the live broker is a single monolithic class that does not implement any of the Protocol interfaces.
2. **No simulator connectors for order/position/history** -- The `IOrder`, `IPosition`, `IHistory`, `IMarketData`, and `IMarketBook` interfaces have no simulator implementations. Order execution in backtesting is handled elsewhere in the execution engine, not through these connectors.
3. **Legacy code in live broker** -- `initialize_platform()` (env-var based) is still present alongside `initialize_platformV2()` (config-based). The old method is commented out in `__init__` but not removed.
4. **`dotenv` import not used** -- `live_mt5_broker.py` imports `load_dotenv` and `find_dotenv` but the calls are commented out.
