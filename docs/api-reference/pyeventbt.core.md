# Package: pyeventbt.core

**Package**: `pyeventbt.core`
**Purpose**: Contains shared core entities used across the framework, including parameter management (`HyperParameter`) and generic value containers (`Variable`).
**Tags**: `#core` `#entities` `#parameters` `#shared`

---

## Modules

| Module | Description |
|---|---|
| `__init__.py` | Empty package marker |
| `entities/__init__.py` | Empty package marker |
| `entities/hyper_parameter.py` | `HyperParameter` model for optimization parameter ranges and discrete value sets |
| `entities/variable.py` | `Variable` model -- simple named numeric value container |

---

## Internal Architecture

```
core/
  __init__.py
  entities/
    __init__.py
    variable.py          <-- Variable(BaseModel): name + value
    hyper_parameter.py   <-- HyperParameter(Variable): adds range specification
```

`Variable` is the base entity -- a named numeric value. `HyperParameter` extends `Variable` by adding a `range` field that can be either:
- `HyperParameterRange`: min/max/step for generating a continuous range
- `HyperParameterValues`: an explicit list of discrete values

This inheritance chain supports both simple named parameters and optimizable parameters with search spaces.

---

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class for `Variable` |
| `pydantic.Field` | Imported in `hyper_parameter.py` (though not actively used in visible code) |

Consumed by:
- `pyeventbt.__init__` (re-exports `HyperParameter` and `Variable`)
- Strategy optimization workflows (parameter sweep over `HyperParameter.range`)

---

## Gaps & Issues

1. **Empty `__init__.py` files**: Neither `core/__init__.py` nor `core/entities/__init__.py` re-export anything. Users must import from the full module path (e.g., `pyeventbt.core.entities.hyper_parameter.HyperParameter`) unless using the top-level `pyeventbt` re-exports.
2. **`Field` imported but unused**: `hyper_parameter.py` imports `Field` from Pydantic but does not use it in the visible code.
3. **No optimization engine visible**: `HyperParameter` defines parameter ranges but there is no optimization engine in the documented modules that consumes these ranges. The optimization loop may exist in undocumented modules or may not yet be implemented.
