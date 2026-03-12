# Module: pyeventbt.utils.utils

**File**: `pyeventbt/utils/utils.py`
**Module**: `pyeventbt.utils.utils`
**Purpose**: Provides utility functions for order type conversion, currency conversion, timeframe bar detection, date formatting, platform compatibility checks, and colored terminal logging.
**Tags**: `#utils` `#static-methods` `#mt5` `#currency` `#logging` `#formatting`

---

## Dependencies

| Import | Source |
|---|---|
| `pandas` (as `pd`) | Third-party |
| `datetime` | Standard library |
| `zoneinfo.ZoneInfo` | Standard library |
| `IDataProvider` | `pyeventbt.data_provider.core.interfaces.data_provider_interface` |
| `Decimal` | `decimal` (standard library) |
| `platform` | Standard library |
| `Enum` | `enum` (standard library, imported but unused) |
| `os` | Standard library (imported but unused) |
| `lru_cache` | `functools` (standard library) |
| `logging` | Standard library |

---

## Module-Level Objects

### `logger`

```python
logger = logging.getLogger("PyEventBT")
```

Module-level logger instance used by `check_platform_compatibility()`.

---

## Classes

### `TerminalColors`

```python
class TerminalColors
```

**Description**: Plain class holding ANSI escape code constants for terminal text coloring.

**Attributes**:

| Attribute | Value | Description |
|---|---|---|
| `HEADER` | `'\033[95m'` | Magenta |
| `OKBLUE` | `'\033[94m'` | Blue |
| `OKCYAN` | `'\033[96m'` | Cyan |
| `OKGREEN` | `'\033[92m'` | Green |
| `WARNING` | `'\033[93m'` | Yellow |
| `FAIL` | `'\033[91m'` | Red |
| `ENDC` | `'\033[0m'` | Reset |
| `BOLD` | `'\033[1m'` | Bold |
| `UNDERLINE` | `'\033[4m'` | Underline |

---

### `LoggerColorFormatter`

```python
class LoggerColorFormatter(logging.Formatter)
```

**Description**: Custom logging formatter that applies ANSI color codes based on log level. Each log level maps to a different color for visual distinction in terminal output.

**Class Attributes**:

| Attribute | Value | Maps to |
|---|---|---|
| `grey` | `"\x1b[38;20m"` | DEBUG |
| `green` | `"\x1b[92;20m"` | (defined but not used in FORMATS) |
| `cian` | `"\x1b[96;20m"` | INFO |
| `yellow` | `"\x1b[93;20m"` | WARNING |
| `red` | `"\x1b[91;1;4m"` | ERROR, CRITICAL |

**Format template**: `"%(asctime)s - %(levelname)s: %(message)s"`

#### Methods

##### `format`

```python
def format(self, record: logging.LogRecord) -> str
```

**Description**: Overrides `logging.Formatter.format()`. Selects a color-coded format string based on `record.levelno`, creates a new `logging.Formatter` with that string, and formats the record.

**Note**: This method shadows the class attribute `format` (the format string template). The class attribute is still accessible through the `FORMATS` dict which captured it at class definition time.

---

### `Utils`

```python
class Utils
```

**Description**: Collection of static utility methods used across the framework. The class has an empty `__init__` and all methods are `@staticmethod`.

#### Methods

##### `order_type_str_to_int`

```python
@staticmethod
@lru_cache(maxsize=10)
def order_type_str_to_int(order_type: str) -> int
```

**Description**: Converts an MT5 order type string to its integer constant.

**Mapping**:
| String | Int |
|---|---|
| `"BUY"` | 0 |
| `"SELL"` | 1 |
| `"BUY_LIMIT"` | 2 |
| `"SELL_LIMIT"` | 3 |
| `"BUY_STOP"` | 4 |
| `"SELL_STOP"` | 5 |
| `"BUY_STOP_LIMIT"` | 6 |
| `"SELL_STOP_LIMIT"` | 7 |
| `"CLOSE_BY"` | 8 |
| (unknown) | -1 |

