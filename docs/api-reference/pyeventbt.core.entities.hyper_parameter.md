# Module: pyeventbt.core.entities.hyper_parameter

**File**: `pyeventbt/core/entities/hyper_parameter.py`
**Module**: `pyeventbt.core.entities.hyper_parameter`
**Purpose**: Defines the `HyperParameter` model and its supporting range types for representing optimizable strategy parameters with defined search spaces.
**Tags**: `#core` `#entity` `#hyperparameter` `#optimization` `#pydantic`

---

## Dependencies

| Import | Source |
|---|---|
| `BaseModel` | `pydantic` |
| `Field` | `pydantic` |
| `Any` | `typing` |
| `Variable` | `pyeventbt.core.entities.variable` (relative import) |

---

## Classes

### `HyperParameterRange`

```python
class HyperParameterRange(BaseModel)
```

**Description**: Defines a continuous numeric range for parameter optimization, specified by minimum, maximum, and step size.

**Fields**:

| Field | Type | Default | Description |
|---|---|---|---|
| `minimum` | `float \| int` | Required | Lower bound of the parameter range (inclusive) |
| `maximum` | `float \| int` | Required | Upper bound of the parameter range (inclusive) |
| `step` | `float \| int` | `1` | Step size for iterating through the range |

---

### `HyperParameterValues`

```python
class HyperParameterValues(BaseModel)
```

**Description**: Defines an explicit list of discrete values for parameter optimization.

**Fields**:

| Field | Type | Default | Description |
|---|---|---|---|
| `values` | `list[float \| int]` | Required | Explicit list of parameter values to evaluate |

---

### `HyperParameter`

```python
class HyperParameter(Variable)
```

**Description**: Extends `Variable` to represent an optimizable strategy parameter. Adds a `range` specification that defines the search space for optimization. The `range` can be either a continuous range (`HyperParameterRange`) or discrete values (`HyperParameterValues`).

**Inherits from**: `Variable` (which provides `name: str` and `value: float | int`)

**Fields** (in addition to inherited `name` and `value`):

| Field | Type | Default | Description |
|---|---|---|---|
| `range` | `HyperParameterRange \| HyperParameterValues` | Required | Defines the search space for this parameter |

**Inherited Fields** (from `Variable`):

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Name identifier for the parameter |
| `value` | `float \| int` | Current value of the parameter |

---

## Data Flow

- **Inbound**: Created by the user when defining strategy parameters for optimization. The `range` field specifies which values to explore.
- **Outbound**: Consumed by an optimization engine (not documented in this module set) that iterates over the range or values list to find optimal parameter settings.

---

## Gaps & Issues

1. **Discrepancy with CLAUDE.md description**: The CLAUDE.md describes `HyperParameter` as having fields `name`, `min_value`, `max_value`, `step`, `current_value` with a `range` property that returns a list. The actual implementation uses a separate `HyperParameterRange` model with `minimum`/`maximum`/`step` and a union type for `range`. The CLAUDE.md description appears outdated.
2. **`Field` imported but unused**: `pydantic.Field` is imported but not used anywhere in the module.
3. **`Any` imported but unused**: `typing.Any` is imported (via the `Variable` import chain or directly) but is not used in this module.
4. **No range generation method**: Unlike what CLAUDE.md describes (a `range` property returning a list), the `HyperParameter` class has `range` as a data field, not a computed property. There is no method to generate the actual list of values from a `HyperParameterRange`.
5. **`value` is required**: Since `Variable.value` is typed as `float | int` without a default, the current value must be provided at instantiation even though it would presumably be set by the optimizer during iteration.
6. **No validation**: There is no validation that `minimum <= maximum` in `HyperParameterRange`, or that `step > 0`.

---

## Requirements Derived

- R-HP-01: Strategy parameters must support both continuous ranges (min/max/step) and discrete value lists for optimization.
- R-HP-02: Each hyper-parameter must carry a name, current value, and its search space definition.
- R-HP-03: The framework must support numeric types (`float` and `int`) for parameter values and ranges.
