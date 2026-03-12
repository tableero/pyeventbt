# pyeventbt.strategy.core.walk_forward

**File**: `pyeventbt/strategy/core/walk_forward.py`

**Module**: `pyeventbt.strategy.core.walk_forward`

**Purpose**: Defines data structures for walk-forward optimization results, including the `WalkforwardType` enum (anchored vs. unanchored) and the `WalkForwardResults` Pydantic model for storing and serializing optimization output.

**Tags**: `#walk-forward` `#optimization` `#pydantic` `#data-model` `#serialization`

---

## Dependencies

- `enum.Enum` (stdlib)
- `pydantic.BaseModel`, `pydantic.ConfigDict`, `pydantic.field_validator`
- `pandas` (`pd.Timestamp`, `pd.DataFrame`, `pd.read_csv`)
- `typing.Dict`, `typing.List`
- `pyeventbt.backtest.core.backtest_results.BacktestResults`

---

## Classes

### `WalkforwardType(str, Enum)`

Enum defining the two walk-forward optimization modes.

| Member | Value | Description |
|---|---|---|
| `ANCHORED` | `'ANCHORED'` | Training window start is fixed; window grows over time. |
| `UNANCHORED` | `'UNANCHORED'` | Training window slides forward; window size remains constant. |

---

### `WalkForwardResults(BaseModel)`

Pydantic model encapsulating the output of a walk-forward optimization run. Supports CSV serialization/deserialization.

#### Model Config

```python
model_config = ConfigDict(arbitrary_types_allowed=True)
```

Required for `pd.DataFrame` and `pd.Timestamp` fields.

#### Fields

| Field | Type | Description |
|---|---|---|
| `backtest_results` | `BacktestResults` | The combined backtest results from all walk-forward windows. |
| `retrainting_timestamps` | `List[pd.Timestamp]` | Timestamps at which the model was retrained. **Note**: field name contains a typo ("retrainting" instead of "retraining"). |
| `hyperparameters_track` | `pd.DataFrame` | DataFrame tracking hyperparameter values across retraining windows. |

#### Field Validators

##### `transform_timstamps(cls, raw: List[str]) -> List[pd.Timestamp]`

- **Decorator**: `@field_validator("retrainting_timestamps", mode="before")`
- **Behavior**: Converts a list of strings to `pd.Timestamp` objects. Runs before standard Pydantic validation.
- **Note**: Validator name also contains a typo ("timstamps" instead of "timestamps").

##### `transform(cls, raw: pd.DataFrame | List[Dict[str, int | float]]) -> pd.DataFrame`

- **Decorator**: `@field_validator("hyperparameters_track", mode="before")`
- **Behavior**: If `raw` is already a `pd.DataFrame`, returns it unchanged. If `raw` is a list of dicts, converts to `pd.DataFrame`. Does not handle other types (implicitly returns `None`).

#### Methods

##### `to_csv(self, path: str) -> None`

Saves all three components to separate CSV files in the specified directory.

| Output File | Content |
|---|---|
| `{path}/backtest_results.csv` | Serialized backtest results via `BacktestResults.save()`. |
| `{path}/retrainting_timestamps.csv` | Timestamps as a DataFrame CSV. |
| `{path}/hyperparameters_track.csv` | Hyperparameter tracking DataFrame CSV. |

**Note**: Uses string concatenation for path joining (`path + "/filename"`) rather than `os.path.join()`.

---

##### `from_csv(path: str) -> WalkForwardResults` (static method)

Loads walk-forward results from CSV files in the specified directory.

**Parameters**:
| Name | Type | Description |
|---|---|---|
| `path` | `str` | Directory containing the three CSV files. |

**Returns**: `WalkForwardResults` instance.

**Behavior**:
1. Loads backtest results via `BacktestResults.load()`.
2. Reads timestamps CSV with `pd.read_csv()`, extracting the second column via `lambda x: x[1]`.
3. Reads hyperparameters CSV with `pd.read_csv()`.

---

## Data Flow

- **Input**: Walk-forward optimization process produces `BacktestResults`, retraining timestamps, and hyperparameter tracking data.
- **Output**: Packaged into `WalkForwardResults` for programmatic access. Can be serialized to/from CSV for persistence.

```
Optimization loop
  |
  +---> BacktestResults (from combined windows)
  +---> List[pd.Timestamp] (retraining points)
  +---> pd.DataFrame (hyperparameter history)
  |
  v
WalkForwardResults
  |
  +---> to_csv(path) ---> 3 CSV files
  +---> from_csv(path) <--- 3 CSV files
```

---

## Gaps & Issues

1. **Typo in field name**: `retrainting_timestamps` should be `retraining_timestamps`. This is a public API surface and would be a breaking change to fix.
2. **Typo in validator name**: `transform_timstamps` should be `transform_timestamps`. This is internal but affects readability.
3. **`transform` validator returns `None` for unhandled types**: If `raw` is neither a `DataFrame` nor a `list`, the validator implicitly returns `None`, which will likely cause a downstream Pydantic validation error without a clear message.
4. **Path construction via string concatenation**: `to_csv()` and `from_csv()` use `path + "/filename"` instead of `os.path.join()`, which would fail on Windows with backslash paths.
5. **`from_csv` timestamp extraction is fragile**: Uses `lambda x: x[1]` to extract the second column, assuming the CSV index is in column 0 and timestamps in column 1. If the CSV format changes, this breaks silently.
6. **Walk-forward optimization not yet integrated**: The `WalkforwardType` enum is defined but not referenced anywhere in `strategy.py`. The optimization imports in `strategy.py` are commented out, suggesting this feature is incomplete.

---

## Requirements Derived

1. **REQ-WF-001**: The system shall support anchored and unanchored walk-forward optimization modes.
2. **REQ-WF-002**: Walk-forward results shall be serializable to and deserializable from CSV files.
3. **REQ-WF-003**: Walk-forward results shall track retraining timestamps and hyperparameter values across optimization windows.
4. **REQ-WF-004**: The `hyperparameters_track` field shall accept both `pd.DataFrame` and `List[Dict]` inputs, converting the latter automatically.
