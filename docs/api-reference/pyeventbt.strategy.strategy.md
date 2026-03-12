# pyeventbt.strategy.strategy

**File**: `pyeventbt/strategy/strategy.py`

**Module**: `pyeventbt.strategy.strategy`

**Purpose**: Provides the `Strategy` class -- the primary user-facing facade for configuring signal/sizing/risk engines, registering scheduled callbacks and hooks, and launching backtests or live trading sessions.

**Tags**: `#facade` `#user-api` `#backtest` `#live-trading` `#decorator-pattern` `#orchestration`

---

## Dependencies

### Standard Library
- `os`
- `queue.Queue`
- `typing.Callable`
- `datetime.datetime`, `datetime.timedelta`
- `functools.partial`
- `logging`

### Third-Party
- `pandas` (imported as `pd`)

### Internal (pyeventbt)
- `pyeventbt.backtest.core.backtest_results.BacktestResults`
- `pyeventbt.core.entities.hyper_parameter.HyperParameter`
- `pyeventbt.hooks.hook_service.HookService`, `Hooks`
- `pyeventbt.strategy.core.walk_forward.WalkForwardResults`, `WalkforwardType`
- `pyeventbt.trading_context.trading_context.TypeContext`
- `pyeventbt.trading_director.trading_director.TradingDirector`
- `pyeventbt.signal_engine.core.configurations.signal_engine_configurations.MACrossoverConfig`
- `pyeventbt.signal_engine.core.interfaces.signal_engine_interface.ISignalEngine`
- `pyeventbt.strategy.core.modules.Modules`
- `pyeventbt.strategy.core.strategy_timeframes.StrategyTimeframes`
- `pyeventbt.data_provider.core.configurations.data_provider_configurations.CSVBacktestDataConfig`, `MT5LiveDataConfig`
- `pyeventbt.data_provider.services.data_provider_service.DataProvider`
- `pyeventbt.events.events.BarEvent`, `ScheduledEvent`, `SignalEvent`
- `pyeventbt.execution_engine.core.configurations.execution_engine_configurations.MT5LiveExecutionConfig`, `MT5SimulatedExecutionConfig`
- `pyeventbt.execution_engine.services.execution_engine_service.ExecutionEngine`
- `pyeventbt.portfolio.portfolio.Portfolio`
- `pyeventbt.portfolio_handler.core.entities.suggested_order.SuggestedOrder`
- `pyeventbt.sizing_engine.core.configurations.sizing_engine_configurations.MinSizingConfig`, `RiskPctSizingConfig`, `FixedSizingConfig`
- `pyeventbt.risk_engine.core.configurations.risk_engine_configurations.PassthroughRiskConfig`
- `pyeventbt.trading_director.core.configurations.trading_session_configurations.MT5BacktestSessionConfig`, `MT5LiveSessionConfig`
- `pyeventbt.signal_engine.services.signal_engine_service.SignalEngineService`
- `pyeventbt.sizing_engine.services.sizing_engine_service.SizingEngineService`
- `pyeventbt.portfolio_handler.portfolio_handler.PortfolioHandler`
- `pyeventbt.risk_engine.services.risk_engine_service.RiskEngineService`
- `pyeventbt.config.Mt5PlatformConfig`
- `pyeventbt.utils.utils.LoggerColorFormatter`, `TerminalColors`, `colorize`
- `.core.account_currencies.AccountCurrencies`
- `.core.verbose_level.VerboseLevel`

### Commented-Out Imports
- `pyeventbt.optimization.cost_functions.cagr_dd_ratio_cost_function`
- `uuid`
- `hyperopt.fmin`, `hp`, `tpe`
- `hyperopt.exceptions.AllTrialsFailed`

---

## Module-Level Objects

### `logger`
- **Type**: `logging.Logger`
- **Name**: `"pyeventbt"`
- **Notes**: `propagate` set to `False`. Handler and formatter are added in `Strategy.__init__`.

---

## Classes

### `Strategy`

The main user-facing class. Users instantiate it, register engines/hooks/schedules via decorators, and call `.backtest()` or `.run_live()`.

#### `__init__(self, logging_level: VerboseLevel = VerboseLevel.INFO) -> None`

Sets up the `pyeventbt` logger with a `LoggerColorFormatter` console handler at the specified level. Calls `__initial_config()` to initialize internal state.

**Parameters**:
| Name | Type | Default | Description |
|---|---|---|---|
| `logging_level` | `VerboseLevel` | `VerboseLevel.INFO` | Logging verbosity level (maps to standard logging integers). |

