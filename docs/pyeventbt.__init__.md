# Module: pyeventbt.__init__

**File**: `pyeventbt/__init__.py`
**Module**: `pyeventbt`
**Purpose**: Defines the public API surface of the PyEventBT package by re-exporting all user-facing classes, configurations, and submodules.
**Tags**: `#init` `#public-api` `#re-exports` `#version`

---

## Dependencies

| Import | Source |
|---|---|
| `importlib.metadata.version` | Standard library |
| `Strategy` | `pyeventbt.strategy.strategy` |
| `Modules` | `pyeventbt.strategy.core.modules` |
| `StrategyTimeframes` | `pyeventbt.strategy.core.strategy_timeframes` |
| `BarEvent`, `SignalEvent`, `OrderEvent`, `FillEvent` | `pyeventbt.events.events` |
| `BaseRiskConfig`, `PassthroughRiskConfig` | `pyeventbt.risk_engine.core.configurations.risk_engine_configurations` |
| `BaseSizingConfig`, `MinSizingConfig`, `FixedSizingConfig`, `RiskPctSizingConfig` | `pyeventbt.sizing_engine.core.configurations.sizing_engine_configurations` |
| `indicators` | `pyeventbt.indicators` (submodule) |
| `Mt5PlatformConfig` | `pyeventbt.config.configs` |
| `HyperParameter` | `pyeventbt.core.entities.hyper_parameter` |
| `Variable` | `pyeventbt.core.entities.variable` |
| `Portfolio` | `pyeventbt.portfolio.portfolio` |
| `Bar` | `pyeventbt.data_provider.core.entities.bar` |
| `QuantdleDataUpdater` | `pyeventbt.data_provider.services.quantdle_data_updater` |
| `BacktestResults` | `pyeventbt.backtest.core.backtest_results` |

---

## Module-Level Attributes

### `__version__`

- **Type**: `str`
- **Value**: Dynamically resolved from `importlib.metadata.version("pyeventbt")`. Falls back to `"0.0.0-dev"` if metadata is unavailable (e.g., editable installs without build).

### `__author__`

- **Type**: `str`
- **Value**: `"Marti Castany, Alain Porto"`

### `__website__`

- **Type**: `str`
- **Value**: `"https://github.com/marticastany/pyeventbt"`

### `__license__`

- **Type**: `str`
- **Value**: `"Apache License, Version 2.0"`

### `__description__`

- **Type**: `str`
- **Value**: `"Event-driven backtesting and live trading framework for MetaTrader 5"`

---

## `__all__` Export List

The module explicitly declares `__all__` with the following symbols:

**Package Info**: `__version__`, `__author__`, `__website__`, `__license__`, `__description__`

**Core Strategy**: `Strategy`, `Modules`, `StrategyTimeframes`

**Events**: `BarEvent`, `SignalEvent`, `OrderEvent`, `FillEvent`

**Risk Engine**: `BaseRiskConfig`, `PassthroughRiskConfig`

**Sizing Engine**: `BaseSizingConfig`, `MinSizingConfig`, `FixedSizingConfig`, `RiskPctSizingConfig`

**Indicators**: `indicators` (submodule)

**Configuration**: `Mt5PlatformConfig`

**Core Entities**: `HyperParameter`, `Variable`

**Portfolio**: `Portfolio`

**Data**: `Bar`, `QuantdleDataUpdater`

**Results**: `BacktestResults`

---

## Data Flow

- **Inbound**: This module imports from all major subpackages at package load time. All subpackage initialization code runs when `import pyeventbt` is executed.
- **Outbound**: Provides a flat namespace so users can write `from pyeventbt import Strategy, BarEvent, SignalEvent` without navigating internal package structure.

---

## Gaps & Issues

1. **Version fallback**: The fallback version `"0.0.0-dev"` is a broad catch-all (`except Exception`). This could silently mask import errors in the metadata resolution path.
2. **Eager loading**: All subpackages are imported at module load time, meaning heavy dependencies (numba, polars, matplotlib) are loaded even if the user only needs a subset of functionality.
3. **`BaseRiskConfig` and `BaseSizingConfig` in public API**: These base classes are exported but are likely only useful for creating custom engine configs, not for direct instantiation. Their inclusion in `__all__` may confuse users.
4. **`Portfolio` exported directly**: The `Portfolio` class is in `__all__` but users typically access it through `Modules.PORTFOLIO` rather than instantiating it directly.
5. **Documentation URL**: The docstring references `https://pyeventbt.com` which may not be active.

---

## Requirements Derived

- R-INIT-01: The package must provide a single flat import namespace for all user-facing types.
- R-INIT-02: Version must be dynamically resolved from package metadata to maintain a single source of truth in `pyproject.toml`.
- R-INIT-03: The package must be importable without MetaTrader 5 installed (backtest-only use case).
