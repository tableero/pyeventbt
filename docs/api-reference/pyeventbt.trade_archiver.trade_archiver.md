# pyeventbt.trade_archiver.trade_archiver

- **File**: `pyeventbt/trade_archiver/trade_archiver.py`
- **Module**: `pyeventbt.trade_archiver.trade_archiver`
- **Purpose**: Implements trade archival -- stores FillEvents with incrementing IDs and provides export to pandas DataFrame, JSON string, Parquet file, and CSV file.
- **Tags**: `trade-archiver`, `fill-events`, `export`, `pandas`, `polars`, `parquet`, `json`, `csv`

## Dependencies

| Dependency | Usage |
|---|---|
| `.core.interfaces.trade_archiver_interface.ITradeArchiver` | Interface/protocol implemented by this class |
| `pyeventbt.events.events.FillEvent` | Event type stored in the archive |
| `pandas` (`pd`) | DataFrame construction for historical trades export and CSV export |
| `polars` (`pl`) | DataFrame construction with explicit schema for Parquet export |
| `decimal.Decimal` | Quantization of financial values (price, commission, profit) during serialization |
| `os` | `os.makedirs` for ensuring export directories exist |
| `json` | `json.dumps` for JSON serialization |
| `logging` | Logger for error/debug messages |

## Classes/Functions

### TradeArchiver

- **Signature**: `class TradeArchiver(ITradeArchiver)`
- **Description**: Stores `FillEvent` objects in an ordered dictionary keyed by incrementing integer IDs. Provides multiple export formats for trade history analysis.

#### Attributes

| Attribute | Type | Description |
|---|---|---|
| `trade_archive` | `dict[int, FillEvent]` | Dictionary mapping trade IDs to FillEvent objects |
| `trade_archive_id` | `int` | Auto-incrementing counter for trade IDs (starts at 0, first trade gets ID 1) |

#### Methods

| Method | Signature | Description | Returns |
|---|---|---|---|
| `archive_trade` | `archive_trade(fill_event: FillEvent) -> None` | Increments `trade_archive_id` and stores the FillEvent. | `None` |
| `get_trade_archive` | `get_trade_archive() -> dict[int, FillEvent]` | Returns the full trade archive dictionary. | `dict[int, FillEvent]` |
| `export_historical_trades_dataframe` | `export_historical_trades_dataframe() -> pd.DataFrame` | Converts the archive to a pandas DataFrame with columns: TYPE, DEAL, SYMBOL, TIME_GENERATED, POSITION_ID, STRATEGY_ID, EXCHANGE, VOLUME, PRICE, SIGNAL_TYPE, COMMISSION, SWAP, FEE, GROSS_PROFIT, CCY. Enum fields are exported as `.value`. | `pd.DataFrame` |
| `export_historical_trades_json` | `export_historical_trades_json() -> str` | Serializes the archive to a JSON string. Financial values are quantized via `Decimal.quantize()` and converted to strings. Datetime is formatted as `%Y-%m-%dT%H:%M:%S`. Returns empty dict on serialization error. | `str` (JSON) |
| `export_historical_trades_parquet` | `export_historical_trades_parquet(file_path: str) -> None` | Exports trades to a Parquet file via polars with zstd compression (level 10). Financial Decimal values are converted to `float` after quantization. Creates parent directories if needed. Logs warning if no trades. Re-raises exceptions after logging. | `None` |
| `export_csv_trade_archive` | `export_csv_trade_archive(file_path: str) -> None` | Exports trades to CSV via `export_historical_trades_dataframe().to_csv()`. Creates parent directories if needed. | `None` |

#### Parquet Schema

The Parquet export enforces this polars schema:

| Column | Polars Type |
|---|---|
| TYPE, DEAL, SYMBOL, EXCHANGE, SIGNAL_TYPE, CCY | `pl.Utf8` |
| TIME_GENERATED | `pl.Datetime` |
| POSITION_ID, STRATEGY_ID | `pl.Int64` |
| VOLUME, PRICE, COMMISSION, SWAP, FEE, GROSS_PROFIT | `pl.Float64` |

## Data Flow

1. During backtest/live execution, the `ExecutionEngine` emits `FillEvent`s.
2. `PortfolioHandler.process_fill_event()` calls `TradeArchiver.archive_trade()` with each FillEvent.
3. Each FillEvent is stored in `trade_archive` with an incrementing integer key.
4. After the backtest/session completes, the user or framework calls an export method:
   - `export_historical_trades_dataframe()` for in-memory pandas analysis.
   - `export_historical_trades_json()` for JSON serialization.
   - `export_historical_trades_parquet(path)` for compressed on-disk storage.
   - `export_csv_trade_archive(path)` for CSV file output.

## Gaps & Issues

1. **FIXME comment (line 65)**: Comment reads "This is deprecated in new versions of pandas" referencing the row accumulation pattern. The actual code now uses list-based accumulation (which is the correct modern approach), but the FIXME comment was not cleaned up.
2. **Return type annotation mismatch on `export_historical_trades_json`**: The docstring says "Returns a dictionary structure" but the return type is `str` (JSON string). On error, it returns `{}` (empty dict) instead of `""` or `"{}"`, mixing return types.
3. **Interface incompleteness**: Only `archive_trade`, `get_trade_archive`, and `export_csv_trade_archive` are declared in `ITradeArchiver`. The DataFrame, JSON, and Parquet export methods are not part of the interface contract.
4. **No thread safety**: The incrementing ID and dict mutation are not synchronized, which could be an issue if archive_trade were called from multiple threads.
5. **Decimal precision inconsistency**: JSON export uses `Decimal('0.00001')` (5 decimal places) while Parquet export uses `Decimal('0.000001')` (6 decimal places) for price, commission, and gross_profit.
6. **No deletion or filtering**: No method to remove trades or query the archive by symbol, date range, or other criteria.

## Requirements Derived

- R-TA-01: Every FillEvent must be archived with a unique incrementing integer ID.
- R-TA-02: Export to pandas DataFrame must include all FillEvent fields with enum values resolved.
- R-TA-03: Parquet export must use zstd compression with an explicit schema.
- R-TA-04: Export methods must create parent directories if they do not exist.
- R-TA-05: JSON export must handle serialization errors gracefully with logging.
