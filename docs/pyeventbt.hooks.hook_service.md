# File: `pyeventbt/hooks/hook_service.py`

## Module

`pyeventbt.hooks.hook_service`

## Purpose

Implements the lifecycle hook system for PyEventBT. Provides the `Hooks` enum defining available hook points and the `HookService` class that manages callback registration and invocation.

## Tags

`hooks`, `callbacks`, `lifecycle`, `extensibility`

## Dependencies

| Dependency | Import |
|---|---|
| `enum` | `Enum` |
| `typing` | `Dict`, `Callable`, `List` |
| `pyeventbt.strategy.core.modules` | `Modules` |

## Classes/Functions

### `Hooks(str, Enum)`

```python
class Hooks(str, Enum):
    ON_START = 'ON_START'
    ON_SIGNAL_EVENT = 'ON_SIGNAL_EVENT'
    ON_ORDER_EVENT = 'ON_ORDER_EVENT'
    ON_END = 'ON_END'
```

**Description**: Enumeration of lifecycle hook points where user callbacks can be registered.

| Member | Value | Description |
|---|---|---|
| `ON_START` | `"ON_START"` | Fires at the start of a backtest or live run, before the main event loop begins |
| `ON_SIGNAL_EVENT` | `"ON_SIGNAL_EVENT"` | Fires when a `SignalEvent` is dequeued (before portfolio handler processes it) |
| `ON_ORDER_EVENT` | `"ON_ORDER_EVENT"` | Fires after an `OrderEvent` has been processed by the execution engine |
| `ON_END` | `"ON_END"` | Fires after the main event loop exits |

**Custom Methods**:

#### `__hash__() -> int`

Returns `hash(self.name)`. Overrides the default enum hash to use the member name.

---

### `HookService`

```python
class HookService:
    def __init__(self) -> None
```

**Description**: Manages registration and invocation of lifecycle hook callbacks. Maintains an internal dictionary mapping `Hooks` members to lists of callback functions. Includes a global enable/disable switch.

**Attributes**:

| Attribute | Type | Visibility | Description |
|---|---|---|---|
| `__hooks_callbacks` | `Dict[Hooks, List[Callable[[Modules], None]]]` | Private | Map of hook type to registered callback list |
| `__hooks_enabled` | `bool` | Private | Global toggle; `True` by default |

---

#### `enable_hooks() -> None`

```python
def enable_hooks(self)
```

Sets `__hooks_enabled = True`. All subsequent `call_callbacks` invocations will execute registered callbacks.

---

#### `disable_hooks() -> None`

```python
def disable_hooks(self)
```

Sets `__hooks_enabled = False`. All subsequent `call_callbacks` invocations will be no-ops.

---

#### `add_hook(hook: Hooks, callback: Callable[[Modules], None]) -> None`

```python
def add_hook(self, hook: Hooks, callback: Callable[[Modules], None])
```

**Description**: Registers a callback for a specific hook point. Multiple callbacks can be registered for the same hook; they execute in registration order.

**Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `hook` | `Hooks` | The lifecycle hook point to attach to |
| `callback` | `Callable[[Modules], None]` | Function to call when the hook fires. Receives a `Modules` instance |

**Implementation**: Uses `dict.setdefault(hook, []).append(callback)` for safe insertion.

---

#### `call_callbacks(hook: Hooks, modules: Modules) -> None`

```python
def call_callbacks(self, hook: Hooks, modules: Modules)
```

**Description**: Invokes all callbacks registered for the given hook, in registration order. If hooks are disabled, returns immediately.

**Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `hook` | `Hooks` | The hook point whose callbacks to invoke |
| `modules` | `Modules` | The `Modules` instance passed to each callback |

**Implementation**: Iterates `self.__hooks_callbacks.get(hook, [])` and calls each callback with `modules`.

## Data Flow

```
User code (Strategy decorators)
    |
    +--> HookService.add_hook(Hooks.ON_START, my_callback)
    |
    ...
    |
TradingDirector.run()
    |
    +--> HookService.call_callbacks(Hooks.ON_START, modules)
    |       |
    |       +--> my_callback(modules)  [if hooks enabled]
    |
    +--> Event loop runs...
    |       |
    |       +--> _handle_signal_event
    |       |       +--> HookService.call_callbacks(Hooks.ON_SIGNAL_EVENT, modules)
    |       |
    |       +--> _handle_order_event
    |               +--> HookService.call_callbacks(Hooks.ON_ORDER_EVENT, modules)
    |
    +--> HookService.call_callbacks(Hooks.ON_END, modules)
```

## Gaps & Issues

1. **No callback removal**: Once registered via `add_hook`, a callback cannot be unregistered. There is no `remove_hook` method.
2. **No error handling in `call_callbacks`**: If any callback raises an exception, it propagates unhandled, potentially terminating the event loop. A try/except with logging would improve robustness.
3. **Callbacks do not receive the triggering event**: The callback signature is `Callable[[Modules], None]`. For `ON_SIGNAL_EVENT` and `ON_ORDER_EVENT`, the callback has no access to the specific event that triggered it.
4. **No ordering control**: Callbacks execute in registration order with no priority mechanism.
5. **`__hash__` override on `Hooks`**: The custom `__hash__` returns `hash(self.name)` rather than the default `Enum.__hash__`. This is unnecessary since `str` enums already have deterministic hashing, and it could cause subtle issues if two differently-valued enums shared a name.
6. **No type checking on `add_hook` arguments**: The `hook` parameter is not validated to be a `Hooks` member at runtime.

## Requirements Derived

| ID | Requirement | Source |
|---|---|---|
| HK-01 | Users must be able to register callbacks for lifecycle events (start, signal, order, end) | `Hooks` enum + `add_hook` method |
| HK-02 | Hooks must be globally enable/disable-able | `enable_hooks` / `disable_hooks` methods |
| HK-03 | Callbacks must receive a `Modules` instance for access to system components | `call_callbacks` signature |
| HK-04 | Multiple callbacks per hook must be supported, executing in registration order | `List[Callable]` storage + iteration |
