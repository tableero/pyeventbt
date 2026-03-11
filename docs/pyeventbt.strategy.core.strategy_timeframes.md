# pyeventbt.strategy.core.strategy_timeframes

**File**: `pyeventbt/strategy/core/strategy_timeframes.py`

**Module**: `pyeventbt.strategy.core.strategy_timeframes`

**Purpose**: Defines the `StrategyTimeframes` enum representing all supported bar timeframes (from 1 minute to 1 year), with methods for conversion to `timedelta` and custom comparison operators for sorting.

**Tags**: `#enum` `#timeframes` `#configuration` `#data-model`

---

## Dependencies

- `enum.Enum` (stdlib)
- `datetime.timedelta` (stdlib)

---

## Classes

### `StrategyTimeframes(str, Enum)`

String enum whose values correspond to pandas-compatible timeframe codes. Supports conversion to `timedelta` and custom comparison/hashing.

#### Members

| Member | Value | Timedelta Equivalent |
|---|---|---|
| `ONE_MIN` | `'1min'` | 1 minute |
| `TWO_MIN` | `'2min'` | 2 minutes |
| `THREE_MIN` | `'3min'` | 3 minutes |
| `FOUR_MIN` | `'4min'` | 4 minutes |
| `FIVE_MIN` | `'5min'` | 5 minutes |
| `SIX_MIN` | `'6min'` | 6 minutes |
| `TEN_MIN` | `'10min'` | 10 minutes |
| `TWELVE_MIN` | `'12min'` | 12 minutes |
| `FIFTEEN_MIN` | `'15min'` | 15 minutes |
| `TWENTY_MIN` | `'20min'` | 20 minutes |
| `THIRTY_MIN` | `'30min'` | 30 minutes |
| `ONE_HOUR` | `'1h'` | 1 hour |
| `TWO_HOUR` | `'2h'` | 2 hours |
| `THREE_HOUR` | `'3h'` | 3 hours |
| `FOUR_HOUR` | `'4h'` | 4 hours |
| `SIX_HOUR` | `'6h'` | 6 hours |
| `EIGHT_HOUR` | `'8h'` | 8 hours |
| `TWELVE_HOUR` | `'12h'` | 12 hours |
| `ONE_DAY` | `'1D'` | 1 day |
| `ONE_WEEK` | `'1W'` | 1 week (7 days) |
| `ONE_MONTH` | `'1M'` | 30 days (approximation) |
| `SIX_MONTH` | `'6M'` | 180 days (approximation) |
| `ONE_YEAR` | `'12M'` | 365 days (approximation) |

#### Methods

##### `to_timedelta(self) -> timedelta`

Converts the timeframe to a `datetime.timedelta` object using a hard-coded lookup dictionary.

**Returns**: `timedelta`

**Notes**: Month and year values are approximations (30 days/month, 365 days/year). The dictionary is reconstructed on every call (not cached).

---

##### `__eq__(self, value: object) -> bool`

Custom equality comparison.

- If `value` is a `str`: compares against `self.value` (the string code).
- If `value` is a `StrategyTimeframes`: compares using `to_timedelta()`.
- Otherwise: raises `ValueError`.

**Note**: This override breaks the default `Enum.__eq__` behavior where identity-based comparison is used. Two members with different string values but equal timedeltas would compare as equal (though no such case exists in the current member list).

---

##### `__gt__(self, value: str) -> bool`

Greater-than comparison using `to_timedelta()`.

**Note**: The type hint says `str` but the implementation calls `value.to_timedelta()`, which only works if `value` is a `StrategyTimeframes` instance.

---

##### `__lt__(self, value: str) -> bool`

Less-than comparison using `to_timedelta()`.

**Note**: Same type hint mismatch as `__gt__`.

---

##### `__hash__(self) -> int`

Returns `hash(self.value)` -- hashes based on the string value.

**Note**: Required because overriding `__eq__` makes the class unhashable by default in Python 3. This ensures `StrategyTimeframes` members can be used as dictionary keys and in sets.

---

## Data Flow

- **Input**: Enum members are referenced by user code and the `Strategy` class when specifying which timeframes a strategy operates on.
- **Output**: Used by `DataProvider` to configure bar aggregation, by `ScheduleService` to determine firing intervals, and by `Strategy` for sorting the timeframe list.

---

## Gaps & Issues

1. **Month/year approximations**: `ONE_MONTH` = 30 days, `SIX_MONTH` = 180 days, `ONE_YEAR` = 365 days. For backtests spanning multiple years, accumulated drift could cause misalignment with calendar boundaries.
2. **Type hint mismatch on `__gt__` and `__lt__`**: Hints declare `value: str` but the body calls `value.to_timedelta()`, which requires `StrategyTimeframes`. Passing an actual `str` would raise `AttributeError`.
3. **`to_timedelta` lookup dictionary rebuilt on every call**: The mapping dictionary is created inside the method body each time. This is a minor performance concern for hot paths.
4. **`__eq__` raises `ValueError` for non-str/non-StrategyTimeframes**: Comparing with other types (e.g., `int`, `None`) raises instead of returning `NotImplemented`, which breaks Python comparison protocol conventions.
5. **Missing `__ge__` and `__le__`**: Only `__gt__` and `__lt__` are defined. `>=` and `<=` comparisons will fall back to default behavior, which may not use timedelta-based ordering.

---

## Requirements Derived

1. **REQ-TF-001**: The system shall support 22 distinct bar timeframes from 1 minute to 1 year.
2. **REQ-TF-002**: Timeframe values shall be compatible with pandas frequency codes for bar aggregation.
3. **REQ-TF-003**: Timeframes shall be orderable (sortable) by duration for determining base timeframe and scheduling intervals.
4. **REQ-TF-004**: Timeframes shall be usable as dictionary keys (hashable).
