# Package: `pyeventbt.hooks`

## Purpose

Provides a lifecycle hook system that allows user code to register callbacks at key points during a trading session: start, end, signal generation, and order placement. The `HookService` manages callback registration and invocation.

## Tags

`hooks`, `callbacks`, `lifecycle`, `extensibility`

## Modules

| Module | File | Description |
|---|---|---|
| `hook_service` | `hook_service.py` | Defines the `Hooks` enum (lifecycle hook types) and the `HookService` class (callback registration and dispatch) |
| `__init__` | `__init__.py` | Empty init file |

## Internal Architecture

```
Hooks (enum)
    ON_START
    ON_SIGNAL_EVENT
    ON_ORDER_EVENT
    ON_END

HookService
    __hooks_callbacks: Dict[Hooks, List[Callable[[Modules], None]]]
    __hooks_enabled: bool
    |
    +-- add_hook(hook, callback)       --> registers callback for a hook type
    +-- call_callbacks(hook, modules)  --> invokes all callbacks for a hook type
    +-- enable_hooks() / disable_hooks() --> global on/off switch
```

The `TradingDirector` holds a `HookService` instance and calls `call_callbacks` at the appropriate points in the event loop:

- `ON_START` -- before entering the main loop (in `run()`)
- `ON_SIGNAL_EVENT` -- when a `SignalEvent` is dequeued (in `_handle_signal_event`)
- `ON_ORDER_EVENT` -- after an `OrderEvent` is processed (in `_handle_order_event`)
- `ON_END` -- after the main loop exits (in `run()`)

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.strategy.core.modules` | `Modules` type used as the callback argument |

**Dependents**:

| Package | Usage |
|---|---|
| `pyeventbt.trading_director.trading_director` | Creates and uses `HookService`; calls hooks at lifecycle points |
| `pyeventbt.strategy` | Likely registers user hooks via `Strategy` decorators |

## Gaps & Issues

1. **No `ON_BAR_EVENT` or `ON_FILL_EVENT` hooks**: Only four hook points exist. There is no hook for bar processing or fill processing, limiting observability of the full event lifecycle.
2. **No hook removal API**: Once a callback is added via `add_hook`, there is no way to remove it.
3. **No error isolation**: If a callback raises an exception, it will propagate up and potentially crash the event loop. There is no try/except wrapper around callback invocations.
4. **Callbacks receive only `Modules`**: There is no access to the event that triggered the hook (e.g., the `SignalEvent` or `OrderEvent`). This limits what hooks can inspect.
5. **Empty `__init__.py`**: No re-exports; consumers must use `from pyeventbt.hooks.hook_service import HookService, Hooks`.
