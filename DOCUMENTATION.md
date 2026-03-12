# PyEventBT — Complete Technical Documentation

**Version:** 0.0.5 | **Python:** ≥3.12 | **License:** Apache 2.0
**Authors:** Marti Castany, Alain Porto

---

## Table of Contents

1. [Package Overview](#1-package-overview)
2. [Dependencies](#2-dependencies)
3. [Directory Structure](#3-directory-structure)
4. [Events (`pyeventbt.events`)](#4-events)
5. [Strategy (`pyeventbt.strategy`)](#5-strategy)
6. [Data Provider (`pyeventbt.data_provider`)](#6-data-provider)
7. [Execution Engine (`pyeventbt.execution_engine`)](#7-execution-engine)
8. [Signal Engine (`pyeventbt.signal_engine`)](#8-signal-engine)
9. [Sizing Engine (`pyeventbt.sizing_engine`)](#9-sizing-engine)
10. [Risk Engine (`pyeventbt.risk_engine`)](#10-risk-engine)
11. [Portfolio (`pyeventbt.portfolio`)](#11-portfolio)
12. [Portfolio Handler (`pyeventbt.portfolio_handler`)](#12-portfolio-handler)
13. [Trading Director (`pyeventbt.trading_director`)](#13-trading-director)
14. [Broker / MT5 Mock (`pyeventbt.broker`)](#14-broker--mt5-mock)
15. [Schedule Service (`pyeventbt.schedule_service`)](#15-schedule-service)
16. [Hooks (`pyeventbt.hooks`)](#16-hooks)
17. [Trade Archiver (`pyeventbt.trade_archiver`)](#17-trade-archiver)
18. [Backtest Results (`pyeventbt.backtest`)](#18-backtest-results)
19. [Indicators (`pyeventbt.indicators`)](#19-indicators)
20. [Configuration (`pyeventbt.config`)](#20-configuration)
21. [Core Entities (`pyeventbt.core`)](#21-core-entities)
22. [Trading Context (`pyeventbt.trading_context`)](#22-trading-context)
23. [Utilities (`pyeventbt.utils`)](#23-utilities)
24. [CSV Data Format](#24-csv-data-format)
25. [Event Loop — Full Execution Flow](#25-event-loop--full-execution-flow)
26. [Architecture Constraints and Rules](#26-architecture-constraints-and-rules)
27. [Core Design Pattern — Understanding and Reusing the Event-Driven Architecture](docs/design_pattern.md)
28. [Architecture Limitations & Distributed Migration Guide](docs/distributed_migration.md)
29. [Component Contracts, Behaviors & Protocols](docs/contracts_protocols.md)
30. [Technical Review — Dependencies, Architecture & Runtime Issues](docs/technical_review.md)
31. [Industry Research — Event-Driven Trading Architectures & Process Separation](docs/industry_research.md)
32. [Architecture Comparison — Industry Research vs PyEventBT](docs/architecture_comparison.md)
33. [Implementation Guide — From Design to Working Code](docs/implementation_guide.md)
34. [Contract, Behavior & Protocol Diagrams](docs/cbp_diagrams.md)

---

## 1. Package Overview

PyEventBT is an **event-driven backtesting and live trading framework** for MetaTrader 5 (MT5). Its core design principle is a single shared `queue.Queue` through which all components communicate exclusively via typed events.

**Public surface (importable from `pyeventbt`):**

| Symbol | Description |
|---|---|
| `Strategy` | Main user-facing class |
| `Modules` | Dependency injection container passed to callbacks |
| `StrategyTimeframes` | Enum of all supported timeframes |
| `BarEvent`, `SignalEvent`, `OrderEvent`, `FillEvent` | Event types |
| `PassthroughRiskConfig`, `BaseRiskConfig` | Risk engine configs |
| `BaseSizingConfig`, `MinSizingConfig`, `FixedSizingConfig`, `RiskPctSizingConfig` | Sizing engine configs |
| `Mt5PlatformConfig` | Live MT5 connection config |
| `HyperParameter`, `Variable` | Strategy parameter helpers |
| `Portfolio` | Portfolio state access |
| `Bar` | Bar data entity |
| `QuantdleDataUpdater` | Historical data downloader |
| `BacktestResults` | Backtest output object |
| `indicators` | Submodule: `SMA`, `KAMA`, `BollingerBands`, `ATR`, `RSI`, ... |

---

## 2. Dependencies

### Required (installed automatically)

| Package | Version | Used for |
|---|---|---|
| `pydantic` | `^2.12.3` | Data validation on all entities, configs, events |
| `numpy` | `^2.0.0` | Array math in indicators |
| `polars` | `^1.35.0` | Primary DataFrame format for bar data |
| `pandas` | `^2.2.3` | PnL export, legacy methods, `BacktestResults` |
| `numba` | `^0.62.1` | `@njit` JIT compilation for indicator loops |
| `matplotlib` | `^3.7.0` | `BacktestResults.plot()` |
| `scipy` | `^1.10.0` | Present in imports; currently in commented optimization code |
| `scikit-learn` | `^1.3.0` | Present in imports; currently in commented optimization code |
| `PyYAML` | `^6.0` | Loading default broker YAML config files |
| `python-dotenv` | `^1.1.1` | `.env` file loading for live credentials |

### Optional

| Package | Condition | Used for |
|---|---|---|
| `MetaTrader5` | Windows only, live mode only | Real MT5 terminal integration |
| `quantdle` | Only if using `QuantdleDataUpdater` | Downloading historical FX data |

### Platform Notes
- **Backtest mode**: works on any OS (macOS, Linux, Windows)
- **Live trading**: requires Windows (MT5 Python API is Windows-only); `check_platform_compatibility()` raises an exception on non-Windows with a clear message

---

## 3. Directory Structure

```
pyeventbt/
├── __init__.py                         Public API surface
├── app.py                              CLI entry point (pyeventbt --version)
├── backtest/
│   └── core/
│       └── backtest_results.py         BacktestResults class
├── broker/
│   └── mt5_broker/
│       ├── mt5_simulator_wrapper.py    Drop-in MT5 API mock (Mt5SimulatorWrapper)
│       ├── connectors/
│       │   ├── live_mt5_broker.py      LiveMT5Broker — live connection
│       │   └── mt5_simulator_connector.py  Simulator connectors
│       ├── core/
│       │   ├── entities/               AccountInfo, SymbolInfo, TradePosition, TradeDeal,
│       │   │                           TradeOrder, TradeRequest, OrderSendResult, Tick,
│       │   │                           ClosedPosition, TerminalInfo, InitCredentials
│       │   └── interfaces/
│       │       └── mt5_broker_interface.py  IPlatform, IAccountInfo, ISymbol, ...
│       └── shared/
│           ├── shared_data.py          SharedData singleton (global broker state)
│           ├── default_account_info.yaml
│           ├── default_terminal_info.yaml
│           └── default_symbols_info.yaml  33 FX pairs with full SymbolInfo specs
├── config/
│   ├── configs.py                      Mt5PlatformConfig
│   └── core/entities/base_config.py    BaseConfig
├── core/
│   └── entities/
│       ├── variable.py                 Variable
│       └── hyper_parameter.py          HyperParameter, HyperParameterRange/Values
├── data_provider/
│   ├── connectors/
│   │   ├── csv_data_connector.py       CSVDataProvider — backtest data engine
│   │   ├── mt5_live_data_connector.py  Mt5LiveDataProvider — live data
│   │   └── historical_csv_data/
│   │       └── EURUSD.csv              Bundled default dataset
│   ├── core/
│   │   ├── configurations/
│   │   │   └── data_provider_configurations.py  CSVBacktestDataConfig, MT5LiveDataConfig
│   │   ├── entities/bar.py             (unused public Bar; actual Bar is in events.py)
│   │   └── interfaces/
│   │       └── data_provider_interface.py  IDataProvider
│   └── services/
│       ├── data_provider_service.py    DataProvider (factory + proxy)
│       └── quantdle_data_updater.py    QuantdleDataUpdater
├── events/
│   └── events.py                       All event types + Bar dataclass
├── execution_engine/
│   ├── connectors/
│   │   ├── mt5_simulator_execution_engine_connector.py
│   │   └── mt5_live_execution_engine_connector.py
│   ├── core/
│   │   ├── configurations/
│   │   │   └── execution_engine_configurations.py  MT5SimulatedExecutionConfig, MT5LiveExecutionConfig
│   │   └── interfaces/
│   │       └── execution_engine_interface.py  IExecutionEngine
│   └── services/
│       └── execution_engine_service.py  ExecutionEngine (factory + proxy)
├── hooks/
│   └── hook_service.py                 HookService, Hooks enum
├── indicators/
│   ├── indicators.py                   KAMA, SMA, BollingerBands, ATR, RSI, ...
│   └── core/interfaces/
│       └── indicator_interface.py      IIndicator
├── portfolio/
│   ├── portfolio.py                    Portfolio
│   └── core/
│       ├── entities/
│       │   ├── open_position.py        OpenPosition
│       │   ├── pending_order.py        PendingOrder
│       │   └── closed_position.py      ClosedPosition
│       └── interfaces/
│           └── portfolio_interface.py  IPortfolio
├── portfolio_handler/
│   ├── portfolio_handler.py            PortfolioHandler
│   └── core/entities/
│       └── suggested_order.py          SuggestedOrder
├── risk_engine/
│   ├── core/
│   │   ├── configurations/
│   │   │   └── risk_engine_configurations.py  BaseRiskConfig, PassthroughRiskConfig
│   │   └── interfaces/
│   │       └── risk_engine_interface.py  IRiskEngine
│   ├── risk_engines/
│   │   └── passthrough_risk_engine.py  PassthroughRiskEngine
│   └── services/
│       └── risk_engine_service.py      RiskEngineService
├── schedule_service/
│   └── schedule_service.py             ScheduleService, Schedules, Schedule
├── signal_engine/
│   ├── core/
│   │   ├── configurations/
│   │   │   └── signal_engine_configurations.py  MACrossoverConfig
│   │   └── interfaces/
│   │       └── signal_engine_interface.py  ISignalEngine
│   ├── services/
│   │   └── signal_engine_service.py    SignalEngineService
│   └── signal_engines/
│       ├── signal_ma_crossover.py      SignalMACrossover (built-in)
│       └── signal_passthrough.py       SignalPassthrough (no-op placeholder)
├── sizing_engine/
│   ├── core/
│   │   ├── configurations/
│   │   │   └── sizing_engine_configurations.py  MinSizingConfig, FixedSizingConfig, RiskPctSizingConfig
│   │   └── interfaces/
│   │       └── sizing_engine_interface.py  ISizingEngine
│   ├── services/
│   │   └── sizing_engine_service.py    SizingEngineService
│   └── sizing_engines/
│       ├── mt5_min_sizing.py           MT5MinSizing
│       ├── mt5_fixed_sizing.py         MT5FixedSizing
│       └── mt5_risk_pct_sizing.py      MT5RiskPctSizing
├── strategy/
│   ├── strategy.py                     Strategy (main user-facing class)
│   └── core/
│       ├── modules.py                  Modules
│       ├── strategy_timeframes.py      StrategyTimeframes enum
│       ├── account_currencies.py       AccountCurrencies enum
│       ├── verbose_level.py            VerboseLevel
│       ├── errors.py                   Custom exception types
│       └── walk_forward.py             WalkForwardResults, WalkforwardType
├── trade_archiver/
│   └── trade_archiver.py              TradeArchiver
├── trading_context/
│   └── trading_context.py             TypeContext enum
├── trading_director/
│   ├── trading_director.py            TradingDirector (main event loop)
│   └── core/configurations/
│       └── trading_session_configurations.py  MT5BacktestSessionConfig, MT5LiveSessionConfig
└── utils/
    └── utils.py                        Utils, LoggerColorFormatter, TerminalColors, check_platform_compatibility
```

---

## 4. Events

**File:** `pyeventbt/events/events.py`

All inter-component communication passes through typed event objects placed on the shared `Queue`.

---

### `EventType` (str, Enum)

| Value | Description |
|---|---|
| `BAR` | OHLCV bar data event |
| `SIGNAL` | Trading signal from signal engine |
| `ORDER` | Sized and risk-approved order |
| `FILL` | Trade execution confirmation |
| `SCHEDULED_EVENT` | Timer-based callback trigger |

---

### `SignalType` (str, Enum)

`BUY`, `SELL`

---

### `OrderType` (str, Enum)

| Value | Description |
|---|---|
| `MARKET` | Execute immediately at current price |
| `LIMIT` | Execute at a price better than specified |
| `STOP` | Execute when price reaches specified level |
| `CONT` | Continuation (internal use) |

---

### `DealType` (str, Enum)

`IN` — position opened, `OUT` — position closed

---

### `Bar` (dataclass, `slots=True`)

Memory-compact OHLCV payload (~56 bytes). Prices stored as integers for precision and performance.

| Field | Type | Description |
|---|---|---|
| `open` | `int` | Open price × 10^digits |
| `high` | `int` | High price × 10^digits |
| `low` | `int` | Low price × 10^digits |
| `close` | `int` | Close price × 10^digits |
| `tickvol` | `int` | Tick volume (number of price changes) |
| `volume` | `int` | Real volume |
| `spread` | `int` | Bid-ask spread in points |
| `digits` | `int` | Symbol decimal precision |

**Computed properties (lazy, cached via `__price_factor`):**

| Property | Returns | Formula |
|---|---|---|
| `open_f` | `float` | `open / 10^digits` |
| `high_f` | `float` | `high / 10^digits` |
| `low_f` | `float` | `low / 10^digits` |
| `close_f` | `float` | `close / 10^digits` |
| `spread_f` | `float` | `spread / 10^digits` |

---

### `BarEvent` (Pydantic BaseModel → EventBase)

Envelope wrapping a `Bar` with market metadata.

| Field | Type | Default |
|---|---|---|
| `type` | `EventType` | `EventType.BAR` |
| `symbol` | `str` | required |
| `datetime` | `datetime` | required |
| `data` | `Bar` | required |
| `timeframe` | `str` | required (e.g. `"1h"`, `"1D"`) |

---

### `SignalEvent` (Pydantic BaseModel → EventBase)

Output of the user's `@strategy.custom_signal_engine` function.

| Field | Type | Notes |
|---|---|---|
| `type` | `EventType` | `EventType.SIGNAL` |
| `symbol` | `str` | |
| `time_generated` | `datetime` | When signal was generated |
| `strategy_id` | `str` | Numeric string = MT5 magic number |
| `forecast` | `Optional[float]` | -20 to +20 scale, default `0.0` |
| `signal_type` | `SignalType` | `"BUY"` or `"SELL"` |
| `order_type` | `OrderType` | `"MARKET"`, `"LIMIT"`, `"STOP"` |
| `order_price` | `Optional[Decimal]` | Required for LIMIT/STOP orders |
| `sl` | `Optional[Decimal]` | Stop loss price (required for `RiskPctSizingConfig`) |
| `tp` | `Optional[Decimal]` | Take profit price |
| `rollover` | `Optional[tuple]` | `(False,"","")` or `(True, "old_contract", "new_contract")` |

---

### `OrderEvent` (Pydantic BaseModel → EventBase)

Created by `RiskEngineService` after sizing and risk approval.

| Field | Type | Notes |
|---|---|---|
| All fields from `SignalEvent` | | |
| `volume` | `Decimal` | Calculated lot size |
| `buffer_data` | `Optional[dict]` | Carry-along data for execution engine |

---

### `FillEvent` (Pydantic BaseModel → EventBase)

Trade execution confirmation emitted by `ExecutionEngine`.

| Field | Type |
|---|---|
| `type` | `EventType.FILL` |
| `deal` | `DealType` (`IN` = open, `OUT` = close) |
| `symbol` | `str` |
| `time_generated` | `datetime` |
| `position_id` | `int` |
| `strategy_id` | `str` |
| `exchange` | `str` |
| `volume` | `Decimal` |
| `price` | `Decimal` |
| `signal_type` | `SignalType` |
| `commission` | `Decimal` |
| `swap` | `Decimal` |
| `fee` | `Decimal` |
| `gross_profit` | `Decimal` |
| `ccy` | `str` (account currency) |

---

### `ScheduledEvent` (Pydantic BaseModel → EventBase)

Fired by `ScheduleService` for `@strategy.run_every` callbacks.

| Field | Type |
|---|---|
| `type` | `EventType.SCHEDULED_EVENT` |
| `schedule_timeframe` | `StrategyTimeframes` |
| `symbol` | `str` |
| `timestamp` | `pd.Timestamp` |
| `former_execution_timestamp` | `pd.Timestamp \| None` |

---

## 5. Strategy

**File:** `pyeventbt/strategy/strategy.py`

### `Strategy`

Top-level user-facing class. Wires all framework components and exposes decorators for registering strategy logic.

**Constructor:**
```python
Strategy(logging_level: VerboseLevel = VerboseLevel.INFO)
```

Sets up colored logger and initializes internal state (empty engine registries, `HookService`, etc.).

---

#### Decorator Methods

**`@strategy.custom_signal_engine(strategy_id, strategy_timeframes)`**

Registers a signal generator function. Called once per bar per registered timeframe.

```python
@strategy.custom_signal_engine(
    strategy_id="1234",
    strategy_timeframes=[StrategyTimeframes.ONE_HOUR]
)
def my_signal(event: BarEvent, modules: Modules) -> list[SignalEvent]:
    ...
    return [SignalEvent(...)]  # or return []
```

- `strategy_id`: numeric string, becomes the MT5 magic number
- `strategy_timeframes`: list of `StrategyTimeframes`; multiple timeframes cause the callback to be invoked for each; branch using `event.timeframe`
- Return type: `list[SignalEvent]` or `[]` (no signal); also accepts single `SignalEvent` for backward compatibility

---

**`@strategy.custom_sizing_engine(strategy_id)`**

Registers a custom sizing function (overrides predefined config).

```python
@strategy.custom_sizing_engine(strategy_id="1234")
def my_sizing(signal_event: SignalEvent, modules: Modules) -> SuggestedOrder:
    return SuggestedOrder(signal_event=signal_event, volume=Decimal("0.01"))
```

---

**`@strategy.custom_risk_engine(strategy_id)`**

Registers a custom risk function (overrides predefined config).

```python
@strategy.custom_risk_engine(strategy_id="1234")
def my_risk(suggested_order: SuggestedOrder, modules: Modules) -> float:
    return float(suggested_order.volume)  # return 0.0 to block order
```

- Return value is the final volume; `0.0` blocks the order entirely

---

**`@strategy.run_every(interval: StrategyTimeframes)`**

Registers a periodic callback independent of signal generation.

```python
@strategy.run_every(StrategyTimeframes.ONE_DAY)
def daily_rebalance(event: ScheduledEvent, modules: Modules) -> None:
    print(f"Daily rebalance at {event.timestamp}")
```

- Triggering uses elapsed time comparison (`current - last >= interval.to_timedelta()`)
- `event.former_execution_timestamp` gives the previous execution time

---

**`@strategy.hook(hook: Hooks)`**

Registers lifecycle callbacks.

```python
@strategy.hook(Hooks.ON_START)
def on_start(modules: Modules) -> None:
    print("Strategy starting")
```

Available hooks: `ON_START`, `ON_SIGNAL_EVENT`, `ON_ORDER_EVENT`, `ON_END`

---

#### Configuration Methods

| Method | Argument | Effect |
|---|---|---|
| `configure_predefined_signal_engine(conf, timeframes)` | `MACrossoverConfig` | Use built-in MA crossover signal engine |
| `configure_predefined_sizing_engine(conf)` | `MinSizingConfig \| FixedSizingConfig \| RiskPctSizingConfig` | Set sizing method |
| `configure_predefined_risk_engine(conf)` | `PassthroughRiskConfig` | Set risk filter (only passthrough available by default) |

---

#### Execution Methods

**`Strategy.backtest(...) → BacktestResults`**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `strategy_id` | `str` | `"123456"` | Numeric string; becomes MT5 magic number |
| `initial_capital` | `float` | `10000.0` | Starting account balance |
| `account_currency` | `AccountCurrencies` | `USD` | `"USD"`, `"EUR"`, or `"GBP"` |
| `account_leverage` | `int` | `30` | Account leverage multiplier |
| `start_date` | `datetime` | `1970-01-01` | Backtest start |
| `end_date` | `datetime` | `datetime.now()` | Backtest end |
| `backtest_name` | `str` | `"Backtests"` | Name for export files |
| `symbols_to_trade` | `list[str]` | `['EURUSD']` | Tradeable symbols |
| `csv_dir` | `str \| None` | `None` | Path to CSV files; `None` uses bundled EURUSD.csv |
| `run_scheduled_taks` | `bool` | `False` | Whether to run `@run_every` schedules |
| `export_backtest_csv` | `bool` | `False` | Export results to CSV |
| `export_backtest_parquet` | `bool` | `True` | Export results to Parquet |
| `backtest_results_dir` | `str \| None` | `None` | Export directory; `None` = `~/Desktop/PyEventBT/` |

**`Strategy.run_live(...) → None` (blocks)**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `mt5_configuration` | `Mt5PlatformConfig` | required | MT5 credentials and path |
| `strategy_id` | `str` | `"default"` | Numeric string |
| `initial_capital` | `float` | `10000.0` | For portfolio tracking |
| `symbols_to_trade` | `list[str]` | `['EURUSD']` | |
| `heartbeat` | `float` | `0.1` | Seconds between loop iterations |

---

#### Other Methods

| Method | Description |
|---|---|
| `deactivate_schedules()` / `activate_schedules()` | Enable/disable `@run_every` callbacks |
| `enable_hooks()` / `disable_hooks()` | Enable/disable hook callbacks |

---

### `Modules` (Pydantic BaseModel)

**File:** `pyeventbt/strategy/core/modules.py`

Dependency injection container passed to all user callbacks.

| Field | Type |
|---|---|
| `TRADING_CONTEXT` | `TypeContext` — `"BACKTEST"` or `"LIVE"` |
| `DATA_PROVIDER` | `IDataProvider` |
| `EXECUTION_ENGINE` | `IExecutionEngine` |
| `PORTFOLIO` | `IPortfolio` |

---

### `StrategyTimeframes` (str, Enum)

**File:** `pyeventbt/strategy/core/strategy_timeframes.py`

| Enum value | String | Duration |
|---|---|---|
| `ONE_MIN` | `'1min'` | 1 minute |
| `TWO_MIN` | `'2min'` | 2 minutes |
| `THREE_MIN` | `'3min'` | |
| `FOUR_MIN` | `'4min'` | |
| `FIVE_MIN` | `'5min'` | |
| `SIX_MIN` | `'6min'` | |
| `TEN_MIN` | `'10min'` | |
| `TWELVE_MIN` | `'12min'` | |
| `FIFTEEN_MIN` | `'15min'` | |
| `TWENTY_MIN` | `'20min'` | |
| `THIRTY_MIN` | `'30min'` | |
| `ONE_HOUR` | `'1h'` | 1 hour |
| `TWO_HOUR` | `'2h'` | |
| `THREE_HOUR` | `'3h'` | |
| `FOUR_HOUR` | `'4h'` | |
| `SIX_HOUR` | `'6h'` | |
| `EIGHT_HOUR` | `'8h'` | |
| `TWELVE_HOUR` | `'12h'` | |
| `ONE_DAY` | `'1D'` | 1 day |
| `ONE_WEEK` | `'1W'` | 1 week |
| `ONE_MONTH` | `'1M'` | ~30 days |
| `SIX_MONTH` | `'6M'` | ~180 days |
| `ONE_YEAR` | `'12M'` | 365 days |

**Methods:**
- `to_timedelta() → timedelta` — convert to Python timedelta
- `__eq__` — supports comparison with both `str` and other `StrategyTimeframes`
- `__lt__`, `__gt__` — sortable by duration
- `__hash__` — hashable (usable as dict key)

**Note:** `event.timeframe` is a plain `str`; comparison with `StrategyTimeframes` works because `__eq__` handles strings.

---

## 6. Data Provider

### `IDataProvider` (interface)

**File:** `pyeventbt/data_provider/core/interfaces/data_provider_interface.py`

All data provider implementations must implement:

| Method | Signature | Returns |
|---|---|---|
| `get_latest_bar` | `(symbol, timeframe)` | `BarEvent` |
| `get_latest_bars` | `(symbol, timeframe, N)` | `pl.DataFrame` |
| `get_latest_tick` | `(symbol)` | `dict` |
| `get_latest_bid` | `(symbol)` | `Decimal` |
| `get_latest_ask` | `(symbol)` | `Decimal` |
| `get_latest_datetime` | `(symbol, timeframe)` | `datetime` |
| `update_bars` | `()` | `list[BarEvent]` (connector) or `None` (service) |

---

### `DataProvider` (service, factory + proxy)

**File:** `pyeventbt/data_provider/services/data_provider_service.py`

**Constructor:**
```python
DataProvider(events_queue: Queue, data_config: BaseDataConfig, trading_context: TypeContext)
```

Selects connector based on config type:
- `CSVBacktestDataConfig` → `CSVDataProvider`
- `MT5LiveDataConfig` → `Mt5LiveDataProvider`

**State flags:**
- `continue_backtest: bool` — `False` when all data exhausted
- `close_positions_end_of_data: bool` — `True` when generator reaches `StopIteration`

**`update_bars()`:** calls connector's `update_bars()`, puts each `BarEvent` on the queue, syncs flags.

---

### `CSVBacktestDataConfig` (Pydantic BaseModel)

**File:** `pyeventbt/data_provider/core/configurations/data_provider_configurations.py`

| Field | Type | Required | Description |
|---|---|---|---|
| `csv_path` | `str` | yes | Directory containing `{SYMBOL}.csv` files |
| `account_currency` | `str` | yes | `"USD"`, `"EUR"`, or `"GBP"` |
| `tradeable_symbol_list` | `list[str]` | yes | Symbols to trade |
| `base_timeframe` | `str` | yes | Must be first and smallest in `timeframes_list` |
| `timeframes_list` | `list[str]` | yes | All timeframes, ascending order |
| `backtest_start_timestamp` | `datetime \| None` | no | Filter start |
| `backtest_end_timestamp` | `datetime` | no | Filter end (default: `datetime.now()`) |

---

### `MT5LiveDataConfig` (Pydantic BaseModel)

| Field | Type |
|---|---|
| `tradeable_symbol_list` | `list[str]` |
| `timeframes_list` | `list[str]` |

---

### `CSVDataProvider` (backtest data engine)

**File:** `pyeventbt/data_provider/connectors/csv_data_connector.py`

**Full initialization pipeline:**

1. Validates that `base_timeframe == timeframes_list[0]`
2. Validates `account_currency ∈ {"USD","EUR","GBP"}`
3. Builds auxiliary symbol list: cross-rate pairs needed to convert P&L to account currency (e.g. trading GBPJPY with USD account needs USDJPY)
4. For each symbol in `symbol_list` (tradeable + auxiliary):
   - Lazy-scans CSV with Polars (`scan_csv`)
   - Parses datetime, casts OHLCV types
   - Filters to `[start_date, end_date]`
   - Collects eagerly, sorts, rechunks
   - Resamples to M1 via `group_by_dynamic("datetime", every="1m")`
   - Annotates bars with `minute_idx` (global minutes since epoch) and `minute_of_day`
5. Builds a merged sorted datetime master index across all symbols
6. For each symbol: left-joins onto master index → forward-fills gaps (OHLC from prior close; tickvol=volume=spread=1 for filled bars)
7. Resamples from the M1 base to all requested timeframes using `group_by_dynamic`
   - Weekly bars get a -7-day correction to fix Polars' week boundary alignment
8. Builds integer-based lookup caches: `_base_timestamps`, `_base_idx_map`, `_base_minutes_global`, etc.
9. Creates a **zero-copy generator** per symbol using `memoryview` over Arrow arrays for maximum throughput

**`update_bars()` logic:**
- Calls `next()` on each symbol's generator
- Skips bars where `tickvol == volume == spread == 1` (gap-filled null bars)
- Sets `close_positions_end_of_data = True` on `StopIteration`
- Emits one base-TF `BarEvent` per tradeable symbol
- For each higher timeframe: checks if the current bar crosses a TF boundary using `_base_tf_bar_creates_new_tf_bar()`; if yes, emits a higher-TF `BarEvent` using `get_latest_bar(symbol, tf)` (which returns the second-to-last bar to avoid lookahead)

**`get_latest_bars(symbol, timeframe, N)` anti-lookahead logic:**
- Base timeframe: returns last N bars up to `latest_index_timeframes[symbol][tf]`
- Higher timeframes: excludes the most recently forming bar (returns `[0 : df_len-1]` sliced to N)

**`get_latest_tick(symbol)` anti-lookahead logic:**
- Reads the next (not yet emitted) bar's open price as bid
- Ask = bid + spread × point size
- At end of data, falls back to the last bar's close

**Timeframe boundary detection — `_base_tf_bar_creates_new_tf_bar(latest_dt, tf, symbol)`:**
- Sub-hourly: checks minute bucket (`minute // tfm`)
- Hourly: checks `hour`
- Daily: checks `day`
- Weekly: checks ISO week number
- Monthly: checks `month`

**Supported auxiliary symbols (33 FX pairs):**
AUDCAD, AUDCHF, AUDJPY, AUDNZD, AUDUSD, CADCHF, CADJPY, CHFJPY, EURAUD, EURCAD, EURCHF, EURGBP, EURJPY, EURNZD, EURUSD, GBPAUD, GBPCAD, GBPCHF, GBPJPY, GBPNZD, GBPUSD, NZDCAD, NZDCHF, NZDJPY, NZDUSD, USDCAD, USDCHF, USDJPY, USDSEK, USDNOK, USDMXN, EURMXN, GBPMXN

---

### `Mt5LiveDataProvider` (live data)

**File:** `pyeventbt/data_provider/connectors/mt5_live_data_connector.py`

**Constructor:**
```python
Mt5LiveDataProvider(configs: MT5LiveDataConfig)
```

Requires Windows + MT5 installed. Uses `mt5.copy_rates_from_pos()` to fetch bars.

**`update_bars()` logic:**
- For each symbol × timeframe: calls `get_latest_bar()`
- Only emits `BarEvent` if `latest_bar.datetime > last_bar_tf_datetime[symbol][timeframe]`
- Returns list of new `BarEvent`s

**`get_latest_bars()` → `pl.DataFrame`:**
- Calls `mt5.copy_rates_from_pos(symbol, tf, from_pos=1, count=N)`
- Renames `tick_volume→tickvol`, `real_volume→volume`
- Returns columns: `datetime, open, high, low, close, tickvol, volume, spread`

**`get_latest_tick()` → `dict`:**
- Calls `mt5.symbol_info_tick(symbol)`
- Returns `{time, bid, ask, last, volume, time_msc, flags, volume_real}` with Decimal prices

**Timeframe string mapping (for `_map_timeframe`):**

| String | MT5 constant |
|---|---|
| `'1min'` | `TIMEFRAME_M1` |
| `'5min'` | `TIMEFRAME_M5` |
| `'1h'`/`'1H'` | `TIMEFRAME_H1` |
| `'4h'`/`'4H'` | `TIMEFRAME_H4` |
| `'1D'` | `TIMEFRAME_D1` |
| `'1W'` | `TIMEFRAME_W1` |
| `'1M'` | `TIMEFRAME_MN1` |

---

### `QuantdleDataUpdater`

**File:** `pyeventbt/data_provider/services/quantdle_data_updater.py`
**Requires:** `pip install quantdle`

**Constructor:**
```python
QuantdleDataUpdater(api_key: str, api_key_id: str)
```

**`update_data(csv_dir, symbols, start_date, end_date, timeframe="1min", spread_column="spreadopen") → None`**

Smart cache update:
1. If CSV does not exist: downloads full range and creates it
2. If CSV exists: checks date range; downloads only missing data before/after existing range; merges, deduplicates, re-sorts, saves

**`_convert_to_quantdle_timeframe()`** mapping:

| Internal | Quantdle |
|---|---|
| `"1min"` | `"M1"` |
| `"5min"` | `"M5"` |
| `"15min"` | `"M15"` |
| `"30min"` | `"M30"` |
| `"1h"` / `"1H"` | `"H1"` |
| `"4h"` / `"4H"` | `"H4"` |
| `"1D"` | `"D1"` |
| `"1W"` | `"W1"` |

Output CSV format is identical to the MT5 export format expected by `CSVDataProvider`.

---

## 7. Execution Engine

### `IExecutionEngine` (interface)

**File:** `pyeventbt/execution_engine/core/interfaces/execution_engine_interface.py`

All execution engine implementations must implement:

| Method | Returns | Description |
|---|---|---|
| `_process_order_event(order_event)` | `None` | Route order to broker |
| `_update_values_and_check_executions_and_fills(bar_event)` | `None` | Called every base-TF bar |
| `_send_market_order(order_event)` | `OrderSendResult` | Execute market order |
| `_send_pending_order(order_event)` | `OrderSendResult` | Place pending order |
| `close_position(ticket)` | `OrderSendResult` | Close single position |
| `close_all_strategy_positions()` | `None` | Close all positions |
| `close_strategy_long_positions_by_symbol(symbol)` | `None` | |
| `close_strategy_short_positions_by_symbol(symbol)` | `None` | |
| `cancel_pending_order(ticket)` | `OrderSendResult` | Cancel single pending order |
| `cancel_all_strategy_pending_orders()` | `None` | Cancel all pending orders |
| `cancel_all_strategy_pending_orders_by_type_and_symbol(type, symbol)` | `None` | e.g. all BUY_STOP on EURUSD |
| `update_position_sl_tp(ticket, sl, tp)` | `None` | Modify SL/TP |
| `_get_account_balance/equity/floating_profit/used_margin/free_margin()` | `Decimal` | Account info |
| `_get_strategy_positions()` | `tuple[OpenPosition]` | |
| `_get_strategy_pending_orders()` | `tuple[PendingOrder]` | |
| `_get_total_number_of_positions/pending_orders()` | `int` | |
| `_get_symbol_min_volume(symbol)` | `Decimal` | Minimum lot size |
| `enable_trading()` / `disable_trading()` | `None` | |

---

### Execution Engine Configurations

**File:** `pyeventbt/execution_engine/core/configurations/execution_engine_configurations.py`

**`BaseExecutionConfig`** — empty Pydantic base

**`MT5SimulatedExecutionConfig(BaseExecutionConfig)`**

| Field | Type | Description |
|---|---|---|
| `initial_balance` | `Decimal` | Starting account balance |
| `account_currency` | `str` | Account currency code |
| `account_leverage` | `int` | Leverage multiplier |
| `magic_number` | `int` | Strategy identifier |

**`MT5LiveExecutionConfig(BaseExecutionConfig)`**

| Field | Type |
|---|---|
| `magic_number` | `int` |

---

### `ExecutionEngine` (service, factory + proxy)

**File:** `pyeventbt/execution_engine/services/execution_engine_service.py`

**Constructor:**
```python
ExecutionEngine(events_queue: Queue, data_provider: IDataProvider, execution_config: BaseExecutionConfig)
```

Selects connector:
- `MT5SimulatedExecutionConfig` → `Mt5SimulatorExecutionEngineConnector`
- `MT5LiveExecutionConfig` → `Mt5LiveExecutionEngineConnector`

Has an `__enable_trading` flag; when `False`, `_process_order_event` logs a warning and skips execution.

---

### `Mt5SimulatorExecutionEngineConnector`

**File:** `pyeventbt/execution_engine/connectors/mt5_simulator_execution_engine_connector.py`

Full MT5 trade lifecycle simulation. Maintains an in-memory account state using `SharedData` (the MT5 mock broker).

**Constructor:**
```python
Mt5SimulatorExecutionEngineConnector(
    configs: MT5SimulatedExecutionConfig,
    events_queue: Queue,
    data_provider: IDataProvider
)
```

On initialization: sets account balance/currency/leverage in `SharedData`.

**`_update_values_and_check_executions_and_fills(bar_event)`** — called every bar:
1. Updates unrealized P&L for all open positions (mark to market using latest bid for SELL, ask for BUY)
2. Checks SL/TP hits on open positions
3. Checks pending order triggers (STOP: triggers when price crosses order price; LIMIT: similar)
4. Emits `FillEvent(deal=OUT)` for closed positions, `FillEvent(deal=IN)` for newly triggered pending orders

**`_process_order_event(order_event)`** — routes to:
- `_send_market_order` for `order_type == "MARKET"`
- `_send_pending_order` for `STOP`/`LIMIT`

**Market order fill:**
- BUY: fills at `ask` price; SELL: fills at `bid` price
- Creates `TradePosition`, stores in `SharedData`
- Emits `FillEvent(deal=IN)`
- Updates account balance, equity, margin

**Pending order:**
- Creates `TradeOrder`, stores in `SharedData`
- Triggers when bar's high (BUY_STOP) or low (SELL_STOP) crosses the order price

**Position close:**
- BUY close: at `bid`; SELL close: at `ask`
- Calculates `gross_profit` in profit currency, converts to account currency
- Emits `FillEvent(deal=OUT)` with full P&L

---

### `Mt5LiveExecutionEngineConnector`

**File:** `pyeventbt/execution_engine/connectors/mt5_live_execution_engine_connector.py`

Routes orders to the real MT5 terminal using `mt5.order_send()`. Reads positions and orders from the live MT5 account filtered by magic number.

---

## 8. Signal Engine

### `ISignalEngine` (Protocol)

**File:** `pyeventbt/signal_engine/core/interfaces/signal_engine_interface.py`

```python
def generate_signal(self, bar_event: BarEvent, modules: Modules) -> SignalEvent | list[SignalEvent]
```

---

### `SignalEngineService`

**File:** `pyeventbt/signal_engine/services/signal_engine_service.py`

**Constructor:**
```python
SignalEngineService(events_queue: Queue, modules: Modules, signal_config: BaseSignalEngineConfig = None)
```

Selects engine:
- `MACrossoverConfig` → `SignalMACrossover`
- anything else → `SignalPassthrough` (no-op; replaced when `@custom_signal_engine` is registered)

**`generate_signal(bar_event)`:** calls the engine, puts `SignalEvent`(s) on queue; handles both single and list returns.

**`set_signal_engine(fn)`:** replaces the internal engine with the user's decorated function.

---

### `MACrossoverConfig` (Pydantic BaseModel)

**File:** `pyeventbt/signal_engine/core/configurations/signal_engine_configurations.py`

| Field | Type | Default |
|---|---|---|
| `strategy_id` | `str` | required |
| `signal_timeframe` | `str` | required |
| `ma_type` | `MAType` | `MAType.SIMPLE` |
| `fast_period` | `int \| HyperParameter` | required |
| `slow_period` | `int \| HyperParameter` | required |

`MAType` enum: `SIMPLE`, `EXPONENTIAL`

---

### `SignalMACrossover`

**File:** `pyeventbt/signal_engine/signal_engines/signal_ma_crossover.py`

Built-in moving average crossover signal engine.

**Logic:**
- Skips bars where `bar_event.timeframe != signal_timeframe`
- Gets `slow_period + 1` bars from data provider
- `SIMPLE`: `fast_ma = close[-fast_period:].mean()`, `slow_ma = close.mean()`
- `EXPONENTIAL`: uses pandas `ewm(span=period).mean().iloc[-1]`
- BUY signal when no longs and `fast_ma > slow_ma`; closes shorts first
- SELL signal when no shorts and `fast_ma < slow_ma`; closes longs first
- Signal `forecast` is fixed at `10`
- `time_generated`: backtest = `bar.datetime + timedelta(signal_timeframe)`, live = `datetime.now()`

---

## 9. Sizing Engine

### `ISizingEngine` (Protocol)

```python
def get_suggested_order(self, signal_event: SignalEvent, modules: Modules) -> SuggestedOrder
```

---

### `SizingEngineService`

**File:** `pyeventbt/sizing_engine/services/sizing_engine_service.py`

**Constructor:**
```python
SizingEngineService(events_queue: Queue, modules: Modules, sizing_config: BaseSizingConfig)
```

Selects engine:
- `MinSizingConfig` → `MT5MinSizing`
- `FixedSizingConfig` → `MT5FixedSizing`
- `RiskPctSizingConfig` → `MT5RiskPctSizing`
- Anything else → `MT5MinSizing`

**`set_suggested_order_function(fn)`:** replaces when `@custom_sizing_engine` is registered.

---

### Sizing Engine Configs

**`BaseSizingConfig`** — empty Pydantic base

**`MinSizingConfig(BaseSizingConfig)`** — no fields

**`FixedSizingConfig(BaseSizingConfig)`**

| Field | Type |
|---|---|
| `volume` | `Decimal` |

**`RiskPctSizingConfig(BaseSizingConfig)`**

| Field | Type | Description |
|---|---|---|
| `risk_pct` | `float` | e.g. `1.0` = 1% of equity per trade |

---

### `MT5MinSizing`

Returns `mt5.symbol_info(symbol).volume_min` as `Decimal`. No further calculations.

---

### `MT5FixedSizing`

Returns the configured `volume` as `Decimal`. Ignores all market data.

---

### `MT5RiskPctSizing`

**Requires:** `signal_event.sl != 0` — raises `Exception` otherwise.

**Formula:**
```
price_distance  = abs(entry_price - sl) / tick_size     # in ticks
monetary_risk   = equity × risk_pct / 100               # in account currency
tick_value_accy = contract_size × tick_size             # profit per tick per lot
                  converted to account currency via cross-rate
volume          = monetary_risk / (price_distance × tick_value_accy)
volume          = round(volume / volume_step) × volume_step
```

For MARKET orders, `entry_price` = latest ask (BUY) or bid (SELL).
For STOP/LIMIT orders, `entry_price` = `order_price`.

**Currency conversion method `_convert_currency_amount_to_another_currency`:**
Finds the FX pair containing both `from_ccy` and `to_ccy`, uses latest bid; divides if `to_ccy` is the base currency, multiplies otherwise.

---

## 10. Risk Engine

### `IRiskEngine` (Protocol)

```python
def assess_order(self, suggested_order: SuggestedOrder, modules: Modules) -> float
```

Returns the final volume (`float`). Return `0.0` to block the order.

---

### `RiskEngineService`

**File:** `pyeventbt/risk_engine/services/risk_engine_service.py`

**Constructor:**
```python
RiskEngineService(events_queue: Queue, risk_config: BaseRiskConfig, modules: Modules)
```

Selects engine:
- `PassthroughRiskConfig` → `PassthroughRiskEngine`
- Anything else → `PassthroughRiskEngine`

**`assess_order(suggested_order)`:**
1. Gets `new_volume` from risk engine
2. If `new_volume > 0`, creates `OrderEvent` from `suggested_order.signal_event` fields + `new_volume` + `buffer_data`, puts on queue

**`_create_and_put_order_event(suggested_order, new_volume)`:** constructs `OrderEvent` and puts on queue.

**`set_custom_asses_order(fn)`:** replaces with user's `@custom_risk_engine` function.

---

### Risk Engine Configs

**`BaseRiskConfig`** — empty Pydantic base
**`PassthroughRiskConfig(BaseRiskConfig)`** — no fields

---

### `PassthroughRiskEngine`

```python
def assess_order(self, suggested_order, modules) -> float:
    return suggested_order.volume  # unchanged pass-through
```

---

## 11. Portfolio

### `IPortfolio` (interface)

**File:** `pyeventbt/portfolio/core/interfaces/portfolio_interface.py`

All portfolio implementations must implement the full set of position query and account info methods.

---

### `Portfolio` (implements `IPortfolio`)

**File:** `pyeventbt/portfolio/portfolio.py`

**Constructor:**
```python
Portfolio(
    initial_balance: Decimal,
    execution_engine: IExecutionEngine,
    trading_context: TypeContext,
    base_timeframe: str = '1min'
)
```

**State variables:**

| Variable | Type | Description |
|---|---|---|
| `_balance` | `Decimal` | Current cash balance |
| `_equity` | `Decimal` | Balance + unrealized P&L |
| `_realised_pnl` | `Decimal` | `balance - initial_balance` |
| `_unrealised_pnl` | `Decimal` | `equity - balance` |
| `_strategy_positions` | `tuple[OpenPosition]` | Current open positions |
| `_strategy_pending_orders` | `tuple[PendingOrder]` | Current pending orders |
| `historical_balance` | `dict[datetime, Decimal]` | Time-series (base TF, first symbol only) |
| `historical_equity` | `dict[datetime, Decimal]` | Time-series |

**`_update_portfolio(bar_event)`** — called by `PortfolioHandler` on each base-TF bar:
1. Calls `execution_engine._update_values_and_check_executions_and_fills(bar_event)`
2. Refreshes positions, pending orders, balance, equity from execution engine
3. Records `historical_balance` and `historical_equity` for the first seen symbol only

**Public query methods (accessible as `modules.PORTFOLIO`):**

| Method | Returns |
|---|---|
| `get_account_balance()` | `Decimal` |
| `get_account_equity()` | `Decimal` |
| `get_account_unrealised_pnl()` | `Decimal` |
| `get_account_realised_pnl()` | `Decimal` |
| `get_positions(symbol='', ticket=None)` | `tuple[OpenPosition]` |
| `get_pending_orders(symbol='', ticket=None)` | `tuple[PendingOrder]` |
| `get_number_of_strategy_open_positions_by_symbol(symbol)` | `{"LONG": int, "SHORT": int, "TOTAL": int}` |
| `get_number_of_strategy_pending_orders_by_symbol(symbol)` | `{"BUY_LIMIT": int, "SELL_LIMIT": int, "BUY_STOP": int, "SELL_STOP": int, "TOTAL": int}` |

**Export methods (internal):**

| Method | Description |
|---|---|
| `_export_historical_pnl_dataframe()` | `pd.DataFrame` with `BALANCE`, `EQUITY` columns, datetime index |
| `_export_csv_historical_pnl(path)` | Write CSV |
| `_export_historical_pnl_to_parquet(path)` | Write Parquet (zstd level 10); converts Decimal to Float64 |
| `_export_historical_pnl_json()` | JSON string; timestamps as `"%Y-%m-%dT%H:%M:%S"`, values scaled by 10^4 |
| `_update_portfolio_end_of_backtest()` | Final flush; logs colored P&L summary |

---

### `OpenPosition` (Pydantic BaseModel)

| Field | Type |
|---|---|
| `time_entry` | `datetime` |
| `price_entry` | `Decimal` |
| `type` | `str` — `"BUY"` or `"SELL"` |
| `symbol` | `str` |
| `ticket` | `int` |
| `volume` | `Decimal` |
| `unrealized_profit` | `Decimal` |
| `strategy_id` | `str` |
| `sl` | `Optional[Decimal]` |
| `tp` | `Optional[Decimal]` |
| `swap` | `Optional[Decimal]` |
| `comment` | `Optional[str]` |

---

### `PendingOrder` (Pydantic BaseModel)

| Field | Type | Notes |
|---|---|---|
| `price` | `Decimal` | Trigger price |
| `type` | `str` | `"BUY"`, `"SELL"`, `"BUY_LIMIT"`, `"SELL_LIMIT"`, `"BUY_STOP"`, `"SELL_STOP"` |
| `symbol` | `str` | |
| `ticket` | `int` | |
| `volume` | `Decimal` | |
| `strategy_id` | `str` | |
| `sl` | `Optional[Decimal]` | |
| `tp` | `Optional[Decimal]` | |
| `comment` | `Optional[str]` | |

---

### `ClosedPosition` (Pydantic BaseModel)

| Field | Type |
|---|---|
| `time_entry` | `datetime` |
| `price_entry` | `Decimal` |
| `time_exit` | `datetime` |
| `price_exit` | `Decimal` |
| `strategy_id` | `str` |
| `ticket` | `int` |
| `symbol` | `str` |
| `direction` | `str` |
| `volume` | `Decimal` |
| `commission` | `Decimal` |
| `pnl` | `Decimal` |
| `sl`, `tp`, `swap`, `comment` | `Optional` |

---

## 12. Portfolio Handler

### `PortfolioHandler`

**File:** `pyeventbt/portfolio_handler/portfolio_handler.py`

Orchestrates the signal → sizing → risk → order pipeline.

**Constructor:**
```python
PortfolioHandler(
    events_queue: Queue,
    sizing_engine: ISizingEngine,
    risk_engine: IRiskEngine,
    portfolio: IPortfolio,
    base_timeframe: str = '1min',
    backtest_results_dir: str = None
)
```

**Event handlers (called by `TradingDirector`):**

| Method | Action |
|---|---|
| `process_bar_event(bar_event)` | Returns early if `bar_event.timeframe != base_timeframe`; otherwise calls `Portfolio._update_portfolio()` |
| `process_signal_event(signal_event)` | Calls `SizingEngine.get_suggested_order()` → `RiskEngine.assess_order()` |
| `process_fill_event(fill_event)` | Calls `TradeArchiver.archive_trade(fill_event)` |

**`process_backtest_end(name, export_csv, export_parquet) → BacktestResults`:**
1. `Portfolio._update_portfolio_end_of_backtest()` — final mark-to-market
2. Creates `BacktestResults` from `Portfolio` PnL DataFrame and `TradeArchiver` trades DataFrame
3. Optionally exports CSV to `{base_dir}/PyEventBT/backtest_results_csv/{name}_{timestamp}/`
4. Optionally exports Parquet to `{base_dir}/PyEventBT/backtest_results_parquet/{name}_{timestamp}/`
5. Default `base_dir`: OS-aware `~/Desktop`

---

### `SuggestedOrder` (Pydantic BaseModel)

| Field | Type |
|---|---|
| `signal_event` | `SignalEvent` |
| `volume` | `Decimal` |
| `buffer_data` | `Optional[dict]` |

---

## 13. Trading Director

### `TradingDirector`

**File:** `pyeventbt/trading_director/trading_director.py`

The main event loop controller. Owns the dispatch table and drives backtest/live execution.

**Constructor:**
```python
TradingDirector(
    events_queue: Queue,
    signal_engine_service: SignalEngineService,
    portfolio_handler: PortfolioHandler,
    trading_session_config: BaseTradingSessionConfig,
    modules: Modules,
    run_schedules: bool = False,
    export_backtest: bool = False,
    export_backtest_parquet: bool = False,
    backtest_results_dir: str = None,
    hook_service: HookService = HookService()
)
```

**Session configuration:**
- `MT5BacktestSessionConfig(start_date, initial_capital, backtest_name)` → sets up backtest mode
- `MT5LiveSessionConfig(symbol_list, heartbeat, platform_config)` → sets up live mode, creates `LiveMT5Broker`

**Event dispatch map:**
```python
{
    EventType.BAR:    self._handle_bar_event,
    EventType.SIGNAL: self._handle_signal_event,
    EventType.ORDER:  self._handle_order_event,
    EventType.FILL:   self._handle_fill_event,
}
```

**`_handle_bar_event(event)`:**
1. `PortfolioHandler.process_bar_event(event)` — update portfolio values
2. `ScheduleService.run_scheduled_callbacks(event)` — fire `@run_every` callbacks
3. `SignalEngineService.generate_signal(event)` — run strategy logic

**`_handle_signal_event(event)`:**
1. `HookService.call_callbacks(Hooks.ON_SIGNAL_EVENT, modules)`
2. `PortfolioHandler.process_signal_event(event)` — sizing → risk → OrderEvent

**`_handle_order_event(event)`:**
1. `ExecutionEngine._process_order_event(event)` — execute trade → FillEvent
2. `HookService.call_callbacks(Hooks.ON_ORDER_EVENT, modules)`

**`_handle_fill_event(event)`:**
1. `PortfolioHandler.process_fill_event(event)` — archive trade

**`_run_backtest()` loop:**
```
while DATA_PROVIDER.continue_backtest:
    try:
        event = queue.get(block=False)
        dispatch event
    except queue.Empty:
        DATA_PROVIDER.update_bars()

    if close_positions_end_of_data:
        if any open positions: close_all_strategy_positions() and continue
        elif queue empty: set continue_backtest = False

return _handle_backtest_end()
```

**`_run_live_trading()` loop:**
```
while True:
    try:
        event = queue.get(block=False)
        dispatch event
    except queue.Empty:
        DATA_PROVIDER.update_bars()
    time.sleep(heartbeat)
```

**`run() → BacktestResults | None`:**
1. `HookService.call_callbacks(Hooks.ON_START, modules)`
2. Route to `_run_backtest()` or `_run_live_trading()`
3. `HookService.call_callbacks(Hooks.ON_END, modules)`
4. Return result

---

### Trading Session Configurations

**`MT5BacktestSessionConfig`**

| Field | Type |
|---|---|
| `start_date` | `datetime` |
| `initial_capital` | `float` |
| `backtest_name` | `str` |

**`MT5LiveSessionConfig`**

| Field | Type |
|---|---|
| `symbol_list` | `list[str]` |
| `heartbeat` | `float` |
| `platform_config` | `Mt5PlatformConfig` |

---

## 14. Broker / MT5 Mock

### `SharedData`

**File:** `pyeventbt/broker/mt5_broker/shared/shared_data.py`

Singleton-style class that holds global simulator state as **class-level attributes** (not instance attributes), shared across all objects.

**Class attributes:**
- `last_error_code: tuple` — last error as `(code, message)`
- `credentials: InitCredentials` — login credentials
- `terminal_info: TerminalInfo` — terminal state
- `account_info: AccountInfo` — account state (balance, equity, etc.)
- `symbol_info: dict[str, SymbolInfo]` — specs for all 33 FX pairs

On construction, loads all three YAML files. YAML floats are parsed as `Decimal` (custom YAML constructor).

---

### `Mt5SimulatorWrapper`

**File:** `pyeventbt/broker/mt5_broker/mt5_simulator_wrapper.py`

Drop-in replacement for the `MetaTrader5` Python package. Exposes identical static API.

**Usage:**
```python
from pyeventbt.broker.mt5_broker.mt5_simulator_wrapper import Mt5SimulatorWrapper as mt5
```

**Implemented methods:**

| Method | Returns | Notes |
|---|---|---|
| `initialize(path, login, password, server, timeout, portable)` | `bool` | Saves credentials to `SharedData`, sets connected=True |
| `login(login, password, server, timeout)` | `bool` | Same as initialize (no path) |
| `shutdown()` | `None` | Sets connected=False |
| `version()` | `tuple` | `(500, build, "20 Oct 2023")` |
| `last_error()` | `tuple` | Returns `SharedData.last_error_code` |
| `account_info()` | `AccountInfo` | Returns `SharedData.account_info` |
| `terminal_info()` | `TerminalInfo` | Returns `SharedData.terminal_info` |
| `symbols_total()` | `int` | Count of symbols in `SharedData.symbol_info` |
| `symbols_get(group="*")` | `tuple[SymbolInfo]` | Filters by group pattern (`*` wildcard, `!` exclusion) |
| `symbol_info(symbol)` | `SymbolInfo` | Lookup from `SharedData.symbol_info` |
| `symbol_info_tick(symbol)` | `Tick` | Partially implemented |
| `symbol_select(symbol, enable)` | `bool` | Sets `select` and `visible` on `SymbolInfo` |

**MT5 constants** (class-level):
Full set of TIMEFRAME_*, POSITION_*, ORDER_*, DEAL_*, TRADE_ACTION_*, SYMBOL_*, ACCOUNT_*, TRADE_RETCODE_* constants matching the real MT5 Python package.

**Not implemented:** `market_book_add/get/release`, `copy_rates_*`, `copy_ticks_*`, `orders_*`, `positions_*`, `history_*`

---

### MT5 Simulator Connectors

**File:** `pyeventbt/broker/mt5_broker/connectors/mt5_simulator_connector.py`

Four connector classes, all working exclusively with `SharedData`:

| Class | Interface | Methods |
|---|---|---|
| `PlatformConnector` | `IPlatform` | `initialize`, `login`, `shutdown`, `version`, `last_error` |
| `AccountInfoConnector` | `IAccountInfo` | `account_info()` → `SharedData.account_info` |
| `TerminalInfoConnector` | `ITerminalInfo` | `terminal_info()` → `SharedData.terminal_info` |
| `SymbolConnector` | `ISymbol` | `symbols_total`, `symbols_get`, `symbol_info`, `symbol_info_tick`, `symbol_select` |

---

### `LiveMT5Broker`

**File:** `pyeventbt/broker/mt5_broker/connectors/live_mt5_broker.py`

Initializes and manages the live MT5 terminal connection.

**Constructor:**
```python
LiveMT5Broker(symbol_list: list, config: Mt5PlatformConfig)
```

Initialization sequence:
1. `initialize_platformV2()` — calls `mt5.initialize()` with config credentials
2. `_live_account_warning()` — if DEMO: countdown 3s; if REAL: countdown 10s with warning
3. `_print_account_info()` — logs account details table
4. `_check_algo_trading_enabled()` — raises if `terminal_info().trade_allowed == False`
5. `_add_symbols_to_marketwatch(symbols)` — calls `mt5.symbol_select(symbol, True)` for each

**Methods:**
- `is_connected()` → `bool`: `mt5.terminal_info().connected`
- `is_closed()` → `bool`: `mt5.terminal_info() is None`

---

### Broker Entity Classes

All Pydantic BaseModels in `pyeventbt/broker/mt5_broker/core/entities/`:

**`AccountInfo`** — full MT5 account state:

| Key fields | Type |
|---|---|
| `login`, `leverage`, `limit_orders` | `int` |
| `balance`, `equity`, `margin`, `margin_free`, `profit` | `Decimal` |
| `currency`, `name`, `server`, `company` | `str` |
| `margin_mode` | `int` (0=netting, 1=exchange, 2=hedging) |
| `trade_allowed`, `trade_expert`, `fifo_close` | `bool` |

**`SymbolInfo`** — full MT5 symbol specification (all fields from MT5 API):

Key fields used by the framework:
- `digits: int` — decimal precision
- `volume_min`, `volume_max`, `volume_step: Decimal` — lot size constraints
- `trade_tick_size: Decimal` — minimum price movement
- `trade_contract_size: Decimal` — units per lot
- `currency_base`, `currency_profit`, `currency_margin: str` — currencies
- `point: Decimal` — point size (= tick size for most instruments)

**`TradePosition`** — open position matching MT5 `positions_get()` structure:

| Field | Type | Notes |
|---|---|---|
| `ticket` | `int` | Position ID |
| `time`, `time_msc` | `int` | Open time |
| `type` | `int` | 0=BUY, 1=SELL |
| `magic` | `int` | Strategy ID |
| `volume` | `Decimal` | |
| `price_open` | `Decimal` | Entry price |
| `sl`, `tp` | `Decimal` | |
| `price_current` | `Decimal` | Current price |
| `profit` | `Decimal` | Floating P&L |
| `symbol` | `str` | |
| `used_margin_acc_ccy` | `Optional[Decimal]` | Extra field for margin tracking |

**`TradeDeal`** — executed deal (matches MT5 `history_deals_get()`):

| Field | Type | Notes |
|---|---|---|
| `ticket`, `order`, `position_id` | `int` | |
| `type` | `int` | 0=buy, 1=sell, 2=balance, 3=credit |
| `entry` | `int` | 0=in, 1=out |
| `magic` | `int` | |
| `volume`, `price` | `Decimal` | |
| `commission`, `swap`, `fee`, `profit` | `Decimal` | |

**`TradeOrder`** — pending order matching MT5 `orders_get()` structure (ticket, type, state, price_open, sl, tp, etc.)

**`TradeRequest`** — order request matching MT5 `order_send()` input (action, magic, symbol, volume, price, sl, tp, type, etc.)

**`OrderSendResult`** — result of `order_send()`:
- `retcode: int` — `10009 = TRADE_RETCODE_DONE` (success)
- `deal: int`, `order: int` — ticket IDs
- `volume, price, bid, ask: Decimal`
- `request: TradeRequest` — the original request

**`Tick`** — tick data:
- `time: int`, `bid, ask, last, volume_real: Decimal`, `volume: int`, `time_msc: int`, `flags: int`

**`TerminalInfo`** — terminal state (`connected: bool`, `build: int`, `trade_allowed: bool`, etc.)

**`InitCredentials`** — login credentials (`path, login, password, server, timeout, portable`)

**`ClosedPosition`** (broker layer) — closed position with `ticket, symbol, direction, profit, commission, swap, sl, tp, time_entry/exit, price_entry/exit, magic`

---

### MT5 Broker Interfaces

**File:** `pyeventbt/broker/mt5_broker/core/interfaces/mt5_broker_interface.py`

Protocols defining the MT5 API surface:
- `IPlatform` — `initialize`, `login`, `shutdown`, `version`, `last_error`
- `IAccountInfo` — `account_info`
- `ITerminalInfo` — `terminal_info`
- `ISymbol` — `symbols_total`, `symbols_get`, `symbol_info`, `symbol_info_tick`, `symbol_select`
- `IMarketBook` — `market_book_add/get/release` (not implemented in simulator)
- `IMarketData` — `copy_rates_from/from_pos/range`, `copy_ticks_from/range` (not implemented in simulator)
- `IOrder` — `orders_total/get`, `order_calc_margin/profit/check/send`
- `IPosition` — `positions_total/get`
- `IHistory` — `history_orders/deals_total/get`

---

## 15. Schedule Service

**File:** `pyeventbt/schedule_service/schedule_service.py`

### `Schedule` (Pydantic BaseModel)

| Field | Type |
|---|---|
| `name` | `str` (repr of the callback function) |
| `is_active` | `bool` (default `True`) |
| `fn` | `Callable[[ScheduledEvent, Modules], None]` |
| `execute_every` | `StrategyTimeframes` |

Equality by `name` only.

---

### `Schedules`

Container for all registered schedules, indexed by `StrategyTimeframes`.

| Method | Description |
|---|---|
| `add_schedule(timeframe, callback)` | Creates and stores `Schedule`; returns it |
| `activate_schedule(schedule)` | Sets `is_active = True` |
| `deactivate_schedule(schedule)` | Sets `is_active = False` |
| `deactivate_all_schedules()` | Deactivates all |
| `activate_all_schedules()` | Activates all |
| `get_callbacks_to_execute_given_timeframe(tf)` | Returns `list[Callable]` for active schedules of that TF |

---

### `ScheduleService`

**Constructor:**
```python
ScheduleService(modules: Modules)
```

**`add_schedule(timeframe, callback)`:**
- Creates `Schedule` in `Schedules`
- Registers `timeframe` in `__timeframes_to_watch` dict

**`run_scheduled_callbacks(event: BarEvent)`:**
1. For each watched timeframe: checks if `current_timestamp - last_timestamp >= timeframe.to_timedelta()`
2. For all triggering timeframes: calls all active callbacks with a `ScheduledEvent`
3. Updates `last_timestamp` after each callback

**`deactivate_schedules()` / `activate_schedules()`** — delegates to `Schedules`.

---

### `TimeframeWatchInfo` (Pydantic BaseModel)

Tracks `last_timestamp` and `current_timestamp` per watched timeframe. Uses `arbitrary_types_allowed = True` for `pd.Timestamp`.

---

## 16. Hooks

**File:** `pyeventbt/hooks/hook_service.py`

### `Hooks` (str, Enum)

| Value | Trigger point |
|---|---|
| `ON_START` | Before the first bar is processed |
| `ON_SIGNAL_EVENT` | When `TradingDirector._handle_signal_event()` fires |
| `ON_ORDER_EVENT` | When `TradingDirector._handle_order_event()` fires |
| `ON_END` | After `TradingDirector.run()` completes |

Custom `__hash__` ensures hashability.

---

### `HookService`

| Method | Description |
|---|---|
| `add_hook(hook, callback)` | Appends `callback: Callable[[Modules], None]` to hook's list |
| `call_callbacks(hook, modules)` | Calls all registered callbacks for `hook` with `modules`; no-ops if disabled |
| `enable_hooks()` / `disable_hooks()` | Toggle `__hooks_enabled` flag |

---

## 17. Trade Archiver

**File:** `pyeventbt/trade_archiver/trade_archiver.py`

### `TradeArchiver`

In-memory store for all `FillEvent`s during a backtest.

**`archive_trade(fill_event: FillEvent)`:** stores with auto-incrementing integer key.

**Export methods:**

| Method | Returns | Columns |
|---|---|---|
| `export_historical_trades_dataframe()` | `pd.DataFrame` | TYPE, DEAL, SYMBOL, TIME_GENERATED, POSITION_ID, STRATEGY_ID, EXCHANGE, VOLUME, PRICE, SIGNAL_TYPE, COMMISSION, SWAP, FEE, GROSS_PROFIT, CCY |
| `export_historical_trades_json()` | `str` (JSON) | Same fields; Decimal as str (5dp); datetime as `"%Y-%m-%dT%H:%M:%S"` |
| `export_historical_trades_parquet(path)` | `None` | Decimal fields as Float64; zstd level 10 |
| `export_csv_trade_archive(path)` | `None` | CSV of DataFrame |

---

## 18. Backtest Results

**File:** `pyeventbt/backtest/core/backtest_results.py`

### `BacktestResults`

Returned by `Strategy.backtest()`.

**Constructor:**
```python
BacktestResults(backtest_pnl: pd.DataFrame, trades: pd.DataFrame)
```

| Property | Type | Content |
|---|---|---|
| `pnl` | `pd.DataFrame (float)` | Index: datetime, columns: `BALANCE`, `EQUITY` |
| `returns` | `pd.Series` | `pnl['EQUITY'].pct_change()` |
| `trades` | `pd.DataFrame` | Full trade archive from `TradeArchiver` |
| `backtest_pnl` | `pd.DataFrame` | Raw PnL (original Decimal types) |

**`plot()`:** matplotlib line chart of `EQUITY` and `BALANCE` with legend, small margins.

---

## 19. Indicators

**File:** `pyeventbt/indicators/indicators.py`

All indicators are static-method-only classes implementing `IIndicator`.
**Input:** `numpy.ndarray` (float64)
**Output:** `numpy.ndarray` or tuple of arrays
**Performance:** `@njit` (Numba JIT) used for all inner computation loops

**Import pattern:**
```python
from pyeventbt.indicators import SMA, KAMA, BollingerBands, ATR, RSI
# or
from pyeventbt.indicators.indicators import BollingerBands
```

---

### `KAMA` — Kaufman Adaptive Moving Average

```python
KAMA.compute(close: np.ndarray, n_period: int = 10, sc_fastest: float = 2/13, sc_slowest: float = 2/31) -> np.ndarray
```

**Algorithm:**
1. Efficiency Ratio: `ER = abs(close[i] - close[i-n]) / sum(abs(close[j] - close[j-1]))`
2. Smoothing Constant: `SC = (ER × (fast - slow) + slow)²`
3. `KAMA[i] = KAMA[i-1] + SC × (close[i] - KAMA[i-1])`

First `n_period - 1` values are `NaN`. Seeded from `close[n_period - 1]`.

---

### `SMA` — Simple Moving Average

```python
SMA.compute(close: np.ndarray, period: int) -> np.ndarray
```

Rolling arithmetic mean with a window of `period`.

---

### `BollingerBands`

```python
BollingerBands.compute(close: np.ndarray, period: int, std_dev: float) -> tuple[np.ndarray, np.ndarray, np.ndarray]
```

Returns `(upper, middle, lower)`:
- `middle = SMA(close, period)`
- `upper = middle + std_dev × rolling_std`
- `lower = middle - std_dev × rolling_std`

---

### `ATR` — Average True Range

```python
ATR.compute(high: np.ndarray, low: np.ndarray, close: np.ndarray, period: int) -> np.ndarray
```

True Range = `max(high-low, |high-prev_close|, |low-prev_close|)`
ATR = rolling mean of TR over `period`.

---

### `RSI` — Relative Strength Index

```python
RSI.compute(close: np.ndarray, period: int = 14) -> np.ndarray
```

Wilder's smoothing method. Output range 0–100. First `period` values are `NaN`.

---

### `IIndicator` (interface)

**File:** `pyeventbt/indicators/core/interfaces/indicator_interface.py`

```python
class IIndicator:
    @staticmethod
    def compute(*args, **kwargs) -> np.ndarray | tuple:
        raise NotImplementedError()
```

---

## 20. Configuration

**File:** `pyeventbt/config/configs.py`

### `Mt5PlatformConfig` (Pydantic BaseModel → `BaseConfig`)

For live trading only.

| Field | Type | Description |
|---|---|---|
| `path` | `str` | Absolute path to `terminal64.exe` |
| `login` | `int` | MT5 account number |
| `password` | `str` | Account password |
| `server` | `str` | Broker server name (e.g. `"Darwinex-Live"`) |
| `timeout` | `int` | Connection timeout in milliseconds |
| `portable` | `bool` | Whether MT5 is in portable mode |

---

## 21. Core Entities

**File:** `pyeventbt/core/entities/`

### `Variable` (Pydantic BaseModel)

| Field | Type |
|---|---|
| `name` | `str` |
| `value` | `float \| int` |

---

### `HyperParameter` (extends `Variable`)

| Field | Type | Description |
|---|---|---|
| `name` | `str` | |
| `value` | `float \| int` | Current value |
| `range` | `HyperParameterRange \| HyperParameterValues` | Search space definition |

**`HyperParameterRange`:**

| Field | Type | Default |
|---|---|---|
| `minimum` | `float \| int` | required |
| `maximum` | `float \| int` | required |
| `step` | `float \| int` | `1` |

**`HyperParameterValues`:**

| Field | Type |
|---|---|
| `values` | `list[float \| int]` |

Used to define parameters for the optimization engine (currently partially commented out; uses `hyperopt` in commented code).

---

## 22. Trading Context

**File:** `pyeventbt/trading_context/trading_context.py`

### `TypeContext` (str, Enum)

| Value | String |
|---|---|
| `LIVE` | `"LIVE"` |
| `BACKTEST` | `"BACKTEST"` |

Module-level global `trading_context = TypeContext.BACKTEST`.

Functions:
- `get_trading_context() → TypeContext`
- `set_trading_context(context: TypeContext) → None`

**Note:** The `Modules.TRADING_CONTEXT` field (which users access) is a `TypeContext` instance. It compares equal to its string value (since `TypeContext` is `str, Enum`).

---

## 23. Utilities

**File:** `pyeventbt/utils/utils.py`

### `TerminalColors`

ANSI escape codes as class attributes: `HEADER`, `OKBLUE`, `OKCYAN`, `OKGREEN`, `WARNING`, `FAIL`, `ENDC`, `BOLD`, `UNDERLINE`.

**`colorize(string, color) → str`** — wraps string in ANSI color codes.

---

### `LoggerColorFormatter` (logging.Formatter)

Custom formatter applied to the `pyeventbt` logger:

| Level | Color |
|---|---|
| DEBUG | Grey |
| INFO | Cyan |
| WARNING | Yellow |
| ERROR | Red bold underlined |
| CRITICAL | Red bold underlined |

Format: `"%(asctime)s - %(levelname)s: %(message)s"`

---

### `Utils` (static methods only)

**`order_type_str_to_int(order_type: str) → int`** (cached via `@lru_cache`):

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

**`order_type_int_to_str(order_type: int) → str`** (cached) — reverse mapping.

**`convert_currency_amount_to_another_currency(amount, from_ccy, to_ccy, data_provider) → Decimal`**:
Finds the FX symbol containing both currencies, uses `DataProvider.DATA_PROVIDER.get_latest_bid()`, converts.

**`get_currency_conversion_multiplier_cfd(from_ccy, to_ccy, data_provider) → Decimal`**:
Returns the multiplier (not the converted amount) for CFD currency conversion.

**`get_fx_futures_suffix(symbol) → tuple[str, str]`**:
Returns `(current_contract, next_contract)` suffix pair based on current month:
- Jan–Mar: `(H, M)`, Apr–Jun: `(M, U)`, Jul–Sep: `(U, Z)`, Oct–Dec: `(Z, H)`

**`convert_currency_amount_to_another_currency_futures(amount, from_ccy, to_ccy, data_provider) → Decimal`**:
For futures: one currency must be USD; maps to CME FX futures contracts (6A=AUD, 6B=GBP, 6E=EUR, 6J=JPY, etc.)

**`dateprint() → str`**:
Returns current time in MT5 server timezone (America/New_York + 7h = Asia/Nicosia with US DST).

**`cap_forecast(forecast: float) → float`**:
Clamps value to `[-20.0, 20.0]`.

**`check_new_m1_bar_creates_new_tf_bar(latest_bar_time, timeframe) → bool`** (legacy):
Adds 1 minute to the timestamp, then checks if it's a multiple of the timeframe in seconds. Supports: `1min`, `5min`, `15min`, `30min`, `1H`, `4H`, `1D`.

---

### `check_platform_compatibility(raise_exception: bool = True) → bool`

Module-level function. Checks if the OS is Windows:
- `raise_exception=True` (default): raises `Exception` on non-Windows
- `raise_exception=False`: logs a warning and returns `False`

Used in sizing engines and live data connector to gracefully handle non-Windows environments.

---

### `print_percentage_bar(percentage, bar_length=50, additional_message='', end='\r')`

Prints a text progress bar: `[████████----------] 40.00% (message)`.

---

## 24. CSV Data Format

Files must be named exactly `{SYMBOL}.csv` (e.g. `EURUSD.csv`).
**No header row.** Comma-separated.

```
2020.01.02,00:00:00,1.11808,1.11808,1.11808,1.11808,1,0,15
2020.01.02,00:01:00,1.11808,1.11823,1.11808,1.11820,3,0,14
```

| Column | Format | Description |
|---|---|---|
| date | `YYYY.MM.DD` | Bar date |
| time | `HH:MM:SS` | Bar time (MT5 server time) |
| open | float | Open price |
| high | float | High price |
| low | float | Low price |
| close | float | Close price |
| tickvol | int | Tick volume |
| volume | int | Real volume (often 0 for FX) |
| spread | int | Spread in **points** (not pips) |

This is the default MT5 history export format. A bundled `EURUSD.csv` covering 2020–2023 is included at:
`pyeventbt/data_provider/connectors/historical_csv_data/EURUSD.csv`

---

## 25. Event Loop — Full Execution Flow

### Backtest initialization sequence

```
Strategy.backtest()
  │
  ├─ sort strategy_timeframes ascending
  ├─ CSVBacktestDataConfig(csv_path, account_currency, symbols, base_tf, timeframes, start, end)
  ├─ DataProvider(queue, config, BACKTEST)
  │    └─ CSVDataProvider(config)
  │         └─ loads CSVs → M1 → aligns → resamples → generators
  │
  ├─ MT5SimulatedExecutionConfig(balance, currency, leverage, magic)
  ├─ ExecutionEngine(queue, data_provider, config)
  │    └─ Mt5SimulatorExecutionEngineConnector(config, queue, dp)
  │         └─ sets SharedData.account_info.balance, currency, leverage
  │
  ├─ Portfolio(balance, execution_engine, BACKTEST, base_tf)
  │
  ├─ Modules(BACKTEST, DataProvider, ExecutionEngine, Portfolio)
  │
  ├─ SignalEngineService(queue, modules, signal_config)
  ├─ SizingEngineService(queue, modules, sizing_config)
  ├─ RiskEngineService(queue, risk_config, modules)
  │
  ├─ MT5BacktestSessionConfig(start_date, capital, name)
  ├─ PortfolioHandler(queue, sizing, risk, portfolio, base_tf, results_dir)
  │
  └─ TradingDirector(queue, signal_svc, portfolio_handler, session_config, modules, ...)
       └─ run() → _run_backtest()
```

### Per-bar event chain

```
queue is empty
→ DataProvider.update_bars()
   → CSVDataProvider generator: next(symbol_generator)
   → emits BarEvent(symbol, datetime, Bar, base_timeframe)
   → for each higher TF where boundary crossed: emits BarEvent(symbol, datetime, Bar, higher_tf)
   → puts all BarEvents onto queue

queue has BarEvent
→ TradingDirector._handle_bar_event(event)
   ├─ PortfolioHandler.process_bar_event(event)
   │    └─ (only for base TF) Portfolio._update_portfolio(event)
   │         └─ ExecutionEngine._update_values_and_check_executions_and_fills(event)
   │              ├─ mark all open positions to market
   │              ├─ check SL/TP hits → close position → emit FillEvent(OUT)
   │              └─ check pending order triggers → fill → emit FillEvent(IN)
   │
   ├─ ScheduleService.run_scheduled_callbacks(event)
   │    └─ for each registered @run_every timeframe that elapsed:
   │         fn(ScheduledEvent(timeframe, symbol, ts, former_ts), modules)
   │
   └─ SignalEngineService.generate_signal(event)
        └─ user's strategy function(event, modules) → [SignalEvent, ...]
             └─ each SignalEvent put on queue

queue has SignalEvent
→ TradingDirector._handle_signal_event(event)
   ├─ HookService.call_callbacks(ON_SIGNAL_EVENT, modules)
   └─ PortfolioHandler.process_signal_event(event)
        ├─ SizingEngineService.get_suggested_order(signal) → SuggestedOrder
        └─ RiskEngineService.assess_order(suggested_order)
             └─ if volume > 0: create OrderEvent → put on queue

queue has OrderEvent
→ TradingDirector._handle_order_event(event)
   ├─ ExecutionEngine._process_order_event(event)
   │    ├─ MARKET → _send_market_order → creates TradePosition in SharedData → emit FillEvent(IN)
   │    └─ STOP/LIMIT → _send_pending_order → creates TradeOrder in SharedData
   └─ HookService.call_callbacks(ON_ORDER_EVENT, modules)

queue has FillEvent
→ TradingDirector._handle_fill_event(event)
   └─ PortfolioHandler.process_fill_event(event)
        └─ TradeArchiver.archive_trade(fill_event)

end of data (StopIteration on all generators)
→ DataProvider.close_positions_end_of_data = True
→ TradingDirector: close all open positions → process resulting FillEvents
→ DataProvider.continue_backtest = False
→ exit loop
→ PortfolioHandler.process_backtest_end(name, export_csv, export_parquet)
   ├─ Portfolio._update_portfolio_end_of_backtest()
   ├─ (export files if requested)
   └─ return BacktestResults(pnl_df, trades_df)
```

---

## 26. Architecture Constraints and Rules

1. **`strategy_id` must be a numeric string.** It is passed directly to `int(strategy_id)` to become the MT5 magic number. Non-numeric strings will raise at runtime.

2. **`base_timeframe` must be the first and smallest element of `strategy_timeframes`.** The `Strategy` class sorts timeframes ascending before passing to the data provider, which validates that `timeframes_list[0] == base_timeframe`.

3. **Only `USD`, `EUR`, `GBP` are supported as account currencies in backtest.** `CSVDataProvider._check_account_currency_is_supported()` raises otherwise.

4. **`RiskPctSizingConfig` requires `SignalEvent.sl != 0`.** The sizing engine raises `Exception` if `sl == 0`.

5. **Bar data gaps are filled with phantom bars.** Missing timestamps in a symbol's CSV are filled with OHLC from the prior close and `tickvol=volume=spread=1`. These bars are skipped during `update_bars()` to avoid false signals.

6. **Higher-TF lookahead prevention.** `get_latest_bars()` for non-base timeframes returns bars up to and excluding the currently forming bar (second-to-last). `get_latest_tick()` uses the next bar's open as bid to avoid using data the strategy doesn't yet have.

7. **Integer price encoding.** `Bar` stores prices as `int` (= `float × 10^digits`). Reconstruct with `.close_f` or `value / 10**digits`. The `digits` field is obtained from `Mt5SimulatorWrapper.symbol_info(symbol).digits`.

8. **Portfolio updates only on base-TF bars.** `PortfolioHandler.process_bar_event()` returns early for higher-TF events to avoid redundant P&L recalculations. Historical time-series is recorded only for the first-seen symbol.

9. **Auxiliary cross-rate symbols are loaded transparently.** When trading GBPJPY with a USD account, `CSVDataProvider` automatically loads USDJPY (or JPYUSD) for P&L conversion. These symbols must have CSV files in `csv_dir`.

10. **MT5 Python package is optional.** All imports of `MetaTrader5` are wrapped in `try/except ImportError`. The package works fully on macOS and Linux in backtest mode.

11. **Live trading requires Windows.** `check_platform_compatibility()` raises on non-Windows. This is checked before any `import MetaTrader5` attempt.

12. **Hedging account mode assumed.** `SharedData.default_account_info.yaml` sets `margin_mode: 2` (hedging), meaning multiple positions in opposite directions on the same symbol are allowed simultaneously.

13. **`SharedData` uses class-level attributes** (not instance attributes). This is a singleton pattern — all code that imports `SharedData` shares the same state. The constructor mutates class variables.

14. **`StrategyTimeframes` string comparison.** `event.timeframe` in `BarEvent` is a plain `str`. Comparing it to `StrategyTimeframes.ONE_HOUR` works because `StrategyTimeframes.__eq__` handles `str` inputs. Always use this pattern rather than `.value` access.

15. **`Modules` is a Pydantic model with `arbitrary_types_allowed = True`** to support `IDataProvider`, `IExecutionEngine`, and `IPortfolio` interface types.

16. **Rollover support.** `SignalEvent.rollover` tuple `(True, "old_contract", "new_contract")` is passed through to the execution engine to handle futures contract rollovers. Currently propagated but the rollover logic is in the execution connectors.

---

## 27–31. Extended Documentation (Split into Separate Files)

The following sections have been extracted into standalone documents for easier navigation:

| Section | File | Description |
|---|---|---|
| **27. Core Design Pattern** | [`docs/design_pattern.md`](docs/design_pattern.md) | The event-driven pattern explained, integration guide, minimal skeleton code |
| **28. Architecture Limitations & Migration** | [`docs/distributed_migration.md`](docs/distributed_migration.md) | Coupling analysis, 5-step migration path, distributed target architecture |
| **29. Contracts, Behaviors & Protocols** | [`docs/contracts_protocols.md`](docs/contracts_protocols.md) | Formal specs for every component, multi-broker/multi-provider patterns, checklists |
| **30. Technical Review** | [`docs/technical_review.md`](docs/technical_review.md) | All known issues by severity, dependency analysis, prioritized fix backlog |
| **31. Industry Research** | [`docs/industry_research.md`](docs/industry_research.md) | LMAX, NautilusTrader, LEAN, framework comparison, process separation patterns |

See also:
- [`docs/event_flow_diagram.md`](docs/event_flow_diagram.md) — 10 mermaid diagrams covering the full event pipeline
- [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) — Functional requirements and known gaps (sections 19–21)
- [`docs/EXAMPLES_COOKBOOK.md`](docs/EXAMPLES_COOKBOOK.md) — 17 practical strategy patterns
