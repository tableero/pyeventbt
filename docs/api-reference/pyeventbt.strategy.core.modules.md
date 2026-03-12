# pyeventbt.strategy.core.modules

**File**: `pyeventbt/strategy/core/modules.py`

**Module**: `pyeventbt.strategy.core.modules`

**Purpose**: Defines the `Modules` Pydantic model, a dependency-injection container that bundles the core framework components (trading context, data provider, execution engine, portfolio) and is passed into every user-defined callback function.

**Tags**: `#pydantic` `#dependency-injection` `#data-model` `#user-api`

---

## Dependencies

- `pydantic.BaseModel`
- `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider`
- `pyeventbt.execution_engine.core.interfaces.execution_engine_interface.IExecutionEngine`
- `pyeventbt.portfolio.core.interfaces.portfolio_interface.IPortfolio`
- `pyeventbt.trading_context.trading_context.TypeContext`

---

## Classes

### `Modules(BaseModel)`

A Pydantic model that acts as a service locator / dependency-injection container. An instance is created during `Strategy.backtest()` or `Strategy.run_live()` and passed to all user-defined signal, sizing, risk, scheduled, and hook callback functions.

#### Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `TRADING_CONTEXT` | `TypeContext` | `TypeContext.BACKTEST` | The current execution context (BACKTEST or LIVE). |
| `DATA_PROVIDER` | `IDataProvider` | -- (required) | Interface to the data provider. Provides access to bar data via `get_latest_bars()`. |
| `EXECUTION_ENGINE` | `IExecutionEngine` | -- (required) | Interface to the execution engine. Allows manual order placement. |
| `PORTFOLIO` | `IPortfolio` | -- (required) | Interface to the portfolio. Provides access to open/closed positions and account balance. |

#### Config

```python
class Config:
    arbitrary_types_allowed = True
```

Required because the interface types (`IDataProvider`, `IExecutionEngine`, `IPortfolio`) are not standard Pydantic-serializable types.

---

## Data Flow

- **Input**: Created by `Strategy.backtest()` or `Strategy.run_live()` after all framework components are instantiated.
- **Output**: Passed as a parameter to every user callback:
  - Signal engine: `fn(bar_event: BarEvent, modules: Modules) -> SignalEvent`
  - Sizing engine: `fn(signal_event: SignalEvent, modules: Modules) -> SuggestedOrder`
  - Risk engine: `fn(suggested_order: SuggestedOrder, modules: Modules) -> float`
  - Scheduled callback: `fn(scheduled_event: ScheduledEvent, modules: Modules) -> None`
  - Hook callback: `fn(modules: Modules) -> None`

Users access framework services through the Modules fields:

```python
@strategy.custom_signal_engine()
def my_signal_engine(bar_event: BarEvent, modules: Modules) -> SignalEvent:
    bars = modules.DATA_PROVIDER.get_latest_bars("EURUSD", N=50)
    positions = modules.PORTFOLIO.get_open_positions()
    ...
```

---

## Gaps & Issues

1. **Uses Pydantic v1-style `Config` inner class**: Pydantic v2 recommends `model_config = ConfigDict(...)` instead. The `walk_forward.py` module in the same package already uses the v2 style, creating inconsistency.
2. **No validation or immutability**: The model does not enforce immutability (`frozen=True`), meaning user callbacks could accidentally mutate the shared `Modules` instance.
3. **ALL_CAPS field names**: While intentional to signal "these are framework-level constants," this deviates from Python naming conventions for instance attributes and may confuse static analysis tools.

---

## Requirements Derived

1. **REQ-MOD-001**: The `Modules` container shall provide user callbacks with access to the data provider, execution engine, portfolio, and trading context.
2. **REQ-MOD-002**: The `Modules` container shall support arbitrary (non-serializable) types as field values.
3. **REQ-MOD-003**: A single `Modules` instance shall be shared across all callbacks within a single backtest or live trading session.
