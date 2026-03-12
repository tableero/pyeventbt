# File: `pyeventbt/trading_context/trading_context.py`

## Module

`pyeventbt.trading_context.trading_context`

## Purpose

Maintains a global flag indicating whether the system is operating in backtest or live trading mode. Used by other components to branch behavior based on the execution context.

## Tags

`context`, `global-state`, `enum`, `backtest`, `live-trading`

## Dependencies

| Dependency | Import |
|---|---|
| `enum` | `Enum` |

## Classes/Functions

### `TypeContext(str, Enum)`

```python
class TypeContext(str, Enum):
    LIVE = "LIVE"
    BACKTEST = "BACKTEST"
```

**Description**: Enumeration of the two possible execution contexts.

| Member | Value | Description |
|---|---|---|
| `LIVE` | `"LIVE"` | System is running in live trading mode |
| `BACKTEST` | `"BACKTEST"` | System is running a historical backtest |

---

### Module-Level Variable: `trading_context`

```python
trading_context = TypeContext.BACKTEST
```

**Description**: The global state variable holding the current execution context. Defaults to `BACKTEST`.

---

### `get_trading_context() -> TypeContext`

```python
def get_trading_context():
    return trading_context
```

**Description**: Returns the current trading context.

**Returns**: `TypeContext` -- the current context (`LIVE` or `BACKTEST`).

---

### `set_trading_context(context: TypeContext) -> None`

```python
def set_trading_context(context: TypeContext):
    global trading_context
    trading_context = context
```

**Description**: Sets the global trading context. Uses the `global` keyword to modify the module-level variable.

**Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `context` | `TypeContext` | The new context to set |

## Data Flow

```
Strategy.backtest() / Strategy.run_live()
    |
    +--> set_trading_context(TypeContext.BACKTEST / TypeContext.LIVE)
    |
    +--> TradingDirector runs event loop
            |
            +--> Any component can call get_trading_context()
                 to check if running backtest or live
```

## Gaps & Issues

1. **No input validation**: `set_trading_context` accepts any value despite the `TypeContext` type hint. Passing a string like `"PAPER"` would silently work at runtime.
2. **No return type annotation on `get_trading_context`**: The function lacks a return type hint.
3. **Thread safety**: The `global` variable is not protected by any synchronization primitive. Not an issue for the single-threaded event loop but could be if the architecture evolves.
4. **Default is BACKTEST**: If `set_trading_context` is never called, the system assumes backtest mode. This is a reasonable default but is implicit.

## Requirements Derived

| ID | Requirement | Source |
|---|---|---|
| TC-01 | The system must track whether it is in backtest or live mode via a global context | Module-level `trading_context` variable |
| TC-02 | The default context must be `BACKTEST` | `trading_context = TypeContext.BACKTEST` |
| TC-03 | Components must be able to read the context without receiving it as a parameter | `get_trading_context()` function |
