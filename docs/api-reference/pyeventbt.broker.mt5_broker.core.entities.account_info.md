# File: `pyeventbt/broker/mt5_broker/core/entities/account_info.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.account_info`

## Purpose
Defines the `AccountInfo` Pydantic model representing a MetaTrader 5 trading account's properties. Mirrors the structure returned by `mt5.account_info()` in the real MT5 Python API. Used by both the simulator (via `SharedData`) and live broker paths.

## Tags
`entity`, `pydantic`, `account`, `mt5`, `trading`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for all monetary fields |

## Classes/Functions

### `class AccountInfo(BaseModel)`

Pydantic model representing MT5 account information.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `login` | `int` | Account login number |
| `trade_mode` | `int` | Account trade mode (0=DEMO, 1=CONTEST, 2=REAL) |
| `leverage` | `int` | Account leverage ratio |
| `limit_orders` | `int` | Maximum number of pending orders allowed |
| `margin_so_mode` | `int` | Stop Out mode (0=PERCENT, 1=MONEY) |
| `trade_allowed` | `bool` | Whether trading is allowed on the account |
| `trade_expert` | `bool` | Whether Expert Advisor trading is allowed |
| `margin_mode` | `int` | Margin calculation mode (0=NETTING, 1=EXCHANGE, 2=HEDGING) |
| `currency_digits` | `int` | Number of decimal places for the account currency |
| `fifo_close` | `bool` | Whether FIFO rule is applied for position closing |
| `balance` | `Decimal` | Account balance in deposit currency |
| `credit` | `Decimal` | Credit facility amount |
| `profit` | `Decimal` | Current floating profit/loss |
| `equity` | `Decimal` | Account equity (balance + credit + profit) |
| `margin` | `Decimal` | Margin currently used |
| `margin_free` | `Decimal` | Free margin available |
| `margin_level` | `Decimal` | Margin level as percentage |
| `margin_so_call` | `Decimal` | Margin Call level |
| `margin_so_so` | `Decimal` | Stop Out level |
| `margin_initial` | `Decimal` | Initial margin required for all open positions |
| `margin_maintenance` | `Decimal` | Maintenance margin for all open positions |
| `assets` | `Decimal` | Current asset value |
| `liabilities` | `Decimal` | Current liabilities |
| `commission_blocked` | `Decimal` | Blocked commission amount |
| `name` | `str` | Account holder name |
| `server` | `str` | Trade server name |
| `currency` | `str` | Account currency (e.g., "USD") |
| `company` | `str` | Broker company name |

## Data Flow

```
YAML (default_account_info.yaml)
    |
    v
SharedData._load_default_account_info()
    |
    v
AccountInfo(**yaml_data)  -->  SharedData.account_info (class-level attr)
    |
    v
AccountInfoConnector.account_info() returns SharedData.account_info
    |
    v
Mt5SimulatorWrapper.account_info() / LiveMT5Broker._print_account_info()
```

## Gaps & Issues

1. **No `_asdict()` method** -- The live broker code calls `mt5.account_info()._asdict()`. Real MT5 returns a named tuple with `_asdict()`. Pydantic models use `.model_dump()` instead. This could cause runtime errors if simulator `AccountInfo` is passed to code expecting the named-tuple API.
2. **No field validation** -- No range checks on fields like `leverage` (should be positive), `trade_mode` (should be 0-2), or `margin_level` (should be non-negative).
3. **All fields required** -- No defaults are provided; every field must be supplied during construction.

## Requirements Derived

- **REQ-ENTITY-ACC-001**: `AccountInfo` must contain all fields returned by the real `mt5.account_info()` call to ensure API compatibility.
- **REQ-ENTITY-ACC-002**: Monetary fields must use `Decimal` type to prevent floating-point precision loss in financial calculations.
