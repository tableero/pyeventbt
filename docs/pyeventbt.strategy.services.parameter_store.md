# pyeventbt.strategy.services.parameter_store

**File**: `pyeventbt/strategy/services/parameter_store.py`

**Module**: `pyeventbt.strategy.services.parameter_store`

**Purpose**: Provides a simple key-value store for named parameters, allowing strategies to persist and retrieve configuration values by name.

**Tags**: `#service` `#parameter-management` `#key-value-store` `#bug`

---

## Dependencies

- `typing.Dict` (stdlib)
- `typing.Any` (stdlib)

---

## Classes

### `ParameterStore`

A key-value store for named parameters. Provides methods to add, get, and set parameter values.

#### Class Attributes

| Attribute | Type | Initial Value | Description |
|---|---|---|---|
| `__paramters` | `Dict[str, Any]` | `{}` | **BUG**: Defined at class level, meaning it is shared across all instances. Also contains a typo ("paramters" instead of "parameters"). |

#### Methods

##### `add_parameter(self, parameter_name: str, value: Any) -> None`

Adds a parameter to the store. Uses `dict.setdefault()`, so if the parameter already exists, the existing value is preserved and the new value is silently ignored.

**Parameters**:
| Name | Type | Description |
|---|---|---|
| `parameter_name` | `str` | The key name for the parameter. |
| `value` | `Any` | The value to store. |

**Returns**: `None`

---

##### `get_parameter(self, parameter_name: str) -> Any`

Retrieves a parameter value by name.

**Parameters**:
| Name | Type | Description |
|---|---|---|
| `parameter_name` | `str` | The key name of the parameter to retrieve. |

**Returns**: `Any` -- the stored value.

**Raises**: `KeyError` if `parameter_name` does not exist in the store (standard `dict` behavior, no custom error handling).

---

##### `set_parameter(self, parameter_name: str, value: Any) -> None`

Updates an existing parameter's value.

**Parameters**:
| Name | Type | Description |
|---|---|---|
| `parameter_name` | `str` | The key name of the parameter to update. |
| `value` | `Any` | The new value. |

**Returns**: `None`

**Behavior**: If the key exists, updates the value. If the key does not exist, the `KeyError` is caught and silently swallowed (`except KeyError: pass`).

**Note**: The `try/except` is inverted from the likely intent. `dict.__setitem__` does not raise `KeyError` -- it creates the key if missing. The `except KeyError: pass` block is dead code. The method will always succeed, even for non-existent keys, making it identical in behavior to direct dict assignment.

---

## Data Flow

- **Input**: User code or framework code calls `add_parameter()` or `set_parameter()` to store values.
- **Output**: Values retrieved via `get_parameter()`.

```
Caller --> add_parameter("stop_loss", 50) --> __paramters dict
Caller --> get_parameter("stop_loss") --> 50
Caller --> set_parameter("stop_loss", 75) --> __paramters dict updated
```

---

## Gaps & Issues

1. **Critical bug -- class-level shared state**: `__paramters` is defined as a class attribute (`Dict[str, Any] = {}`), not as an instance attribute in `__init__`. All `ParameterStore` instances share the same dictionary. If two strategies use separate `ParameterStore` instances, they would see and overwrite each other's parameters. The fix is:
   ```python
   def __init__(self):
       self.__paramters: Dict[str, Any] = {}
   ```

2. **Typo in attribute name**: `__paramters` should be `__parameters`. Due to Python name mangling (`_ParameterStore__paramters`), this is cosmetic but harms readability and grep-ability.

3. **`set_parameter` dead exception handling**: `dict.__setitem__` never raises `KeyError`, so `except KeyError: pass` is unreachable. The method always creates or updates the key, contradicting the apparent intent of only updating existing keys.

4. **`add_parameter` silent no-op on duplicates**: Using `dict.setdefault()` means adding a parameter with an existing name silently keeps the old value. No warning or error is raised.

5. **No `has_parameter` or `remove_parameter` methods**: The API is incomplete for a general-purpose parameter store.

6. **Not integrated into the framework**: `ParameterStore` is not used by `Strategy`, `Modules`, or any other class in the codebase. Its intended consumers are unclear.

7. **No thread safety**: If used in a multithreaded context (e.g., live trading), concurrent reads and writes to the shared dict could cause race conditions.

---

## Requirements Derived

1. **REQ-PS-001**: The system shall provide a key-value store for user-defined strategy parameters.
2. **REQ-PS-002**: Parameter addition shall use "first write wins" semantics (subsequent adds for the same key are ignored).
3. **REQ-PS-003**: Parameter retrieval for non-existent keys shall raise a `KeyError`.
4. **REQ-PS-004**: Each `ParameterStore` instance should maintain its own isolated parameter set (currently violated by the class-level attribute bug).
