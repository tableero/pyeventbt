# pyeventbt.strategy.services

**Package**: `pyeventbt.strategy.services`

**Purpose**: Contains service classes that provide auxiliary functionality to the strategy package. Currently holds only the `ParameterStore` service for managing user-defined parameters.

**Tags**: `#services` `#parameter-management` `#storage`

---

## Modules

| Module | File | Description |
|---|---|---|
| `pyeventbt.strategy.services.parameter_store` | `parameter_store.py` | `ParameterStore` class for storing and retrieving named parameters. |
| `pyeventbt.strategy.services.__init__` | `__init__.py` | Package initializer. Empty beyond the license header. |

---

## Internal Architecture

This sub-package is minimal, containing a single service class. The `ParameterStore` provides a simple key-value store for named parameters. It is not currently wired into the `Strategy` class or the `Modules` container, suggesting it may be used independently or is part of an incomplete feature.

```
strategy.services
  |
  +-- parameter_store.py  (ParameterStore: key-value parameter storage)
```

---

## Cross-Package Dependencies

| Module | External Dependencies |
|---|---|
| `parameter_store.py` | `typing.Dict`, `typing.Any` (stdlib only) |

---

## Gaps & Issues

1. **`ParameterStore` not integrated**: The class is not referenced in `strategy.py`, `Modules`, or any other module in the strategy package. Its intended use is unclear.
2. **Class-level mutable state bug**: `ParameterStore.__paramters` is a class-level `dict`, meaning all instances share the same parameter storage. This is almost certainly a bug -- it should be an instance-level attribute initialized in `__init__`.
3. **`__init__.py` exports nothing**: No public API surface from this sub-package.
