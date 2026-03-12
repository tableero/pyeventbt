# File: `pyeventbt/data_provider/core/configurations/data_provider_configurations.py`

## Module

`pyeventbt.data_provider.core.configurations.data_provider_configurations`

## Purpose

Defines Pydantic configuration models used to parameterize data providers. The `DataProvider` service uses `isinstance()` checks on these configs to select the appropriate connector (CSV or MT5 live).

## Tags

`configuration`, `pydantic`, `backtest`, `live`, `csv`, `mt5`, `settings`

## Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class for all config models |
| `pandas` | Imported but **unused** in current source |
| `datetime.datetime` | Type for timestamp fields |

## Classes/Functions

### `BaseDataConfig(BaseModel)`

Empty base configuration model. Serves as a type-dispatch target for the service layer's `isinstance()` check.

**Signature:**
```python
class BaseDataConfig(BaseModel):
    pass
```

**Attributes:** None.

---

### `MT5LiveDataConfig(BaseDataConfig)`

Configuration for live data feed from an MT5 terminal.

**Signature:**
```python
class MT5LiveDataConfig(BaseDataConfig):
    tradeable_symbol_list: list
    timeframes_list: list
```

**Attributes:**

| Attribute | Type | Required | Default | Description |
|---|---|---|---|---|
| `tradeable_symbol_list` | `list` | Yes | -- | List of trading symbols to subscribe to (e.g., `["EURUSD", "USDJPY"]`) |
| `timeframes_list` | `list` | Yes | -- | List of timeframe strings (e.g., `["1min", "1H", "1D"]`) |

---

### `CSVBacktestDataConfig(BaseDataConfig)`

Configuration for CSV-based backtest data ingestion.

**Signature:**
```python
class CSVBacktestDataConfig(BaseDataConfig):
    csv_path: str
    account_currency: str
    tradeable_symbol_list: list
    base_timeframe: str
    timeframes_list: list
    backtest_start_timestamp: datetime | None = None
    backtest_end_timestamp: datetime = datetime.now()
```

**Attributes:**

| Attribute | Type | Required | Default | Description |
|---|---|---|---|---|
| `csv_path` | `str` | Yes | -- | Directory path containing per-symbol CSV files (e.g., `"./data/"`) |
| `account_currency` | `str` | Yes | -- | Account base currency; must be one of `"USD"`, `"EUR"`, `"GBP"` |
| `tradeable_symbol_list` | `list` | Yes | -- | List of symbols to trade |
| `base_timeframe` | `str` | Yes | -- | Lowest-resolution timeframe; must be first element of `timeframes_list` |
| `timeframes_list` | `list` | Yes | -- | Ordered list of timeframes, starting with `base_timeframe` |
| `backtest_start_timestamp` | `datetime | None` | No | `None` | Start of backtest window; `None` means use all available data |
| `backtest_end_timestamp` | `datetime` | No | `datetime.now()` | End of backtest window |

## Data Flow

```
User code
    --> CSVBacktestDataConfig(...) or MT5LiveDataConfig(...)
        --> passed to DataProvider.__init__(data_config=...)
            --> isinstance(data_config, MT5LiveDataConfig) --> Mt5LiveDataProvider
            --> isinstance(data_config, CSVBacktestDataConfig) --> CSVDataProvider
```

## Gaps & Issues

1. **`backtest_end_timestamp` default is evaluated at import time.** `datetime.now()` is called once when the module is first imported, not when each `CSVBacktestDataConfig` instance is created. This means all instances created after the first import share the same default timestamp. The fix is to use `default_factory` or a Pydantic `validator`.
2. **`BaseDataConfig` has no shared fields.** Both subclasses independently define `tradeable_symbol_list`. Lifting this field to the base class would reduce duplication and ensure consistent typing.
3. **Untyped `list` fields.** `tradeable_symbol_list` and `timeframes_list` are typed as bare `list` instead of `list[str]`, losing type safety.
4. **`pandas` is imported but unused.** The `import pandas as pd` statement on line 12 has no consumers in this file.
5. **No `Mt5PlatformConfig` reference.** The knowledge notes and `CLAUDE.md` mention a `plaform_config: Mt5PlatformConfig` field on `MT5LiveDataConfig`, but it does not exist in the actual source. Either it was removed or never added.

## Requirements Derived

- **REQ-DP-CFG-001:** Fix `backtest_end_timestamp` default to use `default_factory=datetime.now` or a Pydantic validator so each instance gets a fresh timestamp.
- **REQ-DP-CFG-002:** Lift `tradeable_symbol_list: list[str]` to `BaseDataConfig`.
- **REQ-DP-CFG-003:** Add explicit generic types to all `list` annotations (e.g., `list[str]`).
- **REQ-DP-CFG-004:** Remove unused `pandas` import.
- **REQ-DP-CFG-005:** If MT5 platform connection parameters are needed, add a `platform_config: Mt5PlatformConfig` field to `MT5LiveDataConfig`.
