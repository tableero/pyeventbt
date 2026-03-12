# Module: pyeventbt.config.configs

**File**: `pyeventbt/config/configs.py`
**Module**: `pyeventbt.config.configs`
**Purpose**: Defines the `Mt5PlatformConfig` model for MetaTrader 5 platform connection parameters.
**Tags**: `#config` `#mt5` `#pydantic` `#connection`

---

## Dependencies

| Import | Source |
|---|---|
| `BaseConfig` | `pyeventbt.config` (relative: `from . import BaseConfig`) |

---

## Classes

### `Mt5PlatformConfig`

```python
class Mt5PlatformConfig(BaseConfig)
```

**Description**: Pydantic model holding all parameters required to establish a connection to a MetaTrader 5 terminal. Used when calling `strategy.run_live()` to configure the live trading connection.

**Inherits from**: `BaseConfig` (which inherits from `pydantic.BaseModel`, providing YAML load/save)

**Fields**:

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `path` | `str` | Yes | -- | Path to the MT5 platform executable (e.g., `C:/Program Files/MT5/terminal64.exe`) |
| `login` | `int` | Yes | -- | Login ID for MT5 platform connection |
| `password` | `str` | Yes | -- | Password for MT5 platform connection |
| `server` | `str` | Yes | -- | Server name for MT5 platform connection |
| `timeout` | `int` | Yes | -- | Timeout in milliseconds for the connection |
| `portable` | `bool` | Yes | -- | Whether the MT5 installation is portable mode |

**Usage example** (from `example_ma_crossover.py`):

```python
mt5_config = Mt5PlatformConfig(
    path="C:\\Program Files\\MetaTrader 5\\terminal64.exe",
    login=1234,
    password="1234",
    server="Demo",
    timeout=60000,
    portable=False
)
strategy.run_live(mt5_configuration=mt5_config, ...)
```

---

## Data Flow

- **Inbound**: Instantiated by the user with MT5 connection credentials. Can also be loaded from YAML via inherited `BaseConfig.load_from_yaml()`.
- **Outbound**: Passed to `Strategy.run_live()` which forwards it to the execution engine and data provider connectors for establishing the MT5 connection.

---

## Gaps & Issues

1. **No default values**: All fields are required with no defaults. The CLAUDE.md documentation incorrectly states `timeout=60000` and `portable=False` as defaults, but the actual code requires all fields to be explicitly provided.
2. **Password stored in plaintext**: The `password` field is a plain `str`. There is no `SecretStr` usage (Pydantic's built-in secret type) to prevent accidental logging or serialization of credentials.
3. **No path validation**: The `path` field accepts any string. There is no validation that the path exists or points to a valid MT5 executable.
4. **Typo**: The `portable` field docstring reads "wether" instead of "whether".
5. **Platform-specific**: MT5 only runs on Windows. This config will be accepted on any platform but will fail at runtime on non-Windows systems. No early validation is performed at config creation time.

---

## Requirements Derived

- R-CFG-01: MT5 live trading must require explicit connection parameters: path, login, password, server, timeout, and portable flag.
- R-CFG-02: Configuration must be serializable to/from YAML for persistent storage (inherited from `BaseConfig`).
- R-CFG-03: Sensitive credentials (password) should be handled with care to avoid accidental exposure.
