# Package: pyeventbt.config

**Package**: `pyeventbt.config`
**Purpose**: Configuration models for the PyEventBT framework, primarily the MetaTrader 5 platform connection configuration and a base configuration class with YAML serialization support.
**Tags**: `#config` `#pydantic` `#yaml` `#mt5`

---

## Modules

| Module | Description |
|---|---|
| `__init__.py` | Re-exports `BaseConfig` from `core.entities.base_config` and wildcard-imports from `configs` |
| `configs.py` | Defines `Mt5PlatformConfig` for MT5 platform connection parameters |
| `core/__init__.py` | Empty package marker |
| `core/entities/__init__.py` | Empty package marker |
| `core/entities/base_config.py` | `BaseConfig` base class extending Pydantic `BaseModel` with YAML load/save |

---

## Internal Architecture

```
config/
  __init__.py         <-- Re-exports BaseConfig + wildcard from configs.py
  configs.py          <-- Mt5PlatformConfig(BaseConfig)
  core/
    __init__.py
    entities/
      __init__.py
      base_config.py  <-- BaseConfig(BaseModel) with YAML serialization
```

The `config` package follows the project-wide convention of `core/entities/` for base data models. `BaseConfig` provides YAML serialization that all configuration classes inherit. `Mt5PlatformConfig` is the only concrete configuration currently defined here.

The `__init__.py` uses a wildcard import (`from .configs import *`) to surface all public names from `configs.py` at the package level.

---

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pydantic.BaseModel` | Base class for `BaseConfig` |
| `yaml` (PyYAML) | YAML serialization in `BaseConfig.load_from_yaml()` and `save_to_yaml()` |

This package is imported by:
- `pyeventbt.__init__` (re-exports `Mt5PlatformConfig`)
- Strategy live trading configuration (passes `Mt5PlatformConfig` to `strategy.run_live()`)

---

## Gaps & Issues

1. **Wildcard import in `__init__.py`**: `from .configs import *` imports all public names from `configs.py`. Since `configs.py` does not define `__all__`, this relies on the default behavior of importing all names not starting with underscore. This could inadvertently export internal names.
2. **`BaseConfig` imported via relative package import**: In `configs.py`, `BaseConfig` is imported as `from . import BaseConfig`, which resolves through the `config/__init__.py` re-export chain. This creates a circular-like dependency path that works but is fragile.
3. **No default values for `timeout` and `portable`**: The CLAUDE.md describes `Mt5PlatformConfig` as having defaults (`timeout=60000`, `portable=False`), but the actual code does not set defaults -- all fields are required.
4. **Typo in docstring**: The `portable` field docstring says "wether" instead of "whether".
