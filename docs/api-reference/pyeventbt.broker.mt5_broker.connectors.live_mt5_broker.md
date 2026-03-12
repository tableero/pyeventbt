# File: `pyeventbt/broker/mt5_broker/connectors/live_mt5_broker.py`

## Module
`pyeventbt.broker.mt5_broker.connectors.live_mt5_broker`

## Purpose
Implements the live MT5 broker connector for real-time trading. The `LiveMT5Broker` class initializes a connection to a running MetaTrader 5 terminal, verifies account details, checks algorithmic trading permissions, and adds trading symbols to the MarketWatch. Provides connection status checking methods for the live trading loop.

## Tags
`connector`, `live-trading`, `mt5`, `real-account`, `broker-connection`, `market-watch`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pyeventbt.utils.utils.check_platform_compatibility` | Internal | Platform compatibility check before MT5 import |
| `pyeventbt.broker.mt5_broker.core.entities.order_send_result.OrderSendResult` | Internal | Imported but **not used** in this module |
| `pyeventbt.utils.utils.Utils` | Internal | `Utils.dateprint()` for timestamped error messages |
| `pyeventbt.config.Mt5PlatformConfig` | Internal | Configuration model for MT5 connection parameters |
| `MetaTrader5` (as `mt5`) | External | Real MT5 Python package (conditionally imported) |
| `dotenv` | External | `load_dotenv`, `find_dotenv` -- **imported but not used** (legacy) |
| `logging` | Stdlib | Two loggers: `pyeventbt` and `account_info` |
| `os` | Stdlib | Imported but only used in legacy `initialize_platform()` |
| `time` | Stdlib | `time.sleep()` for account warning countdown |
| `datetime` | Stdlib | Imported but **not used** |

## Classes/Functions

### `class LiveMT5Broker`

Manages the lifecycle of a live MT5 terminal connection.

#### `__init__(self, symbol_list: list, config: Mt5PlatformConfig)`

Constructor. Executes the full initialization sequence:
1. Configures `account_info` logger with green-colored console output
2. Stores `config` reference
3. Calls `initialize_platformV2()` to connect to MT5
4. Calls `_live_account_warning()` to display account type warning with countdown
5. Calls `_print_account_info()` to display formatted account details
6. Calls `_check_algo_trading_enabled()` to verify trading permissions
7. Calls `_add_symbols_to_marketwatch(symbol_list)` to ensure symbols are visible

**Parameters**:
- `symbol_list` (`list`): List of symbol name strings to add to MarketWatch
- `config` (`Mt5PlatformConfig`): MT5 connection configuration

#### `initialize_platform(self)`

**Legacy method** (commented out in `__init__`). Connects to MT5 using environment variables (`MT5_PATH`, `MT5_LOGIN`, `MT5_PASSWORD`, `MT5_SERVER`, `MT5_TIMEOUT`, `MT5_PORTABLE`).

#### `initialize_platformV2(self)`

Connects to MT5 using `self.config` (Mt5PlatformConfig) fields. Logs terminal version on success. Raises and logs exception on failure.

#### `_live_account_warning(self) -> None`

Displays a countdown warning based on account type:
- **DEMO** (`ACCOUNT_TRADE_MODE_DEMO`): 3-second countdown with info-level logging
- **REAL** (`ACCOUNT_TRADE_MODE_REAL`): 10-second countdown with warning-level logging
- **CONTEST**: Single info log, no countdown

#### `_print_account_info(self) -> None`

Prints a formatted box to the `account_info` logger containing: Account ID, Account holder, Broker, Server, Leverage, Account currency, Account balance. Box is 65 characters wide with ASCII borders.

#### `_check_algo_trading_enabled(self) -> None`

Checks `mt5.terminal_info().trade_allowed`. Raises `Exception` if algorithmic trading is disabled.

#### `_add_symbols_to_marketwatch(self, symbols: list) -> None`

Iterates over `symbols` list. For each symbol:
1. Checks if `mt5.symbol_info(symbol)` is not `None`
2. If not already visible, calls `mt5.symbol_select(symbol, True)`
3. Logs success or failure for each symbol

#### `is_connected(self) -> bool`

Returns `mt5.terminal_info().connected`.

#### `is_closed(self) -> bool`

Returns `True` if `mt5.terminal_info()` returns `None` (terminal shut down), `False` otherwise.

## Data Flow

```
Strategy.run_live() creates LiveMT5Broker(symbol_list, config)
    |
    v
__init__:
    initialize_platformV2() --> mt5.initialize(path, login, password, server, timeout, portable)
    _live_account_warning() --> mt5.account_info().trade_mode checked
    _print_account_info()   --> mt5.account_info()._asdict() formatted and logged
    _check_algo_trading_enabled() --> mt5.terminal_info().trade_allowed checked
    _add_symbols_to_marketwatch() --> mt5.symbol_info() / mt5.symbol_select() per symbol
    |
    v
Trading loop uses is_connected() / is_closed() for health checks
```

## Gaps & Issues

1. **`OrderSendResult` import unused** -- Imported at the top of the file but never referenced.
2. **`datetime` import unused** -- Imported but never referenced.
3. **`dotenv` imports unused** -- `load_dotenv` and `find_dotenv` are imported but the `load_dotenv(find_dotenv())` call is commented out.
4. **`os` import partially unused** -- Only needed for the legacy `initialize_platform()` method which is no longer called.
5. **Legacy `initialize_platform` not removed** -- The env-var-based initialization method remains in the codebase despite being replaced by `initialize_platformV2`.
6. **No graceful MT5 unavailability handling** -- If `mt5` is `None` (package not installed or platform incompatible), calling any method on it in `__init__` will raise `AttributeError`. The conditional import sets `mt5 = None` but `__init__` does not check this.
7. **`_asdict()` assumption** -- `_print_account_info` calls `mt5.account_info()._asdict()`, which works with the real MT5 named tuple but would fail with the simulator's Pydantic `AccountInfo` model (which uses `.model_dump()` instead).
8. **Mixed language comments** -- Several inline comments are in Spanish (e.g., "Anadimos los simbolos al MarketWatch", "Recuperamos el objeto de tipo AccountInfo").
9. **`_check_algo_trading_enabled` raises generic `Exception`** -- Should use a custom exception class for better error handling.

## Requirements Derived

- **REQ-LIVE-001**: The live broker must verify account type and display appropriate warnings (longer delay for REAL accounts).
- **REQ-LIVE-002**: All symbols in the strategy's symbol list must be added to MarketWatch before trading begins.
- **REQ-LIVE-003**: Algorithmic trading must be verified as enabled before allowing the trading loop to start.
- **REQ-LIVE-004**: Connection health must be queryable via `is_connected()` and `is_closed()` during the live trading loop.
