# Module: pyeventbt.core.entities.variable

**File**: `pyeventbt/core/entities/variable.py`
**Module**: `pyeventbt.core.entities.variable`
**Purpose**: Defines the `Variable` model, a simple named numeric value container used as the base for strategy parameters and hyper-parameters.
**Tags**: `#core` `#entity` `#variable` `#pydantic` `#base-class`

---

## Dependencies

| Import | Source |
|---|---|
| `BaseModel` | `pydantic` |

---

## Classes

### `Variable`

```python
class Variable(BaseModel)
```

**Description**: A minimal Pydantic model representing a named numeric value. Serves as the base class for `HyperParameter` and can be used standalone as a simple named parameter container.

**Fields**:

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | `str` | Yes | -- | Identifier name for the variable |
| `value` | `float \| int` | Yes | -- | Numeric value held by the variable |

**Usage**:

```python
from pyeventbt import Variable

var = Variable(name="threshold", value=0.5)
print(var.name)   # "threshold"
print(var.value)  # 0.5
```

---

## Data Flow

- **Inbound**: Instantiated by users or by framework internals to hold named numeric values.
- **Outbound**: Used as a base class for `HyperParameter`. Also importable from the top-level `pyeventbt` namespace.

---

## Gaps & Issues

1. **Discrepancy with CLAUDE.md**: The CLAUDE.md describes `Variable` as having `value: Any = None` (a generic container with optional any-typed value). The actual implementation uses `value: float | int` with no default -- it is strictly numeric and required.
2. **No default value**: Both fields are required. A `Variable` cannot be created without specifying both `name` and `value`, which limits its use as a mutable container where the value might be set later.
3. **Limited type support**: Only `float` and `int` are accepted for `value`. String, boolean, or other parameter types are not supported.
4. **No description field**: There is no `description` or `label` field for documenting what the variable represents, which would be useful in optimization reports or UI displays.

---

## Requirements Derived

- R-VAR-01: Named parameters must carry both a string identifier and a numeric value.
- R-VAR-02: The Variable model must be compatible with Pydantic serialization (JSON, dict) for persistence and transport.
