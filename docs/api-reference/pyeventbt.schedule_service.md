# Package: `pyeventbt.schedule_service`

## Purpose

Provides time-based callback scheduling within the PyEventBT event loop. Allows users to register functions that execute at regular intervals (e.g., every hour, every day) based on bar timestamps rather than wall-clock time.

## Tags

`scheduling`, `callbacks`, `timeframe`, `periodic-execution`

## Modules

| Module | File | Description |
|---|---|---|
| `schedule_service` | `schedule_service.py` | Contains `Schedule`, `Schedules`, `TimeframeWatchInfo`, and `ScheduleService` classes |
| `__init__` | `__init__.py` | Empty init file |

## Internal Architecture

```
ScheduleService
    |
    +-- __modules: Modules                                  (dependency injection)
    +-- __schedules: Schedules                              (schedule registry)
    +-- __timeframes_to_watch: Dict[StrategyTimeframes, TimeframeWatchInfo]
    +-- __last_callback_args: Dict[str, ScheduledEvent]     (unused in current code)
    |
    +-- add_schedule(timeframe, callback)
    +-- run_scheduled_callbacks(event: BarEvent)
    |       |
    |       +--> __get_timeframes_to_trigger(event)
    |       |       checks if time delta >= timeframe.to_timedelta()
    |       |
    |       +--> for each triggered timeframe:
    |               get active callbacks from Schedules
    |               call each with ScheduledEvent + Modules
    |               update last_timestamp
    |
    +-- activate_schedules() / deactivate_schedules()

Schedules (internal registry)
    |
    +-- __schedules: Dict[StrategyTimeframes, List[Schedule]]
    +-- add/activate/deactivate/remove schedule operations
    +-- get_callbacks_to_execute_given_timeframe(tf) --> List[Callable]

Schedule (Pydantic model)
    name: str (repr of callback)
    is_active: bool
    fn: Callable
    execute_every: StrategyTimeframes

TimeframeWatchInfo (Pydantic model)
    last_timestamp: pd.Timestamp
    current_timestamp: pd.Timestamp
```

### Trigger Mechanism

The schedule system is **not** driven by a timer or clock. Instead, on every `BarEvent`, `ScheduleService.run_scheduled_callbacks` is called. It compares the bar's timestamp against the last-seen timestamp for each watched timeframe. If the delta exceeds the timeframe's duration, the callbacks fire.

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events.events` | `BarEvent`, `ScheduledEvent` |
| `pyeventbt.strategy.core.modules` | `Modules` (passed to callbacks) |
| `pyeventbt.strategy.core.strategy_timeframes` | `StrategyTimeframes` (timeframe enum with `to_timedelta()`) |
| `pydantic` | `BaseModel` for `Schedule` and `TimeframeWatchInfo` |
| `pandas` | `pd.Timestamp` for timestamp tracking |

**Dependents**:

| Package | Usage |
|---|---|
| `pyeventbt.trading_director.trading_director` | Creates `ScheduleService`, calls `run_scheduled_callbacks` on each bar, registers schedules via `add_schedule` |

## Gaps & Issues

1. **`__last_callback_args` is declared but never used**: The `Dict[str, ScheduledEvent]` attribute in `ScheduleService.__init__` is initialized but never read or written beyond initialization.
2. **`remove_schedule` in `Schedules` has a bug**: It calls `self.__schedules.pop(schedule, None)` where `schedule` is a `Schedule` object, but `__schedules` is keyed by `StrategyTimeframes`, not `Schedule`. This will never match.
3. **`remove_inactive_schedules` has a bug**: It filters on `key.is_active` where `key` is a `StrategyTimeframes` enum (which has no `is_active` attribute). This would raise an `AttributeError`.
4. **`TimeframeWatchInfo.__eq__` raises `ValueError` incorrectly**: The method does `ValueError(...)` without `raise`, so comparisons with non-`TimeframeWatchInfo` types return `None` (falsy) instead of raising.
5. **Callbacks fire directly, not through the event queue**: `ScheduledEvent` objects are created and passed directly to callbacks rather than being placed on the event queue. This is inconsistent with the event-driven architecture.
6. **No deduplication of schedules**: The same callback can be registered multiple times for the same timeframe.
