# Package: `pyeventbt.broker.mt5_broker`

## Purpose
Implements the MetaTrader 5 (MT5) broker layer for PyEventBT. Provides both a simulated MT5 environment for backtesting and a live MT5 connector for real trading. The `Mt5SimulatorWrapper` class is the central facade -- it can be imported as a drop-in replacement for `import MetaTrader5 as mt5`, enabling the same user code to run against a simulator or the real MT5 terminal.

## Tags
`mt5`, `broker`, `simulator`, `live-trading`, `backtesting`, `facade`

## Modules

| Module / Sub-package | Description |
|---|---|
| `__init__.py` | Package init. Intentionally empty (license header only). |
| `mt5_simulator_wrapper.py` | `Mt5SimulatorWrapper` class -- drop-in MT5 API replacement with all constants and static method delegates. |
| `connectors/` | Concrete connector implementations (`mt5_simulator_connector.py`, `live_mt5_broker.py`). |
| `core/` | Interfaces (`core/interfaces/`) and Pydantic entity models (`core/entities/`). |
| `shared/` | `SharedData` singleton-like class holding global mutable state and YAML default loaders. |

## Internal Architecture

```
mt5_broker/
  mt5_simulator_wrapper.py    # Facade (constants + static methods)
         |
         v
  connectors/
    mt5_simulator_connector.py   # PlatformConnector, AccountInfoConnector, TerminalInfoConnector, SymbolConnector
    live_mt5_broker.py           # LiveMT5Broker (real MT5 terminal connection)
         |
         v
  shared/
    shared_data.py            # SharedData (class-level mutable state)
    default_account_info.yaml
    default_terminal_info.yaml
    default_symbols_info.yaml
         |
         v
  core/
    interfaces/
      mt5_broker_interface.py  # IPlatform, IAccountInfo, ITerminalInfo, ISymbol, IMarketBook, IMarketData, IOrder, IPosition, IHistory
    entities/
      account_info.py, symbol_info.py, tick.py, terminal_info.py, init_credentials.py,
      order_send_result.py, trade_request.py, trade_deal.py, trade_order.py,
      trade_position.py, mt5_closed_position.py
```

**Data flow (backtest path)**:
1. `Mt5SimulatorWrapper` static methods delegate to connector classes.
2. Connector classes read/write `SharedData` class-level attributes.
3. `SharedData.__init__()` loads YAML defaults on first instantiation.
4. Entity models (Pydantic `BaseModel`) provide typed data structures mirroring MT5's C++ types.

**Data flow (live path)**:
1. `LiveMT5Broker.__init__()` calls real `MetaTrader5` package functions directly.
2. Entity models are shared between both paths for type consistency.

## Cross-Package Dependencies

- **Internal**: `pyeventbt.utils.utils` (`check_platform_compatibility`, `Utils.dateprint`)
- **Internal**: `pyeventbt.config` (`Mt5PlatformConfig`)
- **External**: `MetaTrader5` (optional, imported conditionally for live trading)
- **External**: `pydantic`, `pyyaml`, `python-dotenv`

## Gaps & Issues

1. **SharedData is a pseudo-singleton** -- State is stored as class-level attributes, so any instantiation of `SharedData()` overwrites global state. Not thread-safe.
2. **Incomplete simulator methods** -- `symbol_info_tick()` and `symbol_select()` are marked as not fully implemented. `market_book_add/get/release` raise `NotImplementedError`.
3. **Unimplemented interface classes** -- `IMarketBook`, `IMarketData`, `IOrder`, `IPosition`, `IHistory` are defined in the interface module but have no simulator connector implementations.
4. **Deprecated `initialize_platform`** -- `LiveMT5Broker` contains both `initialize_platform()` (env-var based) and `initialize_platformV2()` (config-based). The V1 method is commented out in `__init__` but still present.
5. **Mixed language comments** -- Some inline comments are in Spanish (e.g., "Anadimos los simbolos al MarketWatch").