**Returns**: `int`

---

##### `order_type_int_to_str`

```python
@staticmethod
@lru_cache(maxsize=10)
def order_type_int_to_str(order_type: int) -> str
```

**Description**: Converts an MT5 order type integer to its string representation. Inverse of `order_type_str_to_int`.

**Returns**: `str` (returns `"UNKNOWN"` for unrecognized values)

---

##### `check_new_m1_bar_creates_new_tf_bar`

```python
@staticmethod
def check_new_m1_bar_creates_new_tf_bar(latest_bar_time: pd.Timestamp, timeframe: str) -> bool
```

**Description**: Determines whether a new 1-minute bar completes a bar in a higher timeframe. Adds 1 minute to `latest_bar_time` and checks if the result is a multiple of the target timeframe's duration in seconds.

**Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `latest_bar_time` | `pd.Timestamp` | Timestamp of the latest M1 bar |
| `timeframe` | `str` | Target timeframe string |

**Supported timeframes**: `'1min'`, `'5min'`, `'15min'`, `'30min'`, `'1H'`, `'4H'`, `'1D'`, `'D'`, `'B'`

**Returns**: `bool`

**Raises**: `ValueError` if timeframe is not recognized.

---

##### `convert_currency_amount_to_another_currency`

```python
@staticmethod
def convert_currency_amount_to_another_currency(
    amount: Decimal, from_ccy: str, to_ccy: str, data_provider: IDataProvider
) -> Decimal
```

**Description**: Converts a monetary amount from one currency to another using live FX spot rates from the data provider. Looks up the appropriate FX pair from a hardcoded list and fetches the latest bid price.

**Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `amount` | `Decimal` | Amount to convert |
| `from_ccy` | `str` | Source currency code (e.g., `"EUR"`) |
| `to_ccy` | `str` | Target currency code (e.g., `"USD"`) |
| `data_provider` | `IDataProvider` | Data provider instance for fetching FX rates |

**Returns**: `Decimal`

**Raises**: `IndexError` if no FX pair is found for the given currency combination.

---

##### `get_currency_conversion_multiplier_cfd`

```python
@staticmethod
def get_currency_conversion_multiplier_cfd(
    from_ccy: str, to_ccy: str, data_provider: IDataProvider
) -> Decimal
```

**Description**: Returns the conversion multiplier between two currencies using CFD FX rates. Similar to `convert_currency_amount_to_another_currency` but returns the multiplier (rate) rather than a converted amount.

**Returns**: `Decimal` (multiplier; `Decimal(1)` if currencies are the same)

---

##### `get_fx_futures_suffix`

```python
@staticmethod
def get_fx_futures_suffix(symbol: str) -> tuple[str]
```

**Description**: Returns a tuple of two futures contract symbols (current and next) based on the current calendar month. Uses quarterly contract cycles: H (March), M (June), U (September), Z (December).

**Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `symbol` | `str` | Base futures symbol (e.g., `"6E"` for EUR) |

**Returns**: `tuple[str]` -- `(current_contract, next_contract)`, e.g., `("6E_H", "6E_M")`

---

##### `convert_currency_amount_to_another_currency_futures`

```python
@staticmethod
def convert_currency_amount_to_another_currency_futures(
    amount: Decimal, from_ccy: str, to_ccy: str, data_provider: IDataProvider
) -> Decimal
```

**Description**: Converts a monetary amount between currencies using futures contract prices instead of spot FX. Requires one of the currencies to be USD.

**Returns**: `Decimal`

**Raises**: `Exception` if neither currency is USD.

---

##### `dateprint`

```python
@staticmethod
def dateprint() -> str
```

**Description**: Returns the current date and time formatted as `"dd/mm/yyyy HH:MM:SS.sss"`. Uses `America/New_York` timezone with a +7 hour offset to approximate `Asia/Nicosia` with US DST rules.

**Returns**: `str`

---

##### `cap_forecast`

```python
@staticmethod
def cap_forecast(forecast: float) -> float
```

**Description**: Clamps a forecast value to the range `[-20.0, 20.0]`.