---

#### `__initial_config(self) -> None`

Initializes all internal state:

| Attribute | Type | Initial Value | Description |
|---|---|---|---|
| `__sizing_engine_config` | `None \| MinSizingConfig \| RiskPctSizingConfig \| FixedSizingConfig` | `None` | Pre-defined sizing engine config. |
| `__signal_engine_config` | `None \| MACrossoverConfig` | `None` | Pre-defined signal engine config. |
| `__risk_engine_config` | `PassthroughRiskConfig` | `PassthroughRiskConfig()` | Pre-defined risk engine config. Defaults to passthrough (no filtering). |
| `__signal_engines` | `dict[str, ISignalEngine]` | `{}` | Custom signal engine functions keyed by `strategy_id`. |
| `__sizing_engines` | `dict[str, SizingEngineService]` | `{}` | Custom sizing engine functions keyed by `strategy_id`. |
| `__risk_engines` | `dict[str, RiskEngineService]` | `{}` | Custom risk engine functions keyed by `strategy_id`. |
| `__strategy_id_mg_number_map` | `dict[str, int]` | `{}` | Mapping of strategy IDs to MT5 magic numbers. |
| `__strategy_timeframes` | `list` | `[]` | Accumulated list of timeframes from engine/schedule registrations. |
| `__run_schedules` | `bool` | `True` | Whether scheduled tasks are active. |
| `__hooks` | `HookService` | `HookService()` | Lifecycle hook manager. |
| `__scheduled_events` | `dict[StrategyTimeframes, list[Callable]]` | `{}` | Scheduled callback functions grouped by timeframe interval. |

---

#### `hook(self, hook: Hooks) -> decorator`

Decorator that registers a callback to execute at a specific lifecycle hook point.

**Parameters**:
| Name | Type | Description |
|---|---|---|
| `hook` | `Hooks` | The lifecycle hook to attach to (e.g., `Hooks.ON_START`). |

**Decorated function signature**: `fn(modules: Modules) -> None`

**Returns**: Decorator function (does not return the wrapped function).

---

#### `enable_hooks(self) -> None`

Enables all registered hooks via `HookService.enable_hooks()`.

#### `disable_hooks(self) -> None`

Disables all registered hooks via `HookService.disable_hooks()`.

---

#### `custom_signal_engine(self, strategy_id: str = 'default', strategy_timeframes: list[StrategyTimeframes] = [StrategyTimeframes.ONE_MIN]) -> decorator`

Decorator that registers a user-defined signal engine function.

**Parameters**:
| Name | Type | Default | Description |
|---|---|---|---|
| `strategy_id` | `str` | `'default'` | Identifies which strategy this engine belongs to. |
| `strategy_timeframes` | `list[StrategyTimeframes]` | `[StrategyTimeframes.ONE_MIN]` | Timeframes at which the signal engine will be invoked. |

**Decorated function signature**: `fn(bar_event: BarEvent, modules: Modules) -> SignalEvent | list[SignalEvent] | None`

**Behavior**: Appends new timeframes to `__strategy_timeframes`. Stores the function in `__signal_engines` using `dict.setdefault()` -- first registration for a `strategy_id` wins.

---

#### `custom_sizing_engine(self, strategy_id: str = 'default') -> decorator`

Decorator that registers a user-defined sizing engine function.

**Parameters**:
| Name | Type | Default | Description |
|---|---|---|---|
| `strategy_id` | `str` | `'default'` | Identifies which strategy this engine belongs to. |

**Decorated function signature**: `fn(signal_event: SignalEvent, modules: Modules) -> SuggestedOrder | list[SuggestedOrder] | None`

**Behavior**: Stores the function in `__sizing_engines` using `dict.setdefault()`.

---

#### `custom_risk_engine(self, strategy_id: str = 'default') -> decorator`

Decorator that registers a user-defined risk engine function.

**Parameters**:
| Name | Type | Default | Description |
|---|---|---|---|
| `strategy_id` | `str` | `'default'` | Identifies which strategy this engine belongs to. |

**Decorated function signature**: `fn(suggested_order: SuggestedOrder, modules: Modules) -> float`

**Behavior**: Stores the function in `__risk_engines` using `dict.setdefault()`.

---

#### `configure_predefined_signal_engine(self, conf: MACrossoverConfig, strategy_timeframes: list[StrategyTimeframes] = [StrategyTimeframes.ONE_MIN]) -> None`

