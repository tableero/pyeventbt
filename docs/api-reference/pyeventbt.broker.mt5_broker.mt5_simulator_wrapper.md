# File: `pyeventbt/broker/mt5_broker/mt5_simulator_wrapper.py`

## Module
`pyeventbt.broker.mt5_broker.mt5_simulator_wrapper`

## Purpose
Provides `Mt5SimulatorWrapper`, a drop-in replacement for the `MetaTrader5` Python package. Users can import this class in place of `import MetaTrader5 as mt5` and call the same API surface (e.g., `mt5.initialize()`, `mt5.account_info()`, `mt5.symbol_info()`). All method calls are delegated to connector classes that operate on in-memory `SharedData`, enabling fully offline backtesting without a running MT5 terminal.

## Tags
`facade`, `simulator`, `mt5-api`, `constants`, `static-methods`, `backtesting`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `SharedData` | Internal | Instantiated at class body level to initialize global default data |
| `PlatformConnector` | Internal | Delegates `initialize`, `login`, `shutdown`, `version`, `last_error` |
| `AccountInfoConnector` | Internal | Delegates `account_info` |
| `TerminalInfoConnector` | Internal | Delegates `terminal_info` |
| `SymbolConnector` | Internal | Delegates `symbols_total`, `symbols_get`, `symbol_info`, `symbol_info_tick`, `symbol_select` |
| `AccountInfo` | Internal | Return type for `account_info()` |
| `TerminalInfo` | Internal | Return type for `terminal_info()` |
| `SymbolInfo` | Internal | Return type for `symbol_info()`, `symbols_get()` |
| `Tick` | Internal | Return type for `symbol_info_tick()` |
| `typing.Tuple` | Stdlib | Type hint for `symbols_get` return |

## Classes/Functions

### `class Mt5SimulatorWrapper`

Drop-in replacement for the `MetaTrader5` Python package. Contains MT5 enum constants as class attributes and static methods delegating to connector classes.

#### Class-Level Initialization

```python
SharedData()  # Executes SharedData.__init__() to load YAML defaults into class-level state
```

This runs at import time when the class body is evaluated, populating `SharedData` with default account info, terminal info, and symbol info from YAML files.

#### Constants (Class Attributes)

**Timeframe constants**:
- `TIMEFRAME_M1` = 1 through `TIMEFRAME_M30` = 30 (minute timeframes)
- `TIMEFRAME_H1` = 16385 through `TIMEFRAME_H12` = 16396 (hourly, using `0x4000` bit flag)
- `TIMEFRAME_D1` = 16408 (daily), `TIMEFRAME_W1` = 32769 (weekly), `TIMEFRAME_MN1` = 49153 (monthly)

**Tick copy/flag constants**: `COPY_TICKS_ALL`, `COPY_TICKS_INFO`, `COPY_TICKS_TRADE`, `TICK_FLAG_BID`, `TICK_FLAG_ASK`, `TICK_FLAG_LAST`, `TICK_FLAG_VOLUME`, `TICK_FLAG_BUY`, `TICK_FLAG_SELL`

**Position type/reason**: `POSITION_TYPE_BUY` (0), `POSITION_TYPE_SELL` (1), `POSITION_REASON_CLIENT` through `POSITION_REASON_EXPERT`

**Order type**: `ORDER_TYPE_BUY` (0) through `ORDER_TYPE_CLOSE_BY` (8)

**Order state**: `ORDER_STATE_STARTED` (0) through `ORDER_STATE_REQUEST_CANCEL` (9)

**Order filling**: `ORDER_FILLING_FOK` (0), `ORDER_FILLING_IOC` (1), `ORDER_FILLING_RETURN` (2), `ORDER_FILLING_BOC` (3)

**Order time**: `ORDER_TIME_GTC` (0), `ORDER_TIME_DAY` (1), `ORDER_TIME_SPECIFIED` (2), `ORDER_TIME_SPECIFIED_DAY` (3)

**Order reason**: `ORDER_REASON_CLIENT` (0) through `ORDER_REASON_SO` (6)

**Deal type**: `DEAL_TYPE_BUY` (0) through `DEAL_TAX` (17)

**Deal entry**: `DEAL_ENTRY_IN` (0) through `DEAL_ENTRY_OUT_BY` (3)

**Deal reason**: `DEAL_REASON_CLIENT` (0) through `DEAL_REASON_SPLIT` (9)

**Trade action**: `TRADE_ACTION_DEAL` (1), `TRADE_ACTION_PENDING` (5), `TRADE_ACTION_SLTP` (6), `TRADE_ACTION_MODIFY` (7), `TRADE_ACTION_REMOVE` (8), `TRADE_ACTION_CLOSE_BY` (10)

**Symbol enums**: Chart mode, calc mode, trade mode, trade execution, swap mode, day of week, GTC mode, option right/mode

