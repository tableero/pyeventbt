# Package: `pyeventbt.trading_director.core.configurations`

## Purpose

Provides Pydantic configuration models that define the parameters for different types of trading sessions (backtest vs live). These models are consumed by `TradingDirector._configure_session()` to set up the appropriate session environment.

## Tags

`configuration`, `pydantic`, `session`, `backtest`, `live-trading`

## Modules

| Module | File | Description |
|---|---|---|
| `trading_session_configurations` | `trading_session_configurations.py` | Defines `BaseTradingSessionConfig`, `MT5BacktestSessionConfig`, and `MT5LiveSessionConfig` |
| `__init__` | `__init__.py` | Empty init file |

## Internal Architecture

Simple inheritance hierarchy:

```
BaseTradingSessionConfig (empty base)
    |
    +-- MT5BacktestSessionConfig
    |       initial_capital, start_date, backtest_name
    |
    +-- MT5LiveSessionConfig
            symbol_list, heartbeat, platform_config
```

`TradingDirector._configure_session()` uses `isinstance` checks against these types to determine the session mode.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pydantic` | `BaseModel` for all config classes |
| `datetime` | `datetime` type for `start_date` |
| `pyeventbt.config.configs` | `Mt5PlatformConfig` used in `MT5LiveSessionConfig` |

## Gaps & Issues

1. **No validation on `initial_capital`**: No minimum value or positivity constraint is enforced.
2. **`heartbeat` has no bounds**: A very small or negative heartbeat value is not rejected.
3. **`BaseTradingSessionConfig` is empty**: It exists solely as a type discriminator for `isinstance` checks. An `ABC` or protocol might be more explicit.
4. **No `__init__.py` re-exports**: The `__init__.py` is empty, so consumers must use the full module path.
