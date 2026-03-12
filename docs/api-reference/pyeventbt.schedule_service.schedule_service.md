# File: `pyeventbt/schedule_service/schedule_service.py`

## Module

`pyeventbt.schedule_service.schedule_service`

## Purpose

Implements time-based callback scheduling for the PyEventBT event loop. Monitors bar timestamps and fires user-registered callbacks when a configured timeframe boundary is crossed.

## Tags

`scheduling`, `callbacks`, `timeframe`, `periodic-execution`, `pydantic`

## Dependencies

| Dependency | Import |
|---|---|
| `typing` | `Callable`, `Dict`, `List` |
| `pyeventbt.events.events` | `BarEvent`, `ScheduledEvent` |
| `pyeventbt.strategy.core.modules` | `Modules` |
| `pyeventbt.strategy.core.strategy_timeframes` | `StrategyTimeframes` |
| `pydantic` | `BaseModel` |
| `pandas` | `pd` (for `pd.Timestamp`) |

## Classes/Functions

### `Schedule(BaseModel)`

```python
class Schedule(BaseModel):
    name: str
    is_active: bool = True
    fn: Callable
    execute_every: StrategyTimeframes
```

**Description**: Represents a single registered scheduled callback. Equality is based on `name` only.

**Attributes**:

| Attribute | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | -- | Identifier for the schedule (set to `repr(callback)` during registration) |
| `is_active` | `bool` | `True` | Whether this schedule is currently active |
| `fn` | `Callable` | -- | The callback function to execute |
| `execute_every` | `StrategyTimeframes` | -- | The timeframe interval at which this schedule fires |

**Methods**:

#### `__eq__(value: object) -> bool`

Compares schedules by `name` only. Does not check `fn`, `is_active`, or `execute_every`.

---

### `Schedules`

```python
class Schedules:
    def __init__(self) -> None
```

**Description**: Internal registry managing a dictionary of schedules grouped by timeframe.

**Attributes**:

| Attribute | Type | Visibility | Description |
|---|---|---|---|
| `__schedules` | `Dict[StrategyTimeframes, List[Schedule]]` | Private | Map of timeframe to list of schedules |

**Methods**:

#### `add_schedule(timeframe: StrategyTimeframes, callback: Callable[[ScheduledEvent, Modules], None]) -> Schedule`

Creates a `Schedule` with `name=repr(callback)`, appends to the timeframe's list, and returns it.

#### `activate_schedule(schedule: Schedule) -> None`

Finds a schedule by equality (`name`) within its timeframe group and sets `is_active = True`.

#### `deactivate_schedule(schedule: Schedule) -> None`

Finds a schedule by equality (`name`) within its timeframe group and sets `is_active = False`.

#### `deactivate_all_schedules() -> None`

Sets `is_active = False` on every schedule across all timeframes.

#### `activate_all_schedules() -> None`

Sets `is_active = True` on every schedule across all timeframes.

#### `remove_schedule(schedule: Schedule) -> None`

Attempts to remove a schedule. **Bug**: Uses `self.__schedules.pop(schedule, None)` where `schedule` is a `Schedule` object but the dict is keyed by `StrategyTimeframes`. This will never match and always print the warning.

#### `remove_inactive_schedules() -> None`

Intended to filter out inactive schedules. **Bug**: Filters on `key.is_active` where `key` is a `StrategyTimeframes` enum, which has no `is_active` attribute. Would raise `AttributeError`.

#### `get_callbacks_to_execute_given_timeframe(timeframe: StrategyTimeframes) -> List[Callable[[ScheduledEvent, Modules], None]]`

Returns the `fn` of all active schedules for the given timeframe.

---

### `TimeframeWatchInfo(BaseModel)`

```python
class TimeframeWatchInfo(BaseModel):
    last_timestamp: pd.Timestamp = None
    current_timestamp: pd.Timestamp = None
```

**Description**: Tracks the last and current timestamps for a watched timeframe. Used to determine if enough time has elapsed to trigger scheduled callbacks.

**Attributes**:

| Attribute | Type | Default | Description |
|---|---|---|---|
| `last_timestamp` | `pd.Timestamp` | `None` | Timestamp of the last callback execution for this timeframe |
| `current_timestamp` | `pd.Timestamp` | `None` | Timestamp of the most recent bar processed |

**Config**: `arbitrary_types_allowed = True` (needed for `pd.Timestamp`).

**Methods**:

#### `__eq__(value: object) -> bool`

Compares both `last_timestamp` and `current_timestamp`. **Bug**: For non-`TimeframeWatchInfo` comparisons, executes `ValueError(...)` without `raise`, so it returns `None` instead of raising an error.

---

### `ScheduleService`

```python
class ScheduleService:
    def __init__(self, modules: Modules) -> None
```

**Description**: Top-level service that manages schedule registration, timeframe monitoring, and callback execution. Created by `TradingDirector` and called on every bar event.

**Attributes**:

| Attribute | Type | Visibility | Description |
|---|---|---|---|
| `__modules` | `Modules` | Private | Dependency injection container, passed to callbacks |
| `__schedules` | `Schedules` | Private | Internal schedule registry |
| `__timeframes_to_watch` | `Dict[StrategyTimeframes, TimeframeWatchInfo]` | Private | Tracks timestamps per watched timeframe |
| `__last_callback_args` | `Dict[str, ScheduledEvent]` | Private | Declared but **never used** in current code |