Sets a pre-defined signal engine configuration (currently only `MACrossoverConfig`).

**Parameters**:
| Name | Type | Default | Description |
|---|---|---|---|
| `conf` | `MACrossoverConfig` | -- | The signal engine configuration. |
| `strategy_timeframes` | `list[StrategyTimeframes]` | `[StrategyTimeframes.ONE_MIN]` | Timeframes for the signal engine. |

---

#### `configure_predefined_sizing_engine(self, conf: MinSizingConfig | RiskPctSizingConfig | FixedSizingConfig) -> None`

Sets a pre-defined sizing engine configuration.

---

#### `configure_predefined_risk_engine(self, conf: PassthroughRiskConfig) -> None`

Sets a pre-defined risk engine configuration.

---

#### `__get_signal_engine(self, strategy_id: str, modules: Modules) -> SignalEngineService`

Creates a `SignalEngineService` instance. If a custom signal engine function was registered for the given `strategy_id`, injects it via `set_signal_engine()`.

---

#### `__get_sizing_engine(self, strategy_id: str, modules: Modules) -> SizingEngineService`

Creates a `SizingEngineService` instance. If a custom sizing engine function was registered, injects it via `set_suggested_order_function()`.

---

#### `__get_risk_engine(self, strategy_id: str, modules: Modules) -> RiskEngineService`

Creates a `RiskEngineService` instance. If a custom risk engine function was registered, injects it via `set_custom_asses_order()`.

---

#### `__create_mg_for_strategy_id(self, strategy_id: str) -> int`

Creates an auto-incrementing magic number for a strategy ID. Uses `dict.setdefault()` to assign `max(existing) + 1`.

**Note**: Currently unused -- the call in `backtest()` is commented out.

---

#### `run_every(self, interval: StrategyTimeframes) -> decorator`

Decorator that registers a scheduled callback to run at a given timeframe interval.

**Parameters**:
| Name | Type | Description |
|---|---|---|
| `interval` | `StrategyTimeframes` | The interval at which the callback should fire. |

**Decorated function signature**: `fn(scheduled_event: ScheduledEvent, modules: Modules) -> None`

**Behavior**: Appends `interval` to `__strategy_timeframes` if not already present. Appends the function to `__scheduled_events[interval]`.

---

#### `deactivate_schedules(self) -> None`

Sets `__run_schedules = False`.

#### `activate_schedules(self) -> None`

Sets `__run_schedules = True`.

---

#### `backtest(self, ...) -> BacktestResults`

Instantiates all framework components and runs a simulated backtest.

**Parameters**:
| Name | Type | Default | Description |
|---|---|---|---|
| `strategy_id` | `str` | `"123456"` | Strategy identifier; cast to `int` for magic number. Must be numeric. |
| `initial_capital` | `float` | `10000.0` | Starting account balance. |
| `account_currency` | `AccountCurrencies` | `AccountCurrencies.USD` | Account denomination currency. |
| `account_leverage` | `int` | `30` | Account leverage ratio. |
| `start_date` | `datetime` | `datetime(1970, 1, 1)` | Backtest start date filter. |
| `end_date` | `datetime` | `datetime.now()` | Backtest end date filter. **Note**: evaluated at import/definition time. |
| `backtest_name` | `str` | `"Backtests"` | Label for this backtest run. |
| `symbols_to_trade` | `list[str]` | `['EURUSD']` | List of instrument symbols. |
| `csv_dir` | `str` | `None` | Path to CSV data directory. Defaults to internal `historical_csv_data` directory. |
| `run_scheduled_taks` | `bool` | `False` | **Note**: parameter exists but is never used in the method body. |
| `export_backtest_csv` | `bool` | `False` | Whether to export results as CSV. |
| `export_backtest_parquet` | `bool` | `True` | Whether to export results as Parquet. |
| `backtest_results_dir` | `str` | `None` | Directory for backtest result output. |

**Returns**: `BacktestResults`

**Behavior**:
1. Creates a fresh `Queue` (allows nested backtests).
2. Sets `trading_context = TypeContext.BACKTEST`.
3. Sorts `__strategy_timeframes` ascending; uses the first as `base_timeframe`.
4. Instantiates `DataProvider` with `CSVBacktestDataConfig`.
5. Instantiates `ExecutionEngine` with `MT5SimulatedExecutionConfig`.
6. Instantiates `Portfolio`, `Modules`.
7. Creates signal/sizing/risk engines.
8. Creates `PortfolioHandler` and `TradingDirector`.
9. Forwards all scheduled events to `TradingDirector`.
10. Calls `TradingDirector.run()`, captures timing, logs duration.

