# File: `pyeventbt/trading_director/core/configurations/trading_session_configurations.py`

## Module

`pyeventbt.trading_director.core.configurations.trading_session_configurations`

## Purpose

Defines the configuration models for backtesting and live trading sessions. These Pydantic models carry the parameters that `TradingDirector` needs to initialize the correct session type.

## Tags

`configuration`, `pydantic`, `session`, `backtest`, `live-trading`

## Dependencies

| Dependency | Import |
|---|---|
| `pydantic` | `BaseModel` |
| `datetime` | `datetime` |
| `pyeventbt.config.configs` | `Mt5PlatformConfig` |

## Classes/Functions

### `BaseTradingSessionConfig(BaseModel)`

```python
class BaseTradingSessionConfig(BaseModel):
    pass
```

**Description**: Empty base class serving as a type discriminator. `TradingDirector._configure_session()` uses `isinstance` checks against this base to dispatch to the correct configuration handler.

**Attributes**: None.

---

### `MT5BacktestSessionConfig(BaseTradingSessionConfig)`

```python
class MT5BacktestSessionConfig(BaseTradingSessionConfig):
    initial_capital: float
    start_date: datetime
    backtest_name: str
```

**Description**: Configuration for an MT5 backtesting session.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `initial_capital` | `float` | Starting account balance for the backtest |
| `start_date` | `datetime` | Date/time at which the backtest begins |
| `backtest_name` | `str` | Human-readable name for the backtest run; used in result export filenames |

---

### `MT5LiveSessionConfig(BaseTradingSessionConfig)`

```python
class MT5LiveSessionConfig(BaseTradingSessionConfig):
    symbol_list: list[str]
    heartbeat: float
    platform_config: Mt5PlatformConfig
```

**Description**: Configuration for an MT5 live trading session.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `symbol_list` | `list[str]` | List of instrument symbols to trade (e.g., `["EURUSD", "GBPUSD"]`) |
| `heartbeat` | `float` | Interval in seconds between event loop iterations |
| `platform_config` | `Mt5PlatformConfig` | MT5 platform connection configuration (server, login, password, etc.) |

## Data Flow

```
Strategy.backtest() / Strategy.run_live()
    |
    +--> Creates MT5BacktestSessionConfig / MT5LiveSessionConfig
    |
    +--> Passes to TradingDirector.__init__(trading_session_config=...)
            |
            +--> TradingDirector._configure_session()
                    |
                    +--> isinstance(config, MT5BacktestSessionConfig) --> _configure_mt5_backtest_session
                    +--> isinstance(config, MT5LiveSessionConfig)     --> _configure_mt5_live_session
```

## Gaps & Issues

1. **No validation constraints**: `initial_capital` accepts zero or negative values. `heartbeat` has no lower bound. These could cause runtime issues (zero capital, busy-wait loop).
2. **`start_date` is not validated against data availability**: If the start date is outside the available data range, the error would only surface later during backtest execution.
3. **`symbol_list` is unused in the backtest config**: Backtest symbol selection is presumably handled elsewhere (data provider config), creating an asymmetry between session types.

## Requirements Derived

| ID | Requirement | Source |
|---|---|---|
| TSC-01 | Backtest sessions require an initial capital, start date, and backtest name | `MT5BacktestSessionConfig` fields |
| TSC-02 | Live sessions require a symbol list, heartbeat interval, and MT5 platform configuration | `MT5LiveSessionConfig` fields |
| TSC-03 | Session configuration must be polymorphic via a common base type | `BaseTradingSessionConfig` inheritance |
