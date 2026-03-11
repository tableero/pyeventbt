# File: `pyeventbt/data_provider/connectors/mt5_live_data_connector.py`

## Module

`pyeventbt.data_provider.connectors.mt5_live_data_connector`

## Purpose

Implements the live MT5 terminal data provider. Polls the MetaTrader 5 API for real-time bar and tick data, converting raw MT5 records into `BarEvent` objects with integer-scaled prices. This connector is used when the strategy is running in live or paper-trading mode against a connected MT5 terminal.

## Tags

`connector`, `mt5`, `live`, `real-time`, `polling`, `metatrader5`, `windows-only`

## Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events.events` | `BarEvent`, `Bar` dataclass |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface` | `IDataProvider` base |
| `pyeventbt.data_provider.core.configurations.data_provider_configurations` | `MT5LiveDataConfig` |
| `pyeventbt.utils.utils` | `check_platform_compatibility` for Windows check before MT5 import |
| `MetaTrader5` | Conditionally imported; `mt5.copy_rates_from_pos`, `mt5.symbol_info`, `mt5.symbol_info_tick` |
| `polars` | DataFrame construction for `get_latest_bars` |
| `pandas` | Legacy `get_latest_bar_old`, `get_latest_bars_old_pandas` return types |
| `decimal.Decimal` | Bid/ask/price values in tick dict |

## Classes/Functions

### Conditional MT5 Import (module level)

```python
try:
    if check_platform_compatibility(raise_exception=False):
        import MetaTrader5 as mt5
    else:
        mt5 = None
except ImportError:
    mt5 = None
```

Gracefully handles non-Windows platforms and missing MT5 installations by setting `mt5 = None`.

---

### `Mt5LiveDataProvider(IDataProvider)`

**Signature:**
```python
class Mt5LiveDataProvider(IDataProvider):
    def __init__(self, configs: MT5LiveDataConfig)
```

**Key Instance Attributes:**

| Attribute | Type | Description |
|---|---|---|
| `symbol_list` | `list[str]` | Trading symbols from config |
| `timeframes_list` | `list[str]` | Timeframes to poll |
| `last_bar_datetime` | `dict[str, datetime]` | Last seen bar time per symbol (legacy, unused by current code) |
| `last_bar_tf_datetime` | `dict[str, dict[str, datetime]]` | Last seen bar time per `{symbol: {timeframe: datetime}}` |
| `futures_tuple` | `tuple[str, ...]` | Tuple of 32 major futures contract base names (currently unused; futures handling is commented out) |

#### Methods

---

##### `_map_timeframe(self, timeframe: str) -> int`

Maps string timeframe to MT5 constant (e.g., `"1min"` -> `mt5.TIMEFRAME_M1`). Supports all standard MT5 timeframes from M1 through MN1, with both upper and lowercase hour variants (`"1h"` and `"1H"`).

**Bug:** The `try/except KeyError` wraps the dictionary *construction*, not the key lookup. The `return timeframe_mapping[timeframe]` on line 99 is in the `else` block, so an invalid timeframe key would raise an unhandled `KeyError`.

---

##### `get_latest_bar(self, symbol: str, timeframe: str = "1min") -> BarEvent | None`

Fetches the last closed bar (position=1) from MT5 via `mt5.copy_rates_from_pos`. Scales prices to integers using `symbol_info().digits`. Falls back to heuristic digits (3 for JPY pairs, 5 otherwise) if `symbol_info` returns `None`.

**Returns:** `BarEvent` on success, `None` on any error.

---

##### `get_latest_bars(self, symbol: str, timeframe: str = "1min", N: int = 2) -> pl.DataFrame | None`

Fetches N bars from position 1 via MT5 API. Converts MT5 time (seconds since epoch) to Polars Datetime. Renames `tick_volume` to `tickvol` and `real_volume` to `volume`.

**Returns:** `pl.DataFrame` with columns `[datetime, open, high, low, close, tickvol, volume, spread]`, or `None` on error.

---

##### `get_latest_tick(self, symbol: str) -> dict`

