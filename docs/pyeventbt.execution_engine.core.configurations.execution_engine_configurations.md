# pyeventbt.execution_engine.core.configurations.execution_engine_configurations

## File
`pyeventbt/execution_engine/core/configurations/execution_engine_configurations.py`

## Module
`pyeventbt.execution_engine.core.configurations.execution_engine_configurations`

## Purpose
Defines Pydantic-based configuration models for selecting and parameterizing execution engine backends. The config type is used by `ExecutionEngine` to instantiate the appropriate connector (simulator or live MT5).

## Tags
`configuration`, `pydantic`, `execution`, `mt5`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class for all config models |
| `decimal.Decimal` | Type for `initial_balance` |

## Classes/Functions

### `BaseExecutionConfig`

| Field | Value |
|---|---|
| **Signature** | `class BaseExecutionConfig(BaseModel)` |
| **Description** | Abstract base configuration for all execution engines. Contains no fields; used as a type discriminator. |
| **Attributes** | None |

### `MT5LiveExecutionConfig`

| Field | Value |
|---|---|
| **Signature** | `class MT5LiveExecutionConfig(BaseExecutionConfig)` |
| **Description** | Configuration for real MT5 broker connectivity. Requires only a magic number to identify strategy orders. |
| **Attributes** | `magic_number: int` -- MT5 magic number used to tag and filter orders/positions belonging to this strategy |

### `MT5SimulatedExecutionConfig`

| Field | Value |
|---|---|
| **Signature** | `class MT5SimulatedExecutionConfig(BaseExecutionConfig)` |
| **Description** | Configuration for the backtesting simulator. Specifies starting capital, currency, leverage, and strategy identifier. |
| **Attributes** | `initial_balance: Decimal` -- starting account balance; `account_currency: str` -- account denomination currency; `account_leverage: int` -- leverage ratio; `magic_number: int` -- strategy identifier |

## Data Flow

```
User creates MT5SimulatedExecutionConfig or MT5LiveExecutionConfig
    --> passed to Strategy
    --> Strategy passes to ExecutionEngine.__init__(execution_config=...)
    --> ExecutionEngine._get_execution_engine(execution_config)
        --> isinstance check selects the appropriate connector
```

## Gaps & Issues

1. **No default values.** Unlike the code knowledge description (which mentioned `AccountCurrencies=USD` and `account_leverage=30`), the actual source has no defaults for `account_currency` or `account_leverage`. Users must specify all fields explicitly.
2. **No `AccountCurrencies` enum.** `account_currency` is a plain `str` with no validation against supported currencies.
3. **No Pydantic validators.** `initial_balance` could be negative, `account_leverage` could be zero or negative, and `magic_number` could be negative -- none are validated.
4. **`magic_number` type.** The field is `int`, but elsewhere `strategy_id` is documented as "must be a string of digits." There is no explicit link or validation ensuring consistency.

## Requirements Derived

- R-EXEC-CFG-01: `MT5SimulatedExecutionConfig` shall require `initial_balance`, `account_currency`, `account_leverage`, and `magic_number`.
- R-EXEC-CFG-02: `MT5LiveExecutionConfig` shall require `magic_number` only.
- R-EXEC-CFG-03: Future versions should add Pydantic validators for positive balance, positive leverage, and valid currency codes.