**Returns**: `float`

---

## Standalone Functions

### `colorize`

```python
def colorize(string: str, color: TerminalColors = TerminalColors.OKBLUE) -> str
```

**Description**: Wraps a string with ANSI color codes for terminal display.

**Returns**: `str`

---

### `check_platform_compatibility`

```python
def check_platform_compatibility(raise_exception: bool = True) -> bool
```

**Description**: Checks if the current platform is Windows, which is required for MT5 live trading. On non-Windows platforms, either raises an exception or logs a warning.

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `raise_exception` | `bool` | `True` | If `True`, raises `Exception` on incompatible platform; if `False`, logs warning and returns `False` |

**Returns**: `bool` (`True` if Windows)

---

### `print_percentage_bar`

```python
def print_percentage_bar(percentage: float, bar_length: int = 50, additional_message: str = '', end: str = '\r') -> None
```

**Description**: Prints a text-based progress bar to stdout. Uses block characters for the filled portion and dashes for empty.

**Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `percentage` | `float` | Required | Progress percentage (0-100) |
| `bar_length` | `int` | `50` | Character width of the bar |
| `additional_message` | `str` | `''` | Extra text shown after the percentage |
| `end` | `str` | `'\r'` | Line ending character (carriage return for in-place updates) |

**Raises**: `ValueError` if percentage is outside 0-100.

---

## Data Flow

- **Inbound**: Called by various framework modules needing order type conversion, currency rates, timeframe checks, or logging setup.
- **Outbound**: Currency conversion methods call `data_provider.DATA_PROVIDER.get_latest_bid()` and `get_latest_tick()` to fetch live market prices. `dateprint()` accesses system clock. `print_percentage_bar` writes to stdout.

---

## Gaps & Issues

1. **Unused imports**: `Enum`, `os` are imported but never used.
2. **Spanish comments**: Multiple inline comments are in Spanish, reducing accessibility for non-Spanish-speaking developers.
3. **Hardcoded FX pair list**: The `all_fx_symbol` tuple in currency conversion methods is hardcoded and duplicated across `convert_currency_amount_to_another_currency` and `get_currency_conversion_multiplier_cfd`. Any new pair must be added in multiple places.
4. **`IndexError` on missing pair**: If no FX pair matches the given currencies, the list comprehension `[...][0]` raises an unhandled `IndexError` rather than a descriptive error.
5. **Timezone approximation**: `dateprint()` approximates `Asia/Nicosia` time by adding 7 hours to `America/New_York`. This is fragile and will be incorrect during periods when US and Cypriot DST transitions do not align.
6. **`format` attribute/method shadowing**: `LoggerColorFormatter` has both a class attribute `format` (string) and a method `format()`. The method shadows the attribute name.
7. **`green` color unused**: `LoggerColorFormatter.green` is defined but not mapped to any log level in `FORMATS`.
8. **Timestamp unit access**: `check_new_m1_bar_creates_new_tf_bar` accesses `pd.Timestamp._value` and `.unit`, which are private/internal pandas attributes and may break across pandas versions.
9. **`Utils` class is unnecessary**: All methods are static with no shared state. These could be standalone module-level functions instead of methods on a class with an empty `__init__`.
10. **`cap_forecast` lacks documentation**: No docstring explaining the significance of the -20/+20 range.

---

## Requirements Derived

- R-UTL-01: Order type mappings must cover all MT5 order types (BUY, SELL, LIMIT, STOP, STOP_LIMIT, CLOSE_BY).
- R-UTL-02: Currency conversion must support major FX pairs and use live market data for rate lookup.
- R-UTL-03: Futures currency conversion must support quarterly contract roll logic (H, M, U, Z cycles).
- R-UTL-04: Platform compatibility checks must prevent MT5-dependent operations from running on non-Windows systems.
- R-UTL-05: Logging must support color-coded output by log level for terminal readability.
- R-UTL-06: Timeframe bar detection must correctly identify when a new higher-timeframe bar has completed based on M1 bar timestamps.