---

#### `run_live(self, ...) -> None`

Instantiates all framework components and runs a live MT5 trading session.

**Parameters**:
| Name | Type | Default | Description |
|---|---|---|---|
| `mt5_configuration` | `Mt5PlatformConfig` | -- | MT5 platform connection configuration. |
| `strategy_id` | `str` | `"default"` | Strategy identifier. |
| `initial_capital` | `float` | `10000.0` | Starting account balance. |
| `symbols_to_trade` | `list[str]` | `['EURUSD']` | List of instrument symbols. |
| `heartbeat` | `float` | `0.1` | Polling interval in seconds for the live data feed. |

**Returns**: `None` (runs indefinitely).

**Behavior**: Same pipeline as `backtest()` but uses `MT5LiveDataConfig`, `MT5LiveExecutionConfig`, `MT5LiveSessionConfig`, and `TypeContext.LIVE`.

---

## Data Flow

```
User registers engines/schedules/hooks
            |
            v
    backtest() or run_live()
            |
            v
    Queue created (event bus)
            |
            v
    DataProvider --> BarEvent --> Queue
            |
            v
    TradingDirector.run() (event loop)
            |
            v
    BAR event --> SignalEngineService --> SIGNAL event --> Queue
    SIGNAL event --> SizingEngineService --> SuggestedOrder
    SuggestedOrder --> RiskEngineService --> ORDER event --> Queue
    ORDER event --> ExecutionEngine --> FILL event --> Queue
    FILL event --> Portfolio (state update)
            |
            v
    BacktestResults returned (backtest) or runs forever (live)
```

---

## Gaps & Issues

1. **Commented-out imports**: `optimization`, `hyperopt`, `uuid` imports suggest removed or incomplete optimization/walk-forward features.
2. **FIXME comments** (lines 394, 500): `"At some point we must move everything on Trading Director to strategy"` -- indicates planned architectural refactoring.
3. **`run_scheduled_taks` parameter unused**: The `backtest()` parameter `run_scheduled_taks` (also misspelled -- should be `tasks`) is never referenced in the method body. The instance attribute `__run_schedules` is passed instead.
4. **`setdefault` for engine registration**: Only the first registration per `strategy_id` is retained; subsequent decorator uses for the same ID are silently ignored. No warning or error is raised.
5. **`__create_mg_for_strategy_id` unused**: The method exists but the call site in `backtest()` is commented out. `strategy_id` is cast directly to `int(strategy_id)` instead.
6. **Mutable default arguments**: `strategy_timeframes` and `symbols_to_trade` parameters use mutable list literals as defaults. This is a known Python footgun that can cause shared state across calls.
7. **`end_date` default evaluated at definition time**: `datetime.now()` is evaluated when the method is defined, not when called. This means the default end date is fixed to the time the module was first loaded.
8. **Decorator functions do not return the wrapped function**: The decorators (`hook`, `custom_signal_engine`, `custom_sizing_engine`, `custom_risk_engine`, `run_every`) call `dict.setdefault()` or `hooks.add_hook()` inside the inner `decorator` function but do not `return fn`. This means the original function reference is replaced with `None` after decoration, making it uncallable outside the decorator context.

---

## Requirements Derived

1. **REQ-STRAT-001**: The system shall provide a facade class that lets users register custom signal, sizing, and risk engines via Python decorators.
2. **REQ-STRAT-002**: The system shall support pre-defined engine configurations (e.g., `MACrossoverConfig`, `MinSizingConfig`, `PassthroughRiskConfig`) as alternatives to custom engines.
3. **REQ-STRAT-003**: The system shall allow users to register periodic scheduled callbacks at configurable timeframe intervals.
4. **REQ-STRAT-004**: The system shall allow users to register lifecycle hooks (e.g., `ON_START`).
5. **REQ-STRAT-005**: The `backtest()` method shall create an isolated event pipeline (fresh Queue) to support nested/sequential backtests.
6. **REQ-STRAT-006**: The `run_live()` method shall connect to MT5 via `Mt5PlatformConfig` and run indefinitely.
7. **REQ-STRAT-007**: Strategy IDs must be numeric strings (cast to `int` for MT5 magic numbers).
8. **REQ-STRAT-008**: The system shall support multiple timeframes per strategy, sorted ascending, with the smallest used as the base timeframe.
9. **REQ-STRAT-009**: Backtest duration shall be logged at WARNING level upon completion.
