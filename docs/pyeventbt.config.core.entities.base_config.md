# Module: pyeventbt.config.core.entities.base_config

**File**: `pyeventbt/config/core/entities/base_config.py`
**Module**: `pyeventbt.config.core.entities.base_config`
**Purpose**: Provides the `BaseConfig` base class for all configuration models in the framework, adding YAML serialization/deserialization on top of Pydantic's `BaseModel`.
**Tags**: `#config` `#base-class` `#pydantic` `#yaml` `#serialization`

---

## Dependencies

| Import | Source |
|---|---|
| `BaseModel` | `pydantic` |
| `yaml` | `PyYAML` |

---

## Classes

### `BaseConfig`

```python
class BaseConfig(BaseModel)
```

**Description**: Abstract-like base class for all PyEventBT configuration models. Extends Pydantic's `BaseModel` with YAML file serialization capabilities. All configuration classes in the framework (e.g., `Mt5PlatformConfig`) inherit from this class.

**Inherits from**: `pydantic.BaseModel`

---

#### Methods

##### `load_from_yaml`

```python
@classmethod
def load_from_yaml(cls, file_path: str = 'config.yaml') -> Self
```

**Description**: Class method that loads configuration from a YAML file. Reads the file, parses it with `yaml.safe_load()`, and passes the resulting dictionary as keyword arguments to the class constructor.

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file_path` | `str` | `'config.yaml'` | Path to the YAML configuration file |

**Returns**: Instance of the calling class (`cls`)

**Raises**:
- `FileNotFoundError` if the file does not exist
- `yaml.YAMLError` if the file contains invalid YAML
- `pydantic.ValidationError` if the parsed data does not match the model's field definitions

---

##### `save_to_yaml`

```python
def save_to_yaml(self, file_path: str = 'config.yaml') -> None
```

**Description**: Instance method that serializes the configuration to a YAML file. Uses `self.model_dump()` to convert the Pydantic model to a dictionary, then writes it with `yaml.dump()`.

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file_path` | `str` | `'config.yaml'` | Path to write the YAML configuration file |

**Returns**: `None`

---

## Data Flow

- **Inbound**: YAML files on disk are read and parsed into configuration model instances via `load_from_yaml()`.
- **Outbound**: Configuration instances can be persisted to YAML files via `save_to_yaml()`. The Pydantic `model_dump()` method converts the model to a plain dictionary before YAML serialization.

---

## Gaps & Issues

1. **No `arbitrary_types_allowed` configuration**: The CLAUDE.md states `BaseConfig` has `class Config: arbitrary_types_allowed = True`, but the actual code does not include this. This means subclasses cannot use non-standard types (e.g., `Decimal`, custom objects) as fields without adding their own Pydantic config.
2. **Default file path is relative**: Both `load_from_yaml()` and `save_to_yaml()` default to `'config.yaml'` which is relative to the current working directory. This could lead to unexpected behavior depending on where the script is run from.
3. **No error handling**: Neither method wraps file I/O in try/except blocks. Errors propagate directly to the caller.
4. **`yaml.dump` defaults**: The `save_to_yaml()` method uses default `yaml.dump()` settings, which may produce less readable output (e.g., no explicit `default_flow_style=False`).
5. **Not truly abstract**: Despite serving as a base class, `BaseConfig` can be instantiated directly (it has no abstract methods or fields). This is acceptable but means it is an empty model if instantiated alone.

---

## Requirements Derived

- R-BCFG-01: All configuration models must support YAML-based persistence for load and save operations.
- R-BCFG-02: Configuration loading must leverage Pydantic validation to ensure type correctness.
- R-BCFG-03: YAML serialization must produce a dictionary representation compatible with re-loading via the same class.
