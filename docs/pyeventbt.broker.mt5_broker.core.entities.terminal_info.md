# File: `pyeventbt/broker/mt5_broker/core/entities/terminal_info.py`

## Module
`pyeventbt.broker.mt5_broker.core.entities.terminal_info`

## Purpose
Defines the `TerminalInfo` Pydantic model representing the state and properties of a MetaTrader 5 terminal. Mirrors the structure returned by `mt5.terminal_info()`. Includes connectivity status, permissions, build information, and file system paths.

## Tags
`entity`, `pydantic`, `terminal`, `mt5`, `connection`, `configuration`

## Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `pydantic.BaseModel` | External | Base class for validated data model |
| `decimal.Decimal` | Stdlib | Precision type for balance/retransmission fields |

## Classes/Functions

### `class TerminalInfo(BaseModel)`

Pydantic model representing MT5 terminal information.

**Attributes**:

| Attribute | Type | Description |
|---|---|---|
| `community_account` | `bool` | Whether MQL5.community account is authorized |
| `community_connection` | `bool` | Whether connection to MQL5.community is established |
| `connected` | `bool` | Whether terminal is connected to the trade server |
| `dlls_allowed` | `bool` | Whether DLL calls are allowed |
| `trade_allowed` | `bool` | Whether trading is allowed (algorithmic trading permission) |
| `tradeapi_disabled` | `bool` | Whether the trading API is disabled |
| `email_enabled` | `bool` | Whether email notifications are enabled |
| `ftp_enabled` | `bool` | Whether FTP publishing is enabled |
| `notifications_enabled` | `bool` | Whether push notifications are enabled |
| `mqid` | `bool` | Whether MetaQuotes ID is set for notifications |
| `build` | `int` | Terminal build number |
| `maxbars` | `int` | Maximum number of bars in chart |
| `codepage` | `int` | Terminal code page for string encoding |
| `ping_last` | `int` | Last ping to trade server in microseconds |
| `community_balance` | `Decimal` | MQL5.community account balance |
| `retransmission` | `Decimal` | Network retransmission percentage |
| `company` | `str` | Broker company name |
| `name` | `str` | Terminal name |
| `language` | `str` | Terminal language |
| `path` | `str` | Terminal installation path |
| `data_path` | `str` | Terminal data directory path |
| `commondata_path` | `str` | Common data directory path |

## Data Flow

```
YAML (default_terminal_info.yaml)
    |
    v
SharedData._load_default_terminal_info()
    |
    v
TerminalInfo(**yaml_data) -> SharedData.terminal_info
    |
    v
PlatformConnector.initialize() sets connected=True
PlatformConnector.shutdown() sets connected=False
    |
    v
TerminalInfoConnector.terminal_info() returns SharedData.terminal_info
LiveMT5Broker._check_algo_trading_enabled() checks trade_allowed
```

## Gaps & Issues

1. **`connected` is mutable** -- The `connected` field is directly mutated by `PlatformConnector.initialize()` and `shutdown()`. Requires Pydantic model mutability configuration.
2. **`build` used for version tuple** -- `PlatformConnector.version()` reads `terminal_info.build` to construct the version tuple, coupling version reporting to terminal info state.

## Requirements Derived

- **REQ-ENTITY-TERM-001**: `trade_allowed` must be checkable to determine if algorithmic trading is permitted before placing orders.
- **REQ-ENTITY-TERM-002**: `connected` must reflect the current connection state and be updated by platform lifecycle methods.
