# File: `pyeventbt/broker/mt5_broker/core/entities/init_credentials.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.init_credentials`

## Purpose
Defines the `InitCredentials` Pydantic model for storing MetaTrader 5 platform initialization credentials. Used during the `initialize()` and `login()` calls to validate and persist credential data in `SharedData`.

## Tags
`entity`, `pydantic`, `credentials`, `authentication`, `mt5`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |

## Classes/Functions

### `class InitCredentials(BaseModel)`

Pydantic model representing MT5 platform initialization credentials.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `path` | `str` | Path to the MetaTrader 5 terminal executable |
| `login` | `int` | Trading account login ID |
| `password` | `str` | Trading account password |
| `server` | `str` | Trade server name |
| `timeout` | `int` | Connection timeout in milliseconds |
| `portable` | `bool` | Whether to run terminal in portable mode |

## Data Flow

```
Mt5SimulatorWrapper.initialize(path, login, password, server, timeout, portable)
    |
    v
PlatformConnector.initialize()
    |
    v
InitCredentials(**creds)  -- Pydantic validation
    |
    v
SharedData.credentials = InitCredentials(...)
SharedData.account_info.login = credentials.login
SharedData.account_info.server = credentials.server
```

The primary purpose of this model is validation: if the credential dict cannot be parsed into a valid `InitCredentials` object, the `initialize()` call returns `False` with an auth failure error code.

## Gaps & Issues

1. **`password` stored in plain text** -- The password is stored as a plain `str` field with no encryption or masking. If `SharedData.credentials` is logged or serialized, the password is exposed.
2. **`server` type inconsistency** -- In `InitCredentials`, `server` is `str`. In `Mt5SimulatorWrapper.initialize()`, the `server` parameter type hint is `int`, though it is passed through as-is. The real MT5 API accepts `str` for server.
3. **No validation rules** -- No constraints on `login > 0`, `timeout > 0`, or `path` being a valid file path.

## Requirements Derived

- **REQ-ENTITY-CRED-001**: Credentials must be validated before being stored in `SharedData` to ensure platform initialization integrity.
- **REQ-ENTITY-CRED-002**: The `login` and `server` fields from credentials must propagate to `AccountInfo` after successful initialization.