Calls `mt5.symbol_info_tick(symbol)`. Returns a dictionary with `Decimal`-typed bid/ask/last/volume_real and integer time/time_msc/flags/volume.

**Returns:** `dict` or empty `dict` on failure.

---

##### `get_latest_bid(self, symbol: str) -> Decimal`

Delegates to `get_latest_tick(symbol)["bid"]`.

---

##### `get_latest_ask(self, symbol: str) -> Decimal`

Delegates to `get_latest_tick(symbol)["ask"]`.

---

##### `get_latest_datetime(self, symbol: str, timeframe: str = "1min") -> datetime`

Returns `get_latest_bar(symbol, timeframe).datetime`.

---

##### `update_bars(self) -> list[BarEvent]`

Polls all symbol x timeframe combinations. For each:
1. Calls `get_latest_bar(symbol, timeframe)`.
2. If `None`, skips.
3. If bar datetime is newer than `last_bar_tf_datetime[symbol][timeframe]`, updates tracking and calls `get_latest_bar` **again** to append the event.

**Returns:** `list[BarEvent]` (may be empty if no new bars).

---

##### Deprecated Methods (still present)

| Method | Notes |
|---|---|
| `get_latest_bar_old(symbol, timeframe)` | Returns `pd.Series` instead of `BarEvent`; uses pandas DataFrame |
| `get_latest_bars_old_pandas(symbol, timeframe, N)` | Returns `pd.DataFrame` instead of `pl.DataFrame` |

## Data Flow

```
update_bars() [called by event loop periodically]
  --> for each symbol in symbol_list:
      --> for each timeframe in timeframes_list:
          --> get_latest_bar(symbol, timeframe)
              --> mt5.copy_rates_from_pos(symbol, tf, pos=1, count=1)
              --> scale prices to int, create Bar + BarEvent
          --> if bar.datetime > last_seen:
              --> update last_bar_tf_datetime
              --> get_latest_bar(symbol, timeframe)  [DUPLICATE CALL]
              --> append to events list
  --> return events list
```

## Gaps & Issues

1. **`update_bars` calls `get_latest_bar` twice per new bar** -- once to check the datetime and once to build the event. The first call's result (`latest_bar`) could be reused directly.
2. **`_map_timeframe` has unreachable `except KeyError`** -- the dictionary is constructed inside `try`, but an invalid key would fail on `return timeframe_mapping[timeframe]` in the `else` block.
3. **`last_bar_datetime` attribute is unused.** It is initialized in `__init__` but never read or updated.
4. **Futures contract handling is entirely commented out** (~20 lines in `update_bars`). The `futures_tuple` attribute is initialized but serves no purpose.
5. **No rate limiting or backoff** on MT5 API polling. Rapid `update_bars` calls could overwhelm the terminal.
6. **`get_latest_bar` uses a hardcoded fallback** (`3` digits for JPY, `5` otherwise) when `symbol_info` is unavailable, which may produce incorrect scaling for exotic instruments.
7. **No `continue_backtest` or `close_positions_end_of_data` attributes** -- these are expected by the service layer for backtest mode but are meaningless for live. However, the service layer accesses them unconditionally when `trading_context == "BACKTEST"`.
8. **Two deprecated pandas-based methods** remain without deprecation markers.

## Requirements Derived

- **REQ-DP-MT5-001:** Reuse the first `get_latest_bar` call result in `update_bars` instead of calling the API twice.
- **REQ-DP-MT5-002:** Fix `_map_timeframe` error handling so invalid timeframes produce a clear error message.
- **REQ-DP-MT5-003:** Remove unused `last_bar_datetime` and `futures_tuple` attributes, or implement futures contract logic.
- **REQ-DP-MT5-004:** Remove or deprecate `get_latest_bar_old` and `get_latest_bars_old_pandas`.
- **REQ-DP-MT5-005:** Consider adding polling interval / rate limiting to prevent excessive MT5 API calls.
- **REQ-DP-MT5-006:** Add `continue_backtest` and `close_positions_end_of_data` stub attributes (or make the service layer check context before accessing them).
