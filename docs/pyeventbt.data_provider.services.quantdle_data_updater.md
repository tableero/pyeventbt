# File: `pyeventbt/data_provider/services/quantdle_data_updater.py`

## Module

`pyeventbt.data_provider.services.quantdle_data_updater`

## Purpose

Standalone utility for downloading historical OHLCV bar data from the Quantdle API and maintaining a local CSV cache. Minimizes API calls by detecting existing CSV files, identifying date range gaps, and only downloading missing data. The resulting CSV files are compatible with `CSVDataProvider` for backtesting.

## Tags

`quantdle`, `data-download`, `csv-cache`, `historical-data`, `utility`, `api-client`

## Dependencies

| Dependency | Usage |
|---|---|
| `quantdle` | Conditionally imported at init time; `qdl.Client` for API access |
| `polars` | CSV reading, DataFrame manipulation, writing |
| `logging` | `pyeventbt` logger |
| `pathlib.Path` | CSV directory and file handling |
| `datetime.datetime` | Date range parameters |

## Classes/Functions

### `QuantdleDataUpdater`

**Signature:**
```python
class QuantdleDataUpdater:
    def __init__(self, api_key: str, api_key_id: str)
```

**Instance Attributes:**

| Attribute | Type | Description |
|---|---|---|
| `qdl` | `module` | The `quantdle` module reference |
| `client` | `quantdle.Client` | Authenticated Quantdle API client |

#### Methods

---

##### `__init__(self, api_key: str, api_key_id: str)`

Imports `quantdle` (raises `ImportError` with install instructions if missing). Creates an authenticated `quantdle.Client`.

---

##### `update_data(self, csv_dir, symbols, start_date, end_date, timeframe="1min", spread_column="spreadopen") -> None`

Main entry point. For each symbol:
- If a CSV file exists, calls `_update_existing_csv` to fill date range gaps.
- If no CSV exists, calls `_create_new_csv` to download the full range.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `csv_dir` | `str` | -- | Directory for CSV files |
| `symbols` | `list[str]` | -- | Symbols to download (e.g., `["EURUSD", "GBPUSD"]`) |
| `start_date` | `datetime` | -- | Requested data range start |
| `end_date` | `datetime` | -- | Requested data range end |
| `timeframe` | `str` | `"1min"` | Bar timeframe |
| `spread_column` | `str` | `"spreadopen"` | Which Quantdle spread column to use |

---

##### `_update_symbol_data(self, csv_path, symbol, start_date, end_date, timeframe, spread_column) -> None`

Dispatches to `_update_existing_csv` or `_create_new_csv` based on whether the CSV file exists.

---

##### `_update_existing_csv(self, csv_file, symbol, start_date, end_date, timeframe, spread_column) -> None`

Reads the existing CSV (headerless, 9 columns matching `CSVDataProvider` format). Parses the datetime range. Downloads data for any gaps:
- **Before existing range:** `start_date` to `csv_start`
- **After existing range:** `csv_end` to `end_date`

Concatenates all DataFrames, removes duplicates by datetime, sorts, drops the datetime column, and overwrites the CSV.

---

##### `_create_new_csv(self, csv_file, symbol, start_date, end_date, timeframe, spread_column) -> None`

Downloads the full requested range from Quantdle and writes a new CSV file (no header).

---

##### `_download_from_quantdle(self, symbol, start_date, end_date, timeframe, spread_column) -> pl.DataFrame | None`

Core download method:
1. Converts timeframe to Quantdle format via `_convert_to_quantdle_timeframe`.
2. Calls `self.client.download_data(symbol=[symbol], timeframe=..., start_date=..., end_date=..., output_format="polars")`.
3. Detects and normalizes the datetime column name.
4. Casts price columns to Float64, volume columns to Int64.
5. Maps the selected spread column (e.g., `"spreadopen"`) to `"spread"` and drops unused spread columns.
6. Creates `date` and `time` columns in MT5 format (`YYYY.MM.DD`, `HH:MM:SS`).
7. Returns a DataFrame with columns: `[date, time, open, high, low, close, tickvol, volume, spread, datetime]`.

**Returns:** `pl.DataFrame` or `None` on error.

---

##### `_convert_to_quantdle_timeframe(self, timeframe: str) -> str`

Maps PyEventBT timeframe strings to Quantdle format.

| Input | Output |
|---|---|
| `"1min"` | `"M1"` |
| `"5min"` | `"M5"` |
| `"15min"` | `"M15"` |
| `"30min"` | `"M30"` |
| `"1h"` / `"1H"` | `"H1"` |
| `"4h"` / `"4H"` | `"H4"` |
| `"1d"` / `"1D"` | `"D1"` |
| `"1w"` / `"1W"` | `"W1"` |

Falls back to returning the input unchanged if not found in the mapping.

## Data Flow

```
User code:
    updater = QuantdleDataUpdater(api_key="...", api_key_id="...")
    updater.update_data(csv_dir="./data", symbols=["EURUSD"], ...)

update_data()
    --> for each symbol:
        _update_symbol_data()
            --> CSV exists? _update_existing_csv()
            |     --> read existing CSV with Polars
            |     --> identify gaps (before/after existing range)
            |     --> _download_from_quantdle() for each gap
            |     --> concat, deduplicate, sort, write CSV
            |
            --> CSV missing? _create_new_csv()
                  --> _download_from_quantdle(full range)
                  --> write CSV

_download_from_quantdle()
    --> _convert_to_quantdle_timeframe()
    --> quantdle.Client.download_data(output_format="polars")
    --> normalize columns (datetime, prices, spread)
    --> add date/time columns in MT5 format
    --> return pl.DataFrame
```

## Gaps & Issues

1. **No gap detection within existing data range.** The updater only checks for missing data *before* the earliest and *after* the latest existing timestamp. If there are gaps in the middle of the existing CSV (e.g., missing days), they are not detected or filled.
2. **`_download_from_quantdle` returns a DataFrame with a `datetime` column** that is dropped before CSV writing, but `_update_existing_csv` parses the existing CSV and adds its own `datetime` column for deduplication. The temporary `datetime` column flows through concatenation correctly but the approach is fragile.
3. **No retry logic for API failures.** If the Quantdle download fails, the error is logged and `None` is returned, silently skipping the data.
4. **Unicode characters in log output.** Uses checkmark and cross symbols that may not render in all terminal encodings.
5. **`_convert_to_quantdle_timeframe` silently passes through unrecognized timeframes** via `dict.get(timeframe, timeframe)`. This could lead to confusing API errors from Quantdle.
6. **`_update_existing_csv` overwrites the CSV in place** without backup. If the process fails mid-write, data could be lost.
7. **Spread column handling assumes Quantdle returns specific column names** (`"spreadmax"`, `"spreadopen"`). If Quantdle changes its schema, the code would fail silently or error.

## Requirements Derived

- **REQ-DP-QDL-001:** Add intra-range gap detection to fill missing data within existing CSV files.
- **REQ-DP-QDL-002:** Add retry logic with exponential backoff for API download failures.
- **REQ-DP-QDL-003:** Validate the timeframe in `_convert_to_quantdle_timeframe` and raise a clear error for unsupported values.
- **REQ-DP-QDL-004:** Create a backup of the existing CSV before overwriting during update operations.
- **REQ-DP-QDL-005:** Add schema validation for Quantdle API responses (expected columns, types).
- **REQ-DP-QDL-006:** Replace Unicode symbols in log messages with ASCII alternatives for cross-platform compatibility.