**Account enums**: `ACCOUNT_TRADE_MODE_DEMO` (0), `ACCOUNT_TRADE_MODE_CONTEST` (1), `ACCOUNT_TRADE_MODE_REAL` (2), `ACCOUNT_STOPOUT_MODE_PERCENT` (0), `ACCOUNT_STOPOUT_MODE_MONEY` (1), margin modes

**Book type**: `BOOK_TYPE_SELL` (1) through `BOOK_TYPE_BUY_MARKET` (4)

**Trade return codes**: `TRADE_RETCODE_REQUOTE` (10004) through `TRADE_RETCODE_FIFO_CLOSE` (10045)

**Function error codes**: `RES_S_OK` (1) through `RES_E_INTERNAL_FAIL_TIMEOUT` (-10005)

#### Static Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `initialize` | `(path: str = '', login: int = 0, password: str = '', server: int = 0, timeout: int = 60000, portable: bool = False) -> bool` | Establishes simulated connection. Delegates to `PlatformConnector.initialize()`. | `bool` |
| `login` | `(login: int = 0, password: str = '', server: int = 0, timeout: int = 60000) -> bool` | Simulated account login. Delegates to `PlatformConnector.login()`. | `bool` |
| `shutdown` | `() -> None` | Closes simulated connection. Delegates to `PlatformConnector.shutdown()`. | `None` |
| `version` | `() -> tuple` | Returns simulated terminal version. Delegates to `PlatformConnector.version()`. | `tuple` |
| `last_error` | `() -> tuple` | Returns last error code/message. Delegates to `PlatformConnector.last_error()`. | `tuple` |
| `account_info` | `() -> AccountInfo` | Returns simulated account info. Delegates to `AccountInfoConnector.account_info()`. | `AccountInfo` |
| `terminal_info` | `() -> TerminalInfo` | Returns simulated terminal info. Delegates to `TerminalInfoConnector.terminal_info()`. | `TerminalInfo` |
| `symbols_total` | `() -> int` | Returns total number of symbols. Delegates to `SymbolConnector.symbols_total()`. | `int` |
| `symbols_get` | `(group: str = "*") -> Tuple[SymbolInfo, ...]` | Returns symbols matching group filter. Delegates to `SymbolConnector.symbols_get()`. | `Tuple[SymbolInfo, ...]` |
| `symbol_info` | `(symbol: str) -> SymbolInfo` | Returns info for a specific symbol. Delegates to `SymbolConnector.symbol_info()`. | `SymbolInfo` |
| `symbol_info_tick` | `(symbol: str) -> Tick` | Returns tick info for a symbol. **Not fully implemented.** | `Tick` |
| `symbol_select` | `(symbol: str, enable: bool = True) -> bool` | Selects/deselects symbol in MarketWatch. **Not fully implemented.** | `bool` |
| `market_book_add` | `()` | **Not implemented.** Raises `NotImplementedError`. | -- |
| `market_book_get` | `()` | **Not implemented.** Raises `NotImplementedError`. | -- |
| `market_book_release` | `()` | **Not implemented.** Raises `NotImplementedError`. | -- |

## Data Flow

```
User code (e.g., ExecutionEngine)
    |
    v
Mt5SimulatorWrapper.initialize(...)
    |
    v
PlatformConnector.initialize(...)
    |
    v
SharedData (class-level attributes mutated)
```

All methods follow the same pattern: `Mt5SimulatorWrapper` static method -> corresponding Connector static method -> reads/writes `SharedData` class attributes.

## Gaps & Issues

1. **`symbol_info_tick` returns `None` implicitly** -- The method body has a TODO comment and no return statement after the success path, so it always returns `None`.
2. **`market_book_*` methods unimplemented** -- Three methods raise `NotImplementedError` directly.
3. **Missing MT5 API methods** -- Several real MT5 API methods are absent: `copy_rates_from`, `copy_rates_from_pos`, `copy_rates_range`, `copy_ticks_from`, `copy_ticks_range`, `orders_total`, `orders_get`, `order_calc_margin`, `order_calc_profit`, `order_check`, `order_send`, `positions_total`, `positions_get`, `history_orders_total`, `history_orders_get`, `history_deals_total`, `history_deals_get`. Corresponding interface contracts exist in `mt5_broker_interface.py` but lack simulator implementations.
4. **Class-body side effect** -- `SharedData()` is instantiated in the class body, causing YAML file loading at import time. This is a hidden side effect of importing the module.

## Requirements Derived

- **REQ-BROKER-SIM-001**: The simulator wrapper must expose the same constant values as the real `MetaTrader5` Python package for all supported enumerations.
- **REQ-BROKER-SIM-002**: Static methods must delegate to connector classes rather than implementing logic directly, preserving separation of concerns.
- **REQ-BROKER-SIM-003**: The simulator must be usable without the `MetaTrader5` package installed (backtest-only mode).
- **REQ-BROKER-SIM-004**: `initialize()` must validate credentials and set `SharedData` state, returning `True` on success and `False` on failure.
