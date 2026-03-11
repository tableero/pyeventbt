# Package: `pyeventbt.trading_context`

## Purpose

Provides a global state mechanism to track whether the system is currently running in backtest or live trading mode. Other components can query this state to adjust their behavior accordingly.

## Tags

`context`, `global-state`, `backtest`, `live-trading`, `enum`

## Modules

| Module | File | Description |
|---|---|---|
| `trading_context` | `trading_context.py` | Defines `TypeContext` enum and module-level getter/setter for the global trading context |
| `__init__` | `__init__.py` | Empty init file |

## Internal Architecture

Extremely simple: a module-level variable `trading_context` of type `TypeContext` defaults to `BACKTEST`. Two functions (`get_trading_context`, `set_trading_context`) provide read/write access. This is a global state pattern -- no classes, no instances.

```
Module-level state:
    trading_context: TypeContext = TypeContext.BACKTEST

    get_trading_context() --> returns current value
    set_trading_context(ctx) --> updates global variable
```

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `enum` | `Enum` base class for `TypeContext` |

**Dependents** (packages that import this module):

| Package | Usage |
|---|---|
| `pyeventbt.trading_director.trading_director` | Imported as `trading_context` module; likely used to set context during session configuration |

## Gaps & Issues

1. **Thread safety**: The global variable is not protected by a lock. In a multi-threaded environment, concurrent reads and writes could produce inconsistent results. However, since PyEventBT uses a single-threaded event loop, this is unlikely to cause issues in practice.
2. **No `__init__.py` re-exports**: Consumers must import the full path `pyeventbt.trading_context.trading_context` or use the module directly.
3. **Global mutable state**: This pattern makes testing harder since the context persists across test runs unless explicitly reset.
4. **`set_trading_context` does not validate input**: Any value (not just `TypeContext` members) could be assigned despite the type hint.