**Methods**:

#### `__get_timeframes_to_trigger(event: BarEvent) -> List[StrategyTimeframes]`

**Description**: Determines which timeframes should fire based on the current bar's timestamp.

**Algorithm**:
1. If the event has no valid `datetime`, return empty list
2. For each watched timeframe:
   - If `current_timestamp` is `None` (first bar), initialize both timestamps and skip
   - Otherwise, update `current_timestamp` to the bar's datetime
   - If `current_timestamp - last_timestamp >= timeframe.to_timedelta()`, add to trigger list
3. Return list of timeframes to trigger

**Returns**: `List[StrategyTimeframes]` -- timeframes whose callbacks should fire.

---

#### `add_schedule(timeframe: StrategyTimeframes, callback: Callable[[ScheduledEvent, Modules], None]) -> Schedule`

Registers a callback for periodic execution at the given timeframe interval. Also adds the timeframe to the watch list if not already present.

**Returns**: `Schedule` -- the created schedule object.

---

#### `deactivate_schedules() -> None`

Deactivates all schedules by delegating to `Schedules.deactivate_all_schedules()`.

---

#### `activate_schedules() -> None`

Activates all schedules by delegating to `Schedules.activate_all_schedules()`.

---

#### `run_scheduled_callbacks(event: BarEvent) -> None`

**Description**: Main entry point called by `TradingDirector` on every bar event. Checks which timeframes should trigger and executes their active callbacks.

**Algorithm**:
1. Call `__get_timeframes_to_trigger(event)` to get triggered timeframes
2. For each triggered timeframe:
   - Get the `TimeframeWatchInfo` for this timeframe
   - Get all active callbacks via `get_callbacks_to_execute_given_timeframe`
   - For each callback:
     - Create a `ScheduledEvent` with the timeframe, symbol, current timestamp, and former timestamp
     - Call `callback(scheduled_event, self.__modules)`
     - Update `last_timestamp = current_timestamp`

## Data Flow

```
TradingDirector._handle_bar_event(bar_event)
    |
    +--> ScheduleService.run_scheduled_callbacks(bar_event)
            |
            +--> __get_timeframes_to_trigger(bar_event)
            |       |
            |       +--> For each watched timeframe:
            |               Compare (current_timestamp - last_timestamp) >= timeframe.to_timedelta()
            |               Return list of triggered timeframes
            |
            +--> For each triggered timeframe:
                    |
                    +--> Schedules.get_callbacks_to_execute_given_timeframe(timeframe)
                    |       |
                    |       +--> Returns [fn for active schedules matching timeframe]
                    |
                    +--> For each callback fn:
                            |
                            +--> Create ScheduledEvent(timeframe, symbol, timestamps)
                            +--> fn(scheduled_event, modules)
                            +--> Update last_timestamp
```

## Gaps & Issues

1. **`remove_schedule` is broken**: Uses `dict.pop(schedule)` with a `Schedule` key on a dict keyed by `StrategyTimeframes`. Will never find a match.
2. **`remove_inactive_schedules` is broken**: Accesses `key.is_active` on `StrategyTimeframes` keys, which do not have that attribute.
3. **`__last_callback_args` is unused**: Initialized in `__init__` but never referenced elsewhere. Dead code.
4. **`TimeframeWatchInfo.__eq__` missing `raise`**: The `ValueError(...)` on line 95 is not raised, silently returning `None` for non-matching types.
5. **Timestamp update happens per-callback, not per-timeframe**: If multiple callbacks are registered for the same timeframe, `last_timestamp` is updated after each callback. This means the second callback gets a different `former_execution_timestamp` than the first if timestamps change. In practice this is benign since `last_timestamp` is set to the same `current_timestamp` each time.
6. **First bar is always skipped**: When `current_timestamp is None`, both timestamps are initialized and the timeframe is not added to the trigger list. This means callbacks never fire on the first bar, even if they should.
7. **No error handling**: If a callback raises an exception, it propagates unhandled and could crash the event loop.
8. **Schedule names use `repr(callback)`**: This could produce non-deterministic names for lambdas or closures, making `activate_schedule`/`deactivate_schedule` unreliable for such callbacks.

## Requirements Derived

| ID | Requirement | Source |
|---|---|---|
| SS-01 | Users must be able to register callbacks that fire at regular timeframe intervals | `add_schedule` method |
| SS-02 | Scheduling must be driven by bar timestamps, not wall-clock time | `__get_timeframes_to_trigger` logic |
| SS-03 | Schedules must be individually or globally activate-able and deactivate-able | `activate_schedule`, `deactivate_schedule`, `activate_all_schedules`, `deactivate_all_schedules` |
| SS-04 | Each callback invocation must receive a `ScheduledEvent` with current and former timestamps plus a `Modules` instance | `run_scheduled_callbacks` implementation |
| SS-05 | The first bar must initialize timestamp tracking without triggering callbacks | First-bar `None` check in `__get_timeframes_to_trigger` |
| SS-06 | Schedules can be disabled entirely when `run_schedules=False` in `TradingDirector` | `deactivate_schedules()` call in `_run_backtest`/`_run_live_trading` |
