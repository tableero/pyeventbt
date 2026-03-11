# PyEventBT — Comprehensive Requirements and Functional Specification

> Synthesized from all documentation files in `/docs/`. Every section exhaustively details all
> discovered information. Intended for cross-project comparison.

---

## 1. Project Overview

### 1.1 Identity

| Field | Value |
|---|---|
| **Name** | pyeventbt |
| **Version** | 0.0.4 |
| **License** | Apache-2.0 |
| **Authors** | Marti Castany, Alain Porto |
| **Homepage** | https://github.com/marticastany/pyeventbt |
| **Development Status** | 1 - Planning (PyPI classifier, despite functional code) |
| **Python Requirement** | ^3.12 |

### 1.2 Purpose and Positioning

PyEventBT is an **event-driven backtesting and live trading framework** for Python and MetaTrader 5.
Its core design goal is to let users write a single strategy that runs identically in simulation (CSV
data) and in live MT5 trading. The shared `queue.Queue` event bus is the unifying abstraction.

### 1.3 Supported Platforms

- **Backtest mode**: Any platform (Linux, macOS, Windows) with Python 3.12+. MT5 installation is not
  required.
- **Live trading mode**: **Windows only** (MetaTrader 5 Python package only runs on Windows). The
  package handles import failures gracefully via `try/except ImportError` so it loads on non-Windows
  platforms.

### 1.4 Dependencies

| Package | Version Constraint | Role |
|---|---|---|
| python-dotenv | ^1.1.1 | Environment variable loading |
| pydantic | ^2.12.3 | Data validation and configuration models |
| numpy | ^2.0.0 | Numerical computation, indicator calculations |
| polars | ^1.35.0 | DataFrame operations for bar data |
| pandas | ^2.2.3 | Timestamp handling, time series utilities, legacy exports |
| numba | ^0.62.1 | JIT compilation for indicator performance |
| matplotlib | ^3.7.0 | Backtest result plotting |
| scipy | ^1.10.0 | Scientific computing (usage largely unclear from documented modules) |
| scikit-learn | ^1.3.0 | Machine learning (usage largely unclear from documented modules) |
| PyYAML | ^6.0 | YAML config file serialization for BaseConfig |

### 1.5 CLI Entry Point

```
pyeventbt = pyeventbt.app:main
```

Supports `--version` flag and `info` command. Defined in `[tool.poetry.scripts]`.

### 1.6 Example Files

| File | Demonstrates |
|---|---|
| `example_ma_crossover.py` | Core decorator API, SMA fast/slow crossover on daily bars, `MinSizingConfig`, `PassthroughRiskConfig`, `backtest()`, `backtest.plot()` |
| `example_bbands_breakout.py` | Multi-timeframe (`ONE_HOUR` + `ONE_DAY`), `BollingerBands`, STOP order types (`BUY_STOP`/`SELL_STOP`), intraday scheduling (order at 08:00, close at 21:00), position management via `modules.EXECUTION_ENGINE` |
| `example_quantdle_ma_crossover.py` | `QuantdleDataUpdater` with API key/ID, `updater.update_data()`, same MA crossover strategy on Quantdle-sourced data |

---

## 2. Architecture Overview

### 2.1 Paradigm

**Event-driven architecture** using a shared `queue.Queue` as the inter-component event bus. All
components communicate exclusively through typed event objects. No direct method calls between
components except through the `TradingDirector` dispatcher.

### 2.2 Package Layout

```
pyeventbt/
  __init__.py          -- Public API surface (re-exports)
  app.py               -- CLI
  config/              -- Mt5PlatformConfig, BaseConfig (YAML-serializable)
  core/                -- Shared entities: HyperParameter, Variable
  utils/               -- Utilities: date formatting, currency conversion, logging
  strategy/            -- Strategy facade, Modules, StrategyTimeframes, VerboseLevel, WalkForward
  events/              -- BarEvent, SignalEvent, OrderEvent, FillEvent, ScheduledEvent, enums
  indicators/          -- KAMA, SMA, EMA, ATR, RSI, ADX, Momentum, BollingerBands, DonchianChannels, MACD, KeltnerChannel
  risk_engine/         -- IRiskEngine, PassthroughRiskEngine, RiskEngineService, configs
  sizing_engine/       -- ISizingEngine, MT5MinSizing, MT5FixedSizing, MT5RiskPctSizing, SizingEngineService, configs
  data_provider/       -- IDataProvider, CSVDataProvider, Mt5LiveDataProvider, DataProvider service, QuantdleDataUpdater, configs
  portfolio/           -- IPortfolio, Portfolio, OpenPosition, ClosedPosition, PendingOrder
  portfolio_handler/   -- PortfolioHandler, SuggestedOrder
  backtest/            -- BacktestResults
  trading_director/    -- TradingDirector, session configs
  execution_engine/    -- IExecutionEngine, Mt5SimulatorExecutionEngineConnector, Mt5LiveExecutionEngineConnector, ExecutionEngine service, configs
  broker/              -- Mt5SimulatorWrapper, SharedData, connectors, entities (SymbolInfo, AccountInfo, etc.)
  signal_engine/       -- ISignalEngine, SignalMACrossover, SignalPassthrough, SignalEngineService, configs
  hooks/               -- Hooks enum, HookService
  schedule_service/    -- ScheduleService, Schedule, Schedules, TimeframeWatchInfo
  trade_archiver/      -- TradeArchiver, ITradeArchiver
  trading_context/     -- TypeContext enum (BACKTEST / LIVE)
```

Each subpackage follows the internal convention:
- `core/interfaces/` — abstract base classes
- `core/entities/` — data models
- `core/configurations/` — config models
- `services/` — business logic
- `connectors/` — external system adapters

### 2.3 Main Loop Flow — Step-by-Step

**Backtest loop** (`TradingDirector._run_backtest`):
1. `ON_START` hook fires.
2. If `run_schedules=False`, all schedules are deactivated.
3. Loop begins:
   a. Try non-blocking dequeue from `events_queue`.
   b. On `queue.Empty`: call `DATA_PROVIDER.update_bars()` which pushes next bar(s) to queue.
   c. On valid `BarEvent`: dispatch to `_handle_bar_event(event)`.
      - `PortfolioHandler.process_bar_event(event)` (updates portfolio state via execution engine)
      - `ScheduleService.run_scheduled_callbacks(event)` (fires time-based callbacks)
      - `SignalEngineService.generate_signal(event)` (may push `SignalEvent` to queue)
   d. On `SignalEvent`: dispatch to `_handle_signal_event(event)`.
      - `HookService.call_callbacks(ON_SIGNAL_EVENT, modules)`
      - `PortfolioHandler.process_signal_event(event)` — runs sizing engine then risk engine;
        risk engine pushes `OrderEvent` to queue if approved.
   e. On `OrderEvent`: dispatch to `_handle_order_event(event)`.
      - `ExecutionEngine._process_order_event(event)` — executes trade; pushes `FillEvent` to queue.
      - `HookService.call_callbacks(ON_ORDER_EVENT, modules)`
   f. On `FillEvent`: dispatch to `_handle_fill_event(event)`.
      - `PortfolioHandler.process_fill_event(event)` — archives trade via TradeArchiver.
   g. On `None` event: log warning.
   h. If `close_positions_end_of_data` and open positions exist: close them; continue loop to process
      resulting FillEvents.
   i. If no open positions and queue is empty: set `continue_backtest = False` and exit loop.
4. `_handle_backtest_end()` called — finalizes portfolio, exports results.
5. `ON_END` hook fires.
6. Return `BacktestResults`.

**Live loop** (`TradingDirector._run_live_trading`):
1. `ON_START` hook fires.
2. If `run_schedules=False`, all schedules are deactivated.
3. Infinite loop:
   a. Try non-blocking dequeue from `events_queue`.
   b. On `queue.Empty`: call `DATA_PROVIDER.update_bars()` (polls MT5 for new bars).
   c. Dispatch events identically to backtest loop.
   d. Sleep for `heartbeat` seconds (default 0.1s).
4. Loop never exits; `ON_END` hook never fires in normal operation.

### 2.4 Queue-Based Communication

| Put by | Event type | Consumed by |
|---|---|---|
| `CSVDataProvider` / `Mt5LiveDataProvider` (via `update_bars()`) | `BarEvent` | `TradingDirector._handle_bar_event` |
| `SignalEngineService.generate_signal()` | `SignalEvent` | `TradingDirector._handle_signal_event` |
| `RiskEngineService._create_and_put_order_event()` | `OrderEvent` | `TradingDirector._handle_order_event` |
| `ExecutionEngine` connectors | `FillEvent` | `TradingDirector._handle_fill_event` |

`ScheduledEvent` is **not queued** — it is created inline by `ScheduleService` and passed directly
to callbacks. `TradingDirector.event_handlers_dict` has no entry for `EventType.SCHEDULED_EVENT`.

### 2.5 Backtest vs Live Mode Differences

| Aspect | Backtest | Live |
|---|---|---|
| Data source | `CSVDataProvider` (CSV files) | `Mt5LiveDataProvider` (MT5 API polling) |
| Data config | `CSVBacktestDataConfig` | `MT5LiveDataConfig` |
| Execution engine | `Mt5SimulatorExecutionEngineConnector` | `Mt5LiveExecutionEngineConnector` |
| Execution config | `MT5SimulatedExecutionConfig` | `MT5LiveExecutionConfig` |
| Session config | `MT5BacktestSessionConfig` | `MT5LiveSessionConfig` |
| Trading context | `TypeContext.BACKTEST` | `TypeContext.LIVE` |
| Loop termination | When all CSV generators exhausted + no open positions | Never (runs until process killed) |
| End-of-data handling | Closes all open positions, then finalizes results | N/A |
| `ON_END` hook | Fires | Does not fire in normal operation |
| Account state | Simulated in-memory (`SharedData`) | Real MT5 terminal |
| P&L calculation | Simulated with spread/swap/commission | Real broker fills |
| Results | `BacktestResults` returned | `None` |

---

## 3. Functional Requirements — Data Ingestion

### 3.1 CSV Format Specification

- **Headerless** — no column names row.
- **Exactly 9 columns** (by position):
  1. `date` — `YYYY.MM.DD`
  2. `time` — `HH:MM:SS`
  3. `open` — float price
  4. `high` — float price
  5. `low` — float price
  6. `close` — float price
  7. `tickvol` — integer
  8. `volume` — integer
  9. `spread` — integer
- **One file per symbol**, named `{SYMBOL}.csv` (implied by directory-based lookup).
- **Scanned lazily** via Polars `pl.scan_csv`, collected after filtering.
- Datetime is parsed from `date` + `time` columns combined.

### 3.2 Supported Timeframes (Full List)

From `StrategyTimeframes(str, Enum)` — values are pandas-compatible frequency codes:

| Member | Value | Timedelta |
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
| `ONE_WEEK` | `'1W'` | 7 days |
| `ONE_MONTH` | `'1M'` | 30 days (approximation) |
| `SIX_MONTH` | `'6M'` | 180 days (approximation) |
| `ONE_YEAR` | `'12M'` | 365 days (approximation) |

Total: **22 distinct timeframes**.

Timeframe string-to-Polars duration mapping (`_timeframe_to_duration`):
`"5min"` → `"5m"`, `"1H"` → `"1h"`, `"1D"` → `"1d"`, `"1W"` → `"1w"`, `"1M"` → `"1mo"`.

### 3.3 Multi-Symbol Support

- `tradeable_symbols: list[str]` — user-specified symbols.
- `symbol_list: list[str]` — tradeable + auxiliary symbols.
- All symbols are loaded and their M1 bars aligned on a **common master datetime index**.
- The master index starts at the **latest first date** across all symbols (to ensure all symbols have
  data from the start date).
- Bar events are emitted only for tradeable symbols, not auxiliary ones.

### 3.4 Auxiliary Symbol Auto-Detection

- Purpose: currency conversion for FX cross-pair margin/profit currency to account currency.
- `_create_auxiliary_symbol_list` inspects `mt5.symbol_info(symbol).currency_margin` and
  `currency_profit` for each tradeable symbol.
- If margin or profit currency differs from account currency, the FX cross pair (e.g., `USDEUR`)
  is added to the auxiliary list.
- A static list of 33 FX pairs is hardcoded as the reference universe. Any currency pair outside
  this list cannot be added dynamically.
- Supported account currencies: `"USD"`, `"EUR"`, `"GBP"` only.

### 3.5 Timeframe Resampling Rules

Polars aggregation map (`_AGG_MAP`):

| Column | Aggregation |
|---|---|
| `open` | `pl.first` |
| `high` | `pl.max` |
| `low` | `pl.min` |
| `close` | `pl.last` |
| `tickvol` | `pl.sum` |
| `volume` | `pl.sum` |
| `spread` | `pl.first` |

- Raw CSV data is first resampled to M1 via `group_by_dynamic`.
- Higher timeframes are then produced from the M1 data via further `group_by_dynamic` passes.
- Weekly bars receive a `- pl.duration(days=7)` datetime offset shift.

### 3.6 Gap-Filling Rules

- After aligning all symbols to the master datetime index, gaps are forward-filled:
  - `close` is forward-filled from the previous bar's close.
  - `open`, `high`, `low` are derived from the filled close (set equal to close).
  - `tickvol`, `volume`, `spread` are all set to `1` to mark the bar as synthetic/gap-filled.
- Synthetic bars (where `tickvol == volume == spread == 1`) are **skipped** during `update_bars()`
  and do not generate bar events.

### 3.7 Lookahead Bias Prevention

- `get_latest_tick`: Uses the **opening price of the next unfinished bar** (first bar after current
  simulation time) as bid. This prevents using the current bar's close as a fill price.
- `get_latest_bar` for higher timeframes: Returns the **second-to-last** bar (not the most recent)
  to avoid returning the currently-forming incomplete bar.
- `get_latest_bars` for higher timeframes: Also excludes the most recent forming bar.
- At end of backtest, `get_latest_tick` falls back to the last bar's close.

### 3.8 Live MT5 Data Pull

- Uses `mt5.copy_rates_from_pos(symbol, timeframe, from_pos=1, count=N)`.
- `from_pos=1` fetches bars starting from the last **completed** bar (not the currently-forming bar).
- Polls all symbol × timeframe combinations in `update_bars()`.
- Tracks last seen bar datetime in `last_bar_tf_datetime[symbol][timeframe]`.
- Only emits a `BarEvent` when bar datetime > last seen datetime.
- **Bug**: `update_bars()` calls `get_latest_bar()` twice per new bar (once to check datetime,
  once to build the event) — the first result is discarded unnecessarily.
- Heuristic digits fallback if `symbol_info` unavailable: 3 digits for JPY pairs, 5 otherwise.

### 3.9 Timeframe Mapping for MT5 Live (mt5_live_data_connector)

| String | MT5 Constant |
|---|---|
| `"1min"` | `mt5.TIMEFRAME_M1` |
| `"5min"` | `mt5.TIMEFRAME_M5` |
| `"15min"` | `mt5.TIMEFRAME_M15` |
| `"30min"` | `mt5.TIMEFRAME_M30` |
| `"1h"` / `"1H"` | `mt5.TIMEFRAME_H1` |
| `"4h"` / `"4H"` | `mt5.TIMEFRAME_H4` |
| `"1d"` / `"1D"` | `mt5.TIMEFRAME_D1` |
| `"1w"` / `"1W"` | `mt5.TIMEFRAME_W1` |
| `"1M"` | `mt5.TIMEFRAME_MN1` |

### 3.10 `get_latest_bars` Return Format

- Returns a Polars DataFrame with **float** prices (not integer-scaled).
- Columns: `[datetime, open, high, low, close, tickvol, volume, spread]`.
- User converts to numpy: `.select('close').to_numpy().flatten()`.

---

## 4. Functional Requirements — Event System

### 4.1 EventType Enum

```python
class EventType(str, Enum):
    BAR           = "BAR"
    SIGNAL        = "SIGNAL"
    ORDER         = "ORDER"
    FILL          = "FILL"
    SCHEDULED_EVENT = "SCHEDULED_EVENT"
```

### 4.2 Supporting Enums

**SignalType**:
- `BUY = "BUY"`
- `SELL = "SELL"`

**OrderType**:
- `MARKET = "MARKET"` — execute at current market price
- `LIMIT = "LIMIT"` — execute at specified price or better
- `STOP = "STOP"` — execute when price reaches a stop level
- `CONT = "CONT"` — continuation/rollover order

**DealType**:
- `IN = "IN"` — position entry
- `OUT = "OUT"` — position exit

### 4.3 EventBase

```python
class EventBase(BaseModel):
    type: EventType
```

Base for all events. Enables `arbitrary_types_allowed` via Pydantic config.

### 4.4 Bar Dataclass

```python
@dataclass(slots=True)
class Bar:
    open:    int
    high:    int
    low:     int
    close:   int
    tickvol: int
    volume:  int
    spread:  int
    digits:  int
```

- All prices stored as integers. Float value = `price / 10 ** digits`.
- Uses `__slots__` for memory efficiency (~56 bytes per instance).
- Cached `price_factor` property (`10 ** digits`) computed on first access.
- Float conversion properties: `open_f`, `high_f`, `low_f`, `close_f`, `spread_f`.

### 4.5 BarEvent

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.BAR` | Discriminator |
| `symbol` | `str` | required | Trading instrument symbol |
| `datetime` | `datetime` | required | Bar timestamp |
| `data` | `Bar` | required | Integer-scaled price payload |
| `timeframe` | `str` | required | Timeframe string (e.g., `"M1"`, `"H1"`) |

### 4.6 SignalEvent

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.SIGNAL` | Discriminator |
| `symbol` | `str` | required | Trading instrument |
| `time_generated` | `datetime` | required | When signal was generated |
| `strategy_id` | `str` | required | Numeric string → MT5 magic number |
| `forecast` | `Optional[float]` | `0.0` | Signal strength; intended range -20 to +20 (undocumented in code) |
| `signal_type` | `SignalType` | required | `BUY` or `SELL` |
| `order_type` | `OrderType` | required | `MARKET`, `LIMIT`, `STOP`, or `CONT` |
| `order_price` | `Optional[Decimal]` | `Decimal('0.0')` | Limit/stop price (ignored for MARKET) |
| `sl` | `Optional[Decimal]` | `Decimal('0.0')` | Stop-loss price |
| `tp` | `Optional[Decimal]` | `Decimal('0.0')` | Take-profit price |
| `rollover` | `Optional[tuple]` | `(False, "", "")` | `(needs_rollover: bool, original_contract: str, new_contract: str)` |

### 4.7 OrderEvent

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.ORDER` | Discriminator |
| `symbol` | `str` | required | Trading instrument |
| `time_generated` | `datetime` | required | Timestamp of order creation |
| `strategy_id` | `str` | required | Numeric string strategy ID |
| `volume` | `Decimal` | required | Position size in lots (from sizing engine) |
| `signal_type` | `SignalType` | required | `BUY` or `SELL` |
| `order_type` | `OrderType` | required | `MARKET`, `LIMIT`, `STOP`, or `CONT` |
| `order_price` | `Optional[Decimal]` | `Decimal('0.0')` | Limit/stop price |
| `sl` | `Optional[Decimal]` | `Decimal('0.0')` | Stop-loss price |
| `tp` | `Optional[Decimal]` | `Decimal('0.0')` | Take-profit price |
| `rollover` | `Optional[tuple]` | `(False, "", "")` | Rollover info (same as SignalEvent) |
| `buffer_data` | `Optional[dict]` | `None` | Arbitrary extra data passed to execution engine |

### 4.8 FillEvent

| Field | Type | Description |
|---|---|---|
| `type` | `EventType` | `EventType.FILL` |
| `deal` | `DealType` | `IN` (entry) or `OUT` (exit) |
| `symbol` | `str` | Trading instrument |
| `time_generated` | `datetime` | Fill timestamp |
| `position_id` | `int` | Broker-assigned position ID |
| `strategy_id` | `str` | Strategy identifier |
| `exchange` | `str` | Exchange or broker name |
| `volume` | `Decimal` | Filled volume |
| `price` | `Decimal` | Execution price |
| `signal_type` | `SignalType` | Direction of the original signal |
| `commission` | `Decimal` | Commission charged |
| `swap` | `Decimal` | Swap cost |
| `fee` | `Decimal` | Additional fees |
| `gross_profit` | `Decimal` | Gross profit (meaningful for exit fills) |
| `ccy` | `str` | Currency denomination for all cost/profit fields |

Live connector maps: `entry == 0` → `DealType.IN`, any other value → `DealType.OUT`.

### 4.9 ScheduledEvent

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `EventType` | `EventType.SCHEDULED_EVENT` | Not user-settable (uses `init_var=False`) |
| `schedule_timeframe` | `StrategyTimeframes` | required | The timeframe that triggered this event |
| `symbol` | `str` | required | Symbol from the triggering `BarEvent` |
| `timestamp` | `pd.Timestamp` | required | Current execution timestamp |
| `former_execution_timestamp` | `pd.Timestamp | None` | `None` | Timestamp of previous execution of this schedule |

Note: `ScheduledEvent` is passed directly to callbacks; it does not go through the queue.

---

## 5. Functional Requirements — Strategy API

### 5.1 Strategy Class Constructor

```python
Strategy(logging_level: VerboseLevel = VerboseLevel.INFO)
```

Sets up `pyeventbt` logger (`propagate=False`) with `LoggerColorFormatter` at the specified level.

### 5.2 Decorator API

#### `@strategy.custom_signal_engine(strategy_id='default', strategy_timeframes=[StrategyTimeframes.ONE_MIN])`

Registers a user-defined signal engine function.

- Decorated function signature: `fn(bar_event: BarEvent, modules: Modules) -> SignalEvent | list[SignalEvent] | None`
- Only the first registration per `strategy_id` is kept (uses `dict.setdefault()`).
- Appends new timeframes to the internal strategy timeframe list.

#### `@strategy.custom_sizing_engine(strategy_id='default')`

Registers a user-defined sizing engine function.

- Decorated function signature: `fn(signal_event: SignalEvent, modules: Modules) -> SuggestedOrder | list[SuggestedOrder] | None`

#### `@strategy.custom_risk_engine(strategy_id='default')`

Registers a user-defined risk engine function.

- Decorated function signature: `fn(suggested_order: SuggestedOrder, modules: Modules) -> float`
- Return value is the approved volume. Volume ≤ 0 means order is rejected.

#### `@strategy.run_every(interval: StrategyTimeframes)`

Registers a scheduled callback to run at a given timeframe interval.

- Decorated function signature: `fn(scheduled_event: ScheduledEvent, modules: Modules) -> None`
- Appends interval to `__strategy_timeframes` if not already present.
- Multiple functions can be registered per interval.

#### `@strategy.hook(hook: Hooks)`

Registers a lifecycle hook callback.

- Decorated function signature: `fn(modules: Modules) -> None`

**Important**: All decorators call `dict.setdefault()` or `hooks.add_hook()` but **do not return `fn`**.
The original function is replaced with `None` after decoration — it cannot be called outside the
decorator context.

### 5.3 Predefined Engine Configuration Methods

```python
strategy.configure_predefined_signal_engine(conf: MACrossoverConfig, strategy_timeframes: list[StrategyTimeframes] = [StrategyTimeframes.ONE_MIN])
strategy.configure_predefined_sizing_engine(conf: MinSizingConfig | RiskPctSizingConfig | FixedSizingConfig)
strategy.configure_predefined_risk_engine(conf: PassthroughRiskConfig)
```

### 5.4 `backtest()` Method — All Parameters with Defaults

```python
strategy.backtest(
    strategy_id: str = "123456",
    initial_capital: float = 10000.0,
    account_currency: AccountCurrencies = AccountCurrencies.USD,
    account_leverage: int = 30,
    start_date: datetime = datetime(1970, 1, 1),
    end_date: datetime = datetime.now(),   # WARNING: evaluated at module load time
    backtest_name: str = "Backtests",
    symbols_to_trade: list[str] = ['EURUSD'],  # WARNING: mutable default
    csv_dir: str = None,
    run_scheduled_taks: bool = False,      # NOTE: misspelling, parameter is unused
    export_backtest_csv: bool = False,
    export_backtest_parquet: bool = True,
    backtest_results_dir: str = None,
) -> BacktestResults
```

Behavior:
1. Creates a fresh `Queue` (allows sequential/nested backtests).
2. Sets `trading_context = TypeContext.BACKTEST`.
3. Sorts `__strategy_timeframes` ascending; uses the first as `base_timeframe`.
4. Instantiates `DataProvider(CSVBacktestDataConfig)`.
5. Instantiates `ExecutionEngine(MT5SimulatedExecutionConfig)` with initial capital, currency, leverage, and `int(strategy_id)` as magic number.
6. Instantiates `Portfolio`, `Modules`.
7. Creates signal/sizing/risk engine services.
8. Creates `PortfolioHandler` and `TradingDirector`.
9. Forwards scheduled events to `TradingDirector.add_schedule()`.
10. Calls `TradingDirector.run()`.
11. Logs backtest duration at WARNING level.
12. Returns `BacktestResults`.

### 5.5 `run_live()` Method — All Parameters

```python
strategy.run_live(
    mt5_configuration: Mt5PlatformConfig,  # required
    strategy_id: str = "default",
    initial_capital: float = 10000.0,
    symbols_to_trade: list[str] = ['EURUSD'],
    heartbeat: float = 0.1,
) -> None
```

Same pipeline as `backtest()` but uses `MT5LiveDataConfig`, `MT5LiveExecutionConfig`,
`MT5LiveSessionConfig`, and `TypeContext.LIVE`. Runs indefinitely.

### 5.6 Schedule Management Methods

```python
strategy.activate_schedules() -> None    # sets __run_schedules = True
strategy.deactivate_schedules() -> None  # sets __run_schedules = False
strategy.enable_hooks() -> None
strategy.disable_hooks() -> None
```

### 5.7 Modules Object Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `TRADING_CONTEXT` | `TypeContext` | `TypeContext.BACKTEST` | `BACKTEST` or `LIVE` |
| `DATA_PROVIDER` | `IDataProvider` | required | Bar data access |
| `EXECUTION_ENGINE` | `IExecutionEngine` | required | Order placement and account queries |
| `PORTFOLIO` | `IPortfolio` | required | Position and balance queries |

Pydantic model with `arbitrary_types_allowed = True` (v1-style inner `Config` class).
A single shared `Modules` instance is passed to all callbacks within a session.

### 5.8 VerboseLevel Values

`VerboseLevel` is a plain `int` subclass (not an Enum):

| Attribute | Value |
|---|---|
| `CRITICAL` | 50 |
| `FATAL` | 50 |
| `ERROR` | 40 |
| `WARNING` | 30 |
| `WARN` | 30 |
| `INFO` | 20 |
| `DEBUG` | 10 |
| `NOTSET` | 0 |

---

## 6. Functional Requirements — Signal Generation

### 6.1 ISignalEngine Interface

Abstract base class. Concrete implementations must implement:

```python
def generate_signal(self, bar_event: BarEvent, modules: Modules) -> SignalEvent | list[SignalEvent] | None
```

### 6.2 SignalEngineService Flow

1. Receives `BarEvent` from `TradingDirector._handle_bar_event`.
2. Calls `self.signal_engine.generate_signal(bar_event, modules)` (or the closure set by
   `set_signal_engine`).
3. If result is a `SignalEvent`, puts it on the queue.
4. If result is a `list[SignalEvent]`, puts each item on the queue individually.
5. `None` result is silently discarded.

`_get_signal_engine` factory:
- `MACrossoverConfig` → `SignalMACrossover`
- Anything else → `SignalPassthrough` (no-op; never emits signals)

`set_signal_engine(new_signal_engine)` — replaces `generate_signal` with a closure calling
`new_signal_engine(bar_event, modules)`. Used by `@strategy.custom_signal_engine` decorator wiring.

### 6.3 Built-in: MACrossover Signal Engine

Config: `MACrossoverConfig`

| Field | Type | Default |
|---|---|---|
| `strategy_id` | `str` | required |
| `signal_timeframe` | `str` | required |
| `ma_type` | `MAType` | `MAType.SIMPLE` |
| `fast_period` | `int | HyperParameter` | required |
| `slow_period` | `int | HyperParameter` | required |

`MAType` enum: `SIMPLE = "SIMPLE"`, `EXPONENTIAL = "EXPONENTIAL"`.

Signal generation logic (`SignalMACrossover.generate_signal`):
1. Filters out bar events not matching `signal_timeframe`.
2. Fetches `slow_period + 1` bars via `modules.DATA_PROVIDER.get_latest_bars(symbol, timeframe, N=slow_period+1)`.
3. Queries `modules.PORTFOLIO.get_number_of_strategy_open_positions_by_symbol(symbol)`.
4. Returns `None` if fewer than 2 rows available.
5. Computes fast MA (mean of last `fast_period` closes) and slow MA (mean of all fetched closes).
   - `MAType.SIMPLE`: uses `pandas.mean()`
   - `MAType.EXPONENTIAL`: uses `pandas.ewm()`
6. BUY condition: `fast_ma > slow_ma` AND no long position open → closes any short, emits BUY signal.
7. SELL condition: `fast_ma < slow_ma` AND no short position open → closes any long, emits SELL signal.
8. Signal has `forecast=10` (hardcoded), `order_type="MARKET"`, `order_price=latest_tick`.

**Note**: The engine calls `modules.EXECUTION_ENGINE.close_strategy_*_positions_by_symbol()` as a
side-effect inside signal generation, coupling signal generation to execution.

### 6.4 Custom Signal Engine Function Signature

```python
def my_signal_engine(bar_event: BarEvent, modules: Modules) -> SignalEvent | list[SignalEvent] | None:
    ...
```

Registered via `@strategy.custom_signal_engine(strategy_id='default', strategy_timeframes=[...])`.

### 6.5 SignalPassthrough

Default engine used when no signal config or custom function is provided. Never emits signals
(`generate_signal` returns `None`). Used as the internal fallback.

---

## 7. Functional Requirements — Position Sizing

### 7.1 ISizingEngine Interface

```python
def get_suggested_order(self, signal_event: SignalEvent, modules: Modules) -> SuggestedOrder
```

### 7.2 MinSizingConfig → MT5MinSizing

Config: `MinSizingConfig` (no fields).

Behavior: queries `mt5.symbol_info(symbol).volume_min` at order time and uses that as the volume.
Falls back to default minimum if symbol info is unavailable.

### 7.3 FixedSizingConfig → MT5FixedSizing

Config: `FixedSizingConfig(volume: Decimal)`.

Behavior: returns the fixed volume for all orders regardless of conditions.

### 7.4 RiskPctSizingConfig → MT5RiskPctSizing

Config: `RiskPctSizingConfig(risk_pct: float)`.

- `risk_pct`: percentage of account equity to risk per trade (e.g., `1` = 1%).

Sizing formula (from `MT5RiskPctSizing.get_suggested_order`):
1. Determine entry price:
   - BUY MARKET → `get_latest_tick(symbol)["ask"]`
   - SELL MARKET → `get_latest_tick(symbol)["bid"]`
   - LIMIT/STOP → `signal_event.order_price`
2. `tick_value_profit_ccy = mt5.symbol_info(symbol).contract_size * mt5.symbol_info(symbol).trade_tick_size`
3. `tick_value_account_ccy = Utils.convert_currency_amount_to_another_currency(tick_value_profit_ccy, profit_ccy, account_ccy, data_provider)`
4. `price_distance = int(abs(entry_price - signal_event.sl) / tick_size)` (integer tick count; note: truncation may underestimate SL distance)
5. `monetary_risk = mt5.account_info().equity * risk_pct / 100`
6. `volume = monetary_risk / (price_distance * tick_value_account_ccy)`
7. `volume = round(volume / volume_step) * volume_step` (rounded to symbol's volume_step)

Requirements:
- `risk_pct > 0` (raises `Exception` if ≤ 0).
- `signal_event.sl != 0` (raises `Exception` if zero — stop-loss is mandatory).

Currency conversion uses `Utils.convert_currency_amount_to_another_currency` with a static list of
30 major FX pairs. Profit currencies outside this list will cause an `IndexError`.

Volume bounds check (min/max) is **not implemented** — calculated volume is not clamped.

### 7.5 SizingEngineService Factory Dispatch

| Config type | Engine instantiated |
|---|---|
| `MinSizingConfig` | `MT5MinSizing(trading_context)` |
| `FixedSizingConfig` | `MT5FixedSizing(configs)` |
| `RiskPctSizingConfig` | `MT5RiskPctSizing(configs, trading_context)` |
| Anything else / `BaseSizingConfig` | `MT5MinSizing(trading_context)` (fallback) |

### 7.6 Custom Sizing Function Signature

```python
def my_sizing_engine(signal_event: SignalEvent, modules: Modules) -> SuggestedOrder | list[SuggestedOrder] | None:
    ...
```

Registered via `@strategy.custom_sizing_engine(strategy_id='default')`. Injected via
`SizingEngineService.set_suggested_order_function(fn)`.

---

## 8. Functional Requirements — Risk Management

### 8.1 IRiskEngine Interface

```python
def assess_order(self, suggested_order: SuggestedOrder, modules: Modules) -> float
```

Returns the approved volume. Volume ≤ 0 means order is rejected.

### 8.2 PassthroughRiskConfig → PassthroughRiskEngine

Config: `PassthroughRiskConfig` (no fields).

Behavior: returns the volume from the `SuggestedOrder` unchanged. All orders pass through without
filtering.

### 8.3 RiskEngineService

`RiskEngineService.assess_order(suggested_order)` flow:
1. Delegates to `self.risk_engine.assess_order(suggested_order, modules)` → `new_volume: float`.
2. If `new_volume > 0.0`: calls `_create_and_put_order_event(suggested_order, new_volume)`.
3. `_create_and_put_order_event` constructs `OrderEvent` copying fields from the underlying
   `SignalEvent`: symbol, time_generated, strategy_id, signal_type, order_type, order_price, sl, tp,
   rollover, buffer_data. Uses the approved `new_volume` as volume.
4. Puts `OrderEvent` on the shared event queue.

`set_custom_asses_order(fn)` — replaces `assess_order` with a closure calling `fn(suggested_order, modules)`.

Note: method name contains a typo — `asses` instead of `assess`.

### 8.4 Custom Risk Engine Function Signature

```python
def my_risk_engine(suggested_order: SuggestedOrder, modules: Modules) -> float:
    ...
    return volume  # > 0 to approve, <= 0 to reject
```

Registered via `@strategy.custom_risk_engine(strategy_id='default')`.

---

## 9. Functional Requirements — Order Execution

### 9.1 Supported Order Types

- **MARKET**: Execute at current market price via `mt5.order_send(TRADE_ACTION_DEAL)`.
- **LIMIT**: Execute at specified price or better via `mt5.order_send(TRADE_ACTION_PENDING)`.
- **STOP**: Execute when price reaches stop level via `mt5.order_send(TRADE_ACTION_PENDING)`.
- **CONT**: Continuation/rollover order — routes to `execute_desired_continuous_trade()` in live
  connector. Used for futures contract rollovers.

### 9.2 Order Filling Mode

Live execution uses `ORDER_FILLING_FOK` (Fill or Kill) exclusively.

### 9.3 SL/TP Handling

- **Simulator**: `_update_values_and_check_executions_and_fills(bar_event)` checks SL/TP on every
  bar. Closes position and emits `FillEvent` when hit.
- **Live**: Relies entirely on MT5 server-side processing. Client-side `_update_values_and_check_executions_and_fills` is a no-op (`pass`).

### 9.4 Partial Close Support

Live connector: `close_position(position_ticket, partial_volume: Decimal = 0.0)`.
- If `partial_volume > 0`, closes only that volume.
- If `partial_volume == 0`, closes the full position.

### 9.5 Position SL/TP Modification

```python
execution_engine.update_position_sl_tp(position_ticket: int, new_sl: Decimal, new_tp: Decimal)
```

Live: sends `TRADE_ACTION_SLTP`. Simulator: directly mutates the `TradePosition` in `open_positions`.

### 9.6 Pending Order Cancellation

```python
execution_engine.cancel_pending_order(order_ticket: int)
execution_engine.cancel_all_strategy_pending_orders()
execution_engine.cancel_all_strategy_pending_orders_by_type_and_symbol(order_type: str, symbol: str)
```

### 9.7 Magic Number / Strategy ID Isolation

- `strategy_id` (string of digits) is cast to `int(strategy_id)` for use as the MT5 magic number.
- All position/order queries filter by `magic_number` to isolate this strategy from others running
  concurrently in the same terminal.
- Auto-incrementing magic number assignment via `__create_mg_for_strategy_id` exists but is
  currently unused (call site is commented out).

### 9.8 Simulated Execution Details

(`Mt5SimulatorExecutionEngineConnector`)

- **Account state**: tracked in memory: `balance`, `equity`, `used_margin`, `free_margin`.
- **Ticket counters**: positions start at 200000000; deals start at 300000000.
- **Market order fill**: at current bar price (from `DATA_PROVIDER.get_latest_tick()`).
- **Pending order trigger**: checked against each new bar's OHLC. Triggered when price condition met.
- **P&L calculation**: includes spread cost, commission (from `SymbolInfo`), swap.
- **Currency conversion**: for instruments where profit currency ≠ account currency.
- **Margin tracking**: calculated but enforcement is a TODO (margin check not implemented).
- **Symbol categories**: hardcoded tuples — ~30 FX pairs, 4 commodities, 10 indices.
- **`_update_shared_data_account_info()`**: syncs in-memory state to `SharedData` after each trade.

### 9.9 Live Execution — Retry Logic

Market orders (`_send_market_order`):
- Sends order via `mt5.order_send(TRADE_ACTION_DEAL)`.
- Polls `mt5.history_deals_get(position=result.order)` up to **100 times** with 50ms sleep between
  attempts (maximum total wait: **5 seconds**).
- For each deal found: calls `_generate_and_put_fill_event()`.

Pending order fill check (`_check_if_pending_orders_filled`):
- Polls up to **20 times** with 50ms sleep (maximum: **1 second**).
- Note: this method exists but is never called from `_update_values_and_check_executions_and_fills`
  (which is a no-op).

**Note**: Deal confirmation retry is blocking — up to 5 seconds of `time.sleep()` blocks the event
loop with no async support.

**Known MT5 bug workaround**: Live accounts may return 0 in `result.deal` field. Code uses
`position=result.order` as the workaround.

### 9.10 Account Modes (Live)

`LiveMT5Broker._live_account_warning()`:
- **DEMO** (`ACCOUNT_TRADE_MODE_DEMO`): 3-second countdown with info-level logging.
- **REAL** (`ACCOUNT_TRADE_MODE_REAL`): 10-second countdown with warning-level logging.
- **CONTEST**: Single info log, no countdown.

---

## 10. Functional Requirements — Portfolio Tracking

### 10.1 State Tracked

| Attribute | Type | Description |
|---|---|---|
| `_initial_balance` | `Decimal` | Starting account balance |
| `_balance` | `Decimal` | Current account balance |
| `_equity` | `Decimal` | Current account equity (balance + floating P&L) |
| `_realised_pnl` | `Decimal` | `balance - initial_balance` |
| `_unrealised_pnl` | `Decimal` | `equity - balance` |
| `_strategy_positions` | `tuple[OpenPosition]` | Currently open positions |
| `_strategy_pending_orders` | `tuple[PendingOrder]` | Currently pending orders |
| `historical_balance` | `dict[datetime, Decimal]` | Balance at each base-TF bar (backtest only) |
| `historical_equity` | `dict[datetime, Decimal]` | Equity at each base-TF bar (backtest only) |

### 10.2 Position Entities

**OpenPosition** fields:
| Field | Type | Required |
|---|---|---|
| `time_entry` | `datetime` | Yes |
| `price_entry` | `Decimal` | Yes |
| `type` | `str` | Yes (`"BUY"` or `"SELL"`) |
| `symbol` | `str` | Yes |
| `ticket` | `int` | Yes |
| `volume` | `Decimal` | Yes |
| `unrealized_profit` | `Decimal` | Yes |
| `strategy_id` | `str` | Yes |
| `sl` | `Optional[Decimal]` | No |
| `tp` | `Optional[Decimal]` | No |
| `swap` | `Optional[Decimal]` | No |
| `comment` | `Optional[str]` | No |

**ClosedPosition** fields:
| Field | Type | Required |
|---|---|---|
| `time_entry` | `datetime` | Yes |
| `price_entry` | `Decimal` | Yes |
| `time_exit` | `datetime` | Yes |
| `price_exit` | `Decimal` | Yes |
| `strategy_id` | `str` | Yes |
| `ticket` | `int` | Yes |
| `symbol` | `str` | Yes |
| `direction` | `str` | Yes |
| `volume` | `Decimal` | Yes |
| `commission` | `Decimal` | Yes |
| `pnl` | `Decimal` | Yes |
| `sl` | `Optional[Decimal]` | No |
| `tp` | `Optional[Decimal]` | No |
| `swap` | `Optional[Decimal]` | No |
| `comment` | `Optional[str]` | No |

**PendingOrder** fields:
| Field | Type | Required |
|---|---|---|
| `price` | `Decimal` | Yes |
| `type` | `str` | Yes (`"BUY_LIMIT"`, `"SELL_LIMIT"`, `"BUY_STOP"`, `"SELL_STOP"`) |
| `symbol` | `str` | Yes |
| `ticket` | `int` | Yes |
| `volume` | `Decimal` | Yes |
| `strategy_id` | `str` | Yes |
| `sl` | `Optional[Decimal]` | No |
| `tp` | `Optional[Decimal]` | No |
| `comment` | `Optional[str]` | No |

### 10.3 Historical Balance/Equity Series

- Recorded only in **backtest mode** on **base timeframe** bar events.
- Recording is limited to the **first-seen symbol** (to avoid duplicate entries when multiple symbols
  emit base-TF bars at the same time).
- Stored as `dict[datetime, Decimal]`.

### 10.4 Portfolio Query API

```python
portfolio.get_account_balance() -> Decimal
portfolio.get_account_equity() -> Decimal
portfolio.get_account_unrealised_pnl() -> Decimal
portfolio.get_account_realised_pnl() -> Decimal
portfolio.get_positions(symbol: str = '', ticket: int = None) -> tuple[OpenPosition]
portfolio.get_pending_orders(symbol: str = '', ticket: int = None) -> tuple[PendingOrder]
portfolio.get_number_of_strategy_open_positions_by_symbol(symbol: str) -> dict[str, int]
    # Returns: {"LONG": int, "SHORT": int, "TOTAL": int}
portfolio.get_number_of_strategy_pending_orders_by_symbol(symbol: str) -> dict[str, int]
    # Returns: {"BUY_LIMIT": int, "SELL_LIMIT": int, "BUY_STOP": int, "SELL_STOP": int, "TOTAL": int}
```

### 10.5 Portfolio Export Formats

- `_export_historical_pnl_dataframe()` → `pd.DataFrame` with columns `BALANCE`, `EQUITY`, datetime index.
- `_export_historical_pnl_to_parquet(file_path)` → Parquet with zstd compression level 10, rounded to 2 decimal places.
- `_export_historical_pnl_json()` → JSON string with balance/equity scaled to integers (4 decimal precision).
- `_export_csv_historical_pnl(file_path)` → CSV via pandas.

---

## 11. Functional Requirements — Scheduling

### 11.1 Registration

```python
@strategy.run_every(interval: StrategyTimeframes)
def my_schedule(scheduled_event: ScheduledEvent, modules: Modules) -> None:
    ...
```

Or imperatively:
```python
trading_director.add_schedule(timeframe: StrategyTimeframes, fn: Callable)
```

### 11.2 Supported Intervals

All 22 values of `StrategyTimeframes` enum are supported as scheduling intervals.

### 11.3 Trigger Logic

`ScheduleService.__get_timeframes_to_trigger(event: BarEvent)`:
1. If bar has no valid datetime, return empty list.
2. For each watched timeframe:
   - If `current_timestamp is None` (first bar): initialize both timestamps and skip (first bar
     never triggers callbacks).
   - Otherwise: update `current_timestamp` to bar's datetime.
   - If `current_timestamp - last_timestamp >= timeframe.to_timedelta()`: add to trigger list.
3. Returns list of timeframes to trigger.

### 11.4 Schedule Model

`Schedule(BaseModel)`:
| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | required | `repr(callback)` at registration time |
| `is_active` | `bool` | `True` | Whether the schedule fires |
| `fn` | `Callable` | required | Callback function |
| `execute_every` | `StrategyTimeframes` | required | Trigger interval |

Equality comparison is based on `name` only.

### 11.5 Activate/Deactivate

```python
schedule_service.activate_schedules()    # activates all schedules
schedule_service.deactivate_schedules()  # deactivates all schedules
```

Individual schedule activation:
```python
schedules.activate_schedule(schedule)
schedules.deactivate_schedule(schedule)
```

Global disable: `TradingDirector` calls `deactivate_schedules()` at loop start when
`run_schedules=False`.

---

## 12. Functional Requirements — Hooks

### 12.1 Hooks Enum

```python
class Hooks(str, Enum):
    ON_START       = 'ON_START'       # fires before main event loop
    ON_SIGNAL_EVENT = 'ON_SIGNAL_EVENT' # fires when SignalEvent dequeued (before PortfolioHandler)
    ON_ORDER_EVENT  = 'ON_ORDER_EVENT'  # fires after OrderEvent processed by ExecutionEngine
    ON_END         = 'ON_END'          # fires after main event loop exits
```

### 12.2 Hook Callback Signature

```python
def my_hook(modules: Modules) -> None:
    ...
```

Registered via `@strategy.hook(hook=Hooks.ON_START)`.

Note: callbacks do **not** receive the triggering event — only `Modules`.

### 12.3 Multiple Callbacks per Hook

Multiple callbacks can be registered per hook. They execute in **registration order** (no priority
mechanism).

Stored as `dict[Hooks, list[Callable[[Modules], None]]]`.

### 12.4 Enable/Disable Hooks Globally

```python
strategy.enable_hooks()   # -> HookService.__hooks_enabled = True
strategy.disable_hooks()  # -> HookService.__hooks_enabled = False
```

When disabled, `call_callbacks()` is a no-op.

---

## 13. Functional Requirements — Indicators

All indicators:
- Are classes with a public **static** `compute()` method.
- Accept **numpy arrays** as input.
- Return **numpy arrays** or **tuples of numpy arrays**.
- Use `@njit` (Numba JIT) on private computation kernels for performance.
- Initialize warm-up period positions with `NaN`.
- Raise `ValueError` for insufficient input data.
- Input pattern: `.select('close').to_numpy().flatten()` from Polars DataFrame.

### KAMA (Kaufman Adaptive Moving Average)

```python
KAMA.compute(close: np.ndarray, n_period: int = 10, period_fast: int = 2, period_slow: int = 30) -> np.ndarray
```

- Adjusts smoothing based on market efficiency ratio.
- Raises `ValueError` if `len(close) < n_period`.
- Indices before `n_period - 1` are `NaN`.

### SMA (Simple Moving Average)

```python
SMA.compute(close: np.ndarray, period: int) -> np.ndarray
```

- Rolling sum approach.
- Raises `ValueError` if `len(close) < period`.
- Indices before `period - 1` are `NaN`.

### EMA (Exponential Moving Average)

```python
EMA.compute(close: np.ndarray, period: int) -> np.ndarray
```

- Initialized with SMA of first `period` values.
- Multiplier: `2 / (period + 1)`.
- Raises `ValueError` if `len(close) < period`.
- Indices before `period - 1` are `NaN`.
- Note: no `smoothing` parameter in actual implementation (unlike some documentation notes).

### BollingerBands

```python
BollingerBands.compute(close: np.ndarray, period: int = 20, std_dev: float = 2.0) -> tuple
```

- Returns `(upper_band, middle_band, lower_band)` — three `np.ndarray`.
- Middle = SMA. Bands = SMA ± std_dev × standard deviation.
- Uses **population** standard deviation (`variance / period`, not `/ (period - 1)`).
- Raises `ValueError` if `len(close) < period`.

### ATR (Average True Range)

```python
ATR.compute(high: np.ndarray, low: np.ndarray, close: np.ndarray, period: int, method: str = 'sma') -> np.ndarray
```

- Computes True Range first, then smooths via SMA or EMA.
- `method`: `'sma'` or `'ema'`. Raises `ValueError` for invalid method.
- Raises `ValueError` if arrays differ in length.
- Indices before `period - 1` are `NaN`.

### RSI (Relative Strength Index)

```python
RSI.compute(close: np.ndarray, period: int = 14) -> np.ndarray
```

- Uses Wilder's smoothed average method.
- Returns values in 0–100 range.
- Raises `ValueError` if `len(close) < period + 1`.
- Indices before `period` are `NaN`.

### ADX (Average Directional Index)

```python
ADX.compute(high: np.ndarray, low: np.ndarray, close: np.ndarray, period: int = 14) -> tuple
```

- Returns `(adx, plus_di, minus_di)` — three `np.ndarray`.
- Raises `ValueError` if arrays differ in length or `len < period * 2`.

### Momentum

```python
Momentum.compute(close: np.ndarray, period: int = 10) -> np.ndarray
```

- Simple price difference: `close[i] - close[i - period]`.
- Raises `ValueError` if `len(close) < period + 1`.
- Indices before `period` are `NaN`.

### DonchianChannels

```python
DonchianChannels.compute(high: np.ndarray, low: np.ndarray, period: int = 20) -> tuple
```

- Returns `(upper, middle, lower)` — three `np.ndarray`.
- Upper = highest high, lower = lowest low, middle = midpoint.
- Raises `ValueError` if arrays differ in length or `len < period`.

### MACD

```python
MACD.compute(close: np.ndarray, fast_period: int = 12, slow_period: int = 26, signal_period: int = 9) -> tuple
```

- Returns `(macd_line, signal_line, histogram)` — three `np.ndarray`.
- MACD line = fast EMA - slow EMA. Signal = EMA of MACD. Histogram = MACD - signal.
- Raises `ValueError` if `len(close) < slow_period + signal_period`.

### KeltnerChannel

```python
KeltnerChannel.compute(high: np.ndarray, low: np.ndarray, close: np.ndarray, period: int = 20, atr_period: int = 10, multiplier: float = 2.0) -> tuple
```

- Returns `(upper, middle, lower)` — three `np.ndarray`.
- Middle = EMA. Bands = EMA ± multiplier × ATR.
- Raises `ValueError` if arrays differ in length or `len < max(period, atr_period)`.

### Public API Export Status

Exported from `__init__.py`: `KAMA`, `SMA`, `EMA`, `ATR`.
**Not exported**: `RSI`, `ADX`, `Momentum`, `BollingerBands`, `DonchianChannels`, `MACD`,
`KeltnerChannel` (accessible via direct module import).

---

## 14. Functional Requirements — Backtest Results

### 14.1 BacktestResults Object

```python
BacktestResults(backtest_pnl: pd.DataFrame, trades: pd.DataFrame)
```

Properties:
| Property | Type | Description |
|---|---|---|
| `pnl` | `pd.DataFrame` | PnL DataFrame cast to float |
| `returns` | `pd.Series` | `pct_change()` of `EQUITY` column |
| `trades` | `pd.DataFrame` | Trades DataFrame as-provided |
| `backtest_pnl` | `pd.DataFrame` | Raw PnL DataFrame (uncast) |

### 14.2 `trades` DataFrame Columns

Produced by `TradeArchiver.export_historical_trades_dataframe()`:

| Column | Type in Parquet | Source |
|---|---|---|
| `TYPE` | `pl.Utf8` | `FillEvent.type.value` |
| `DEAL` | `pl.Utf8` | `FillEvent.deal.value` (`"IN"` or `"OUT"`) |
| `SYMBOL` | `pl.Utf8` | `FillEvent.symbol` |
| `TIME_GENERATED` | `pl.Datetime` | `FillEvent.time_generated` |
| `POSITION_ID` | `pl.Int64` | `FillEvent.position_id` |
| `STRATEGY_ID` | `pl.Int64` | `FillEvent.strategy_id` |
| `EXCHANGE` | `pl.Utf8` | `FillEvent.exchange` |
| `VOLUME` | `pl.Float64` | `FillEvent.volume` |
| `PRICE` | `pl.Float64` | `FillEvent.price` |
| `SIGNAL_TYPE` | `pl.Utf8` | `FillEvent.signal_type.value` |
| `COMMISSION` | `pl.Float64` | `FillEvent.commission` |
| `SWAP` | `pl.Float64` | `FillEvent.swap` |
| `FEE` | `pl.Float64` | `FillEvent.fee` |
| `GROSS_PROFIT` | `pl.Float64` | `FillEvent.gross_profit` |
| `CCY` | `pl.Utf8` | `FillEvent.ccy` |

### 14.3 `pnl` DataFrame Columns

Produced by `Portfolio._export_historical_pnl_dataframe()`:

| Column | Type | Description |
|---|---|---|
| index (DATETIME) | datetime | Bar timestamp |
| `BALANCE` | float | Account balance at that bar |
| `EQUITY` | float | Account equity at that bar |

### 14.4 `plot()` Method

```python
backtest_results.plot() -> None
```

- Renders a matplotlib line chart of `EQUITY` and `BALANCE` columns.
- Title: `"Backtest"`.
- Tight layout margins.
- Legend shown.
- Calls `plt.show()`.

### 14.5 Export Formats

From `PortfolioHandler.process_backtest_end()`:

**CSV export** (when `export_backtest_csv=True`):
- Trades CSV: `Desktop/PyEventBT/backtest_results_csv/{backtest_name}_trades_{timestamp}.csv`
- PnL CSV: `Desktop/PyEventBT/backtest_results_csv/{backtest_name}_pnl_{timestamp}.csv`

**Parquet export** (when `export_backtest_parquet=True`; default):
- Trades Parquet: `Desktop/PyEventBT/backtest_results_parquet/{backtest_name}_trades_{timestamp}.parquet`
- PnL Parquet: `Desktop/PyEventBT/backtest_results_parquet/{backtest_name}_pnl_{timestamp}.parquet`
- Parquet uses **zstd compression, level 10**.
- Custom directory via `backtest_results_dir` parameter.

**Compressed pickle** (`save_compressed_pickle`):
- Uses `zipfile` + `pickle`. Password parameter exists but has no effect (Python `zipfile` does
  not support writing password-protected archives).

### 14.6 WalkForwardResults

```python
WalkForwardResults(
    backtest_results: BacktestResults,
    retrainting_timestamps: List[pd.Timestamp],  # NOTE: typo in field name
    hyperparameters_track: pd.DataFrame,
)
```

- `WalkforwardType` enum: `ANCHORED = 'ANCHORED'` / `UNANCHORED = 'UNANCHORED'`.
- Serializable to/from CSV:
  - `to_csv(path)` → writes `backtest_results.csv`, `retrainting_timestamps.csv`,
    `hyperparameters_track.csv`.
  - `from_csv(path)` → reconstructs from those CSV files.
- Walk-forward optimization logic is **not yet integrated** — the `WalkforwardType` enum is defined
  but not referenced in `strategy.py`. Optimization imports are commented out.

---

## 15. Functional Requirements — Live Trading

### 15.1 MT5 Platform Initialization

`LiveMT5Broker.initialize_platformV2()` calls:
```python
mt5.initialize(
    path=config.path,
    login=config.login,
    password=config.password,
    server=config.server,
    timeout=config.timeout,
    portable=config.portable
)
```

On success: logs terminal version. On failure: raises and logs exception.

### 15.2 MarketWatch Symbol Management

`_add_symbols_to_marketwatch(symbols)`:
1. For each symbol: check `mt5.symbol_info(symbol) is not None`.
2. If not already visible (`symbol_info.visible == False`): call `mt5.symbol_select(symbol, True)`.
3. Log success or failure per symbol.

### 15.3 Algo Trading Check

`_check_algo_trading_enabled()`: checks `mt5.terminal_info().trade_allowed`. Raises `Exception` if
algorithmic trading is disabled in the terminal.

### 15.4 Heartbeat Loop

Default heartbeat: **0.1 seconds** (configurable via `run_live(heartbeat=0.1)`).
`TradingDirector._run_live_trading()` calls `time.sleep(self.heartbeat)` at each iteration.

### 15.5 Deal Retry Logic

Market orders: up to 100 retry attempts, 50ms sleep each = max 5 seconds.
Pending fill check: up to 20 retry attempts, 50ms sleep each = max 1 second.

### 15.6 Futures Continuous Contract Support

- `CONT` order type exists for rollover semantics.
- `SignalEvent.rollover` tuple: `(needs_rollover: bool, original_contract: str, new_contract: str)`.
- `OrderEvent.rollover` carries the same tuple through to execution.
- Live connector routes `CONT` orders to `execute_desired_continuous_trade()`.
- `futures_tuple` attribute in live data connector is initialized but largely unused (futures
  handling in `update_bars()` is commented out).

---

## 16. Functional Requirements — Data Download (Quantdle)

### 16.1 QuantdleDataUpdater Constructor

```python
QuantdleDataUpdater(api_key: str, api_key_id: str)
```

- Imports `quantdle` package (raises `ImportError` with install instructions if missing).
- Creates authenticated `quantdle.Client`.

### 16.2 `update_data()` Method

```python
update_data(
    csv_dir: str,
    symbols: list[str],
    start_date: datetime,
    end_date: datetime,
    timeframe: str = "1min",
    spread_column: str = "spreadopen"
) -> None
```

For each symbol:
- If CSV file exists: calls `_update_existing_csv()` to fill date range gaps.
- If no CSV exists: calls `_create_new_csv()` to download the full range.

### 16.3 Incremental Update Logic

`_update_existing_csv()`:
1. Reads existing CSV (headerless, 9 columns, matching `CSVDataProvider` format).
2. Parses the existing datetime range (earliest and latest bars).
3. Downloads data for gaps:
   - **Before existing range**: from `start_date` to `csv_start` (if `start_date < csv_start`).
   - **After existing range**: from `csv_end` to `end_date` (if `end_date > csv_end`).
4. Concatenates all DataFrames, removes duplicates by datetime, sorts by datetime.
5. Drops `datetime` helper column.
6. Overwrites the CSV (no backup; data loss risk on failure).

**Note**: Intra-range gaps (missing days within existing CSV) are **not detected or filled**.

### 16.4 Quantdle Timeframe Mapping

| PyEventBT | Quantdle |
|---|---|
| `"1min"` | `"M1"` |
| `"5min"` | `"M5"` |
| `"15min"` | `"M15"` |
| `"30min"` | `"M30"` |
| `"1h"` / `"1H"` | `"H1"` |
| `"4h"` / `"4H"` | `"H4"` |
| `"1d"` / `"1D"` | `"D1"` |
| `"1w"` / `"1W"` | `"W1"` |

Unrecognized timeframes are passed through unchanged (silent fallback).

### 16.5 Output Format

Matches `CSVDataProvider` format:
- Headerless CSV.
- Columns: `[date, time, open, high, low, close, tickvol, volume, spread]`.
- `date` format: `YYYY.MM.DD`.
- `time` format: `HH:MM:SS`.
- Price columns: `Float64`. Volume columns: `Int64`.

### 16.6 Download Logic

`_download_from_quantdle()`:
1. Converts timeframe string to Quantdle format.
2. Calls `quantdle.Client.download_data(symbol=[symbol], timeframe=..., start_date=..., end_date=..., output_format="polars")`.
3. Normalizes datetime column name.
4. Casts price columns to `Float64`, volume columns to `Int64`.
5. Maps selected spread column (e.g., `"spreadopen"`) to `"spread"`, drops other spread columns.
6. Creates `date` and `time` columns in MT5 format.
7. Returns `pl.DataFrame` or `None` on error.

---

## 17. MT5 Simulator Details

### 17.1 SharedData Class-Level State

Stored as class-level attributes (pseudo-singleton, reset on every `SharedData()` construction):

| Attribute | Type | Default |
|---|---|---|
| `last_error_code` | `tuple` | `(-1, 'generic fail')` |
| `credentials` | `InitCredentials` | `None` |
| `terminal_info` | `TerminalInfo` | Loaded from `default_terminal_info.yaml` |
| `account_info` | `AccountInfo` | Loaded from `default_account_info.yaml` |
| `symbol_info` | `dict[str, SymbolInfo]` | Loaded from `default_symbols_info.yaml` |

**Default account** (from YAML, `REQ-SHARED-004`):
- Balance: **10000**
- Leverage: **30**
- Currency: **USD**

Runtime attributes (set by execution engine, not declared in class body):
- `open_positions`, `pending_orders`, `closed_positions`, `history_deals`, `next_order_ticket`

### 17.2 Default Symbol Universe

- **~30 major FX pairs** with full `SymbolInfo` (including digits, spread, contract_size, tick values,
  volume_min, volume_step, volume_max, currency_base, currency_profit, currency_margin).
- **4 commodity symbols**.
- **10 index symbols**.
- Hardcoded in both `SharedData` YAML and execution engine connector tuples.

MT5 Timeframe Constants supported:
M1, M2, M3, M4, M5, M6, M10, M12, M15, M20, M30, H1, H2, H3, H4, H6, H8, H12, D1, W1, MN1.

### 17.3 Order/Position/Deal ID Management

- Position tickets: auto-incrementing integer, starting at **200000000**.
- Deal tickets: auto-incrementing integer, starting at **300000000**.
- `next_order_ticket`: stored in `SharedData`, used by all connector methods.

### 17.4 Symbol Connector Behavior

`SymbolConnector.symbols_get(group: str = "*")`:
- Comma-separated conditions.
- `*` wildcard converted to `.*` regex.
- `!` prefix = exclusion condition.
- Multiple inclusion conditions are applied sequentially (AND logic — may differ from real MT5
  which uses OR logic for multiple inclusions).

`symbol_info_tick()` is **not implemented** (returns `None`). Tick data for backtesting is handled
by the `DataProvider`, not the symbol connector.

### 17.5 `Mt5SimulatorWrapper`

Static class that aggregates all connector classes (`PlatformConnector`, `AccountInfoConnector`,
`TerminalInfoConnector`, `SymbolConnector`) into a single `mt5`-compatible namespace. The
`SharedData()` is instantiated at class-body evaluation time (import time), loading all YAML files.

---

## 18. Configuration Objects

### 18.1 `Mt5PlatformConfig`

All fields required, no defaults:

| Field | Type | Description |
|---|---|---|
| `path` | `str` | Path to MT5 terminal executable |
| `login` | `int` | MT5 login ID |
| `password` | `str` | MT5 password (plain string, no `SecretStr`) |
| `server` | `str` | MT5 server name |
| `timeout` | `int` | Connection timeout in milliseconds |
| `portable` | `bool` | Whether portable MT5 installation |

Inherits `BaseConfig` (Pydantic model with YAML load/save capability).

### 18.2 `CSVBacktestDataConfig`

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `csv_path` | `str` | Yes | — | Directory containing per-symbol CSV files |
| `account_currency` | `str` | Yes | — | Must be `"USD"`, `"EUR"`, or `"GBP"` |
| `tradeable_symbol_list` | `list` | Yes | — | Symbols to trade |
| `base_timeframe` | `str` | Yes | — | Lowest-resolution TF; must be first in `timeframes_list` |
| `timeframes_list` | `list` | Yes | — | Ordered list starting with `base_timeframe` |
| `backtest_start_timestamp` | `datetime | None` | No | `None` | Start date; `None` = use all data |
| `backtest_end_timestamp` | `datetime` | No | `datetime.now()` at import time | End date |

**Known issue**: `backtest_end_timestamp` default is evaluated at module import time.

### 18.3 `MT5LiveDataConfig`

| Field | Type | Required | Description |
|---|---|---|---|
| `tradeable_symbol_list` | `list` | Yes | Symbols to subscribe to |
| `timeframes_list` | `list` | Yes | Timeframe strings to poll |

Note: No `platform_config` field in actual source (mentioned in some documentation but absent).

### 18.4 `MT5SimulatedExecutionConfig`

| Field | Type | Required | Description |
|---|---|---|---|
| `initial_balance` | `Decimal` | Yes | Starting account balance |
| `account_currency` | `str` | Yes | Account denomination currency |
| `account_leverage` | `int` | Yes | Leverage ratio |
| `magic_number` | `int` | Yes | Strategy identifier |

No defaults; no validation for positive values.

### 18.5 `MT5LiveExecutionConfig`

| Field | Type | Required | Description |
|---|---|---|---|
| `magic_number` | `int` | Yes | MT5 magic number for order/position filtering |

### 18.6 `MT5BacktestSessionConfig`

| Field | Type | Description |
|---|---|---|
| `initial_capital` | `float` | Starting account balance |
| `start_date` | `datetime` | Backtest start date |
| `backtest_name` | `str` | Label for the backtest run |

### 18.7 `MT5LiveSessionConfig`

| Field | Type | Description |
|---|---|---|
| `symbol_list` | `list[str]` | Symbols to trade |
| `heartbeat` | `float` | Loop iteration sleep interval (seconds) |
| `platform_config` | `Mt5PlatformConfig` | MT5 connection configuration |

### 18.8 Signal Engine Configurations

**`BaseSignalEngineConfig`**:
- `strategy_id: str`
- `signal_timeframe: str`

**`MACrossoverConfig(BaseSignalEngineConfig)`**:
- `strategy_id: str` (redeclared)
- `signal_timeframe: str` (redeclared)
- `ma_type: MAType` — default `MAType.SIMPLE`
- `fast_period: int | HyperParameter`
- `slow_period: int | HyperParameter`

### 18.9 Sizing Engine Configurations

- `BaseSizingConfig` — no fields; defaults to `MT5MinSizing` when passed directly.
- `MinSizingConfig(BaseSizingConfig)` — no fields.
- `FixedSizingConfig(BaseSizingConfig)` — `volume: Decimal`.
- `RiskPctSizingConfig(BaseSizingConfig)` — `risk_pct: float`.

### 18.10 Risk Engine Configurations

- `BaseRiskConfig` — no fields.
- `PassthroughRiskConfig(BaseRiskConfig)` — no fields.

---

## 19. Known Gaps, Bugs, and Incomplete Features

### 19.1 Architecture / Design

| ID | Category | Description |
|---|---|---|
| GAP-ARCH-01 | Architecture | `ScheduledEvent` is not dispatched through the event queue despite having `EventType.SCHEDULED_EVENT`. It is created inline in `ScheduleService`. `TradingDirector.event_handlers_dict` has no entry for it — queuing one would cause `KeyError`. |
| GAP-ARCH-02 | Architecture | `TradingDirector._handle_order_event` calls `self.EXECUTION_ENGINE._process_order_event(event)`, accessing a private method on another object — encapsulation violation. |
| GAP-ARCH-03 | Architecture | `SignalMACrossover` calls `modules.EXECUTION_ENGINE.close_*_positions_by_symbol()` as a side effect inside signal generation — violates event-driven separation of concerns. |
| GAP-ARCH-04 | Architecture | No graceful shutdown for live trading loop (`while True` with no break condition, signal handler, or exception handling). |
| GAP-ARCH-05 | Architecture | `TradingDirector` constructor has mutable default argument: `hook_service: HookService = HookService()` creates a shared instance across calls that omit this parameter. |
| GAP-ARCH-06 | Architecture | Walk-forward optimization (`WalkforwardType`) is defined but not integrated — optimization imports in `strategy.py` are commented out. |

### 19.2 Strategy API

| ID | Category | Description |
|---|---|---|
| GAP-STRAT-01 | Strategy API | Decorators (`hook`, `custom_signal_engine`, etc.) do not return `fn`. The decorated function is replaced with `None`, making it uncallable outside the decorator context. |
| GAP-STRAT-02 | Strategy API | `run_scheduled_taks` parameter in `backtest()` is misspelled ("tasks") and is never used in the method body. |
| GAP-STRAT-03 | Strategy API | `__create_mg_for_strategy_id` method exists but the call site in `backtest()` is commented out. `strategy_id` is cast directly via `int(strategy_id)`. |
| GAP-STRAT-04 | Strategy API | `end_date` default (`datetime.now()`) evaluated at module load time, not at call time. |
| GAP-STRAT-05 | Strategy API | `strategy_timeframes` and `symbols_to_trade` parameters use mutable list literals as defaults. |
| GAP-STRAT-06 | Strategy API | Only the first `custom_signal_engine`/`custom_sizing_engine`/`custom_risk_engine` registration per `strategy_id` is kept (silent `setdefault` behavior). |

### 19.3 Data Provider — CSV

| ID | Category | Description |
|---|---|---|
| GAP-CSV-01 | CSV Connector | Five deprecated methods remain (`get_latest_tick_old`, `get_latest_bar_old_lookahead_bias`, `get_latest_bars_pandas`, two TF-check variants) without deprecation markers. |
| GAP-CSV-02 | CSV Connector | `_base_minutes` and `_base_idx_map_int` caches are initialized in `__init__` but never populated or read. |
| GAP-CSV-03 | CSV Connector | Auxiliary FX pair list hardcodes 33 pairs. New instruments require code changes. |
| GAP-CSV-04 | CSV Connector | Only `USD`, `EUR`, `GBP` are supported as account currencies. |
| GAP-CSV-05 | CSV Connector | `update_bars` sets `close_positions_end_of_data = True` but never flips `continue_backtest = False`. TradingDirector detects end-of-data through its own logic. |
| GAP-CSV-06 | CSV Connector | Weekly bar datetime adjustment (`- pl.duration(days=7)`) may not align correctly with all trading calendars. |
| GAP-CSV-07 | CSV Connector | ANSI color codes used directly in log output — renders as garbage in non-terminal outputs. |
| GAP-CSV-08 | CSV Connector | `get_latest_bars` slice logic for higher TFs: `slice(df_len - N - 1, N)` may return unexpected results if `df_len == N + 1`. |

### 19.4 Data Provider — MT5 Live

| ID | Category | Description |
|---|---|---|
| GAP-MT5D-01 | Live Data | `update_bars()` calls `get_latest_bar()` twice per new bar — first result is discarded. |
| GAP-MT5D-02 | Live Data | `_map_timeframe` has unreachable `except KeyError` — invalid key fails on the return line outside the try block. |
| GAP-MT5D-03 | Live Data | `last_bar_datetime` attribute initialized but never read or updated. |
| GAP-MT5D-04 | Live Data | Futures contract handling entirely commented out. `futures_tuple` attribute serves no purpose. |
| GAP-MT5D-05 | Live Data | No rate limiting or backoff on MT5 API polling. |
| GAP-MT5D-06 | Live Data | Hardcoded fallback digits (3 for JPY, 5 otherwise) when `symbol_info` unavailable. |
| GAP-MT5D-07 | Live Data | Two deprecated pandas-based methods remain without deprecation markers. |

### 19.5 Execution Engine

| ID | Category | Description |
|---|---|---|
| GAP-EXEC-01 | Simulator | Margin check is a TODO — margin is tracked but never enforced for new orders. |
| GAP-EXEC-02 | Simulator | Symbol lists (FX pairs, commodities, indices) are hardcoded tuples. |
| GAP-EXEC-03 | Live | `_update_values_and_check_executions_and_fills` is a no-op (`pass`) — no proactive pending order fill or SL/TP checking. |
| GAP-EXEC-04 | Live | `_check_if_pending_orders_filled` exists but is never called from the standard bar-processing flow. |
| GAP-EXEC-05 | Live | Deal confirmation retry blocks the event loop (up to 5s of `time.sleep()`). |
| GAP-EXEC-06 | Live | `close_position` returns `False` (bool) on error instead of `OrderSendResult`, breaking return type contract. |
| GAP-EXEC-07 | Live | `_check_succesful_order_execution` — "succesful" typo. |
| GAP-EXEC-08 | Both | `_check_common_trade_values` duplicated between simulator and live connectors. |
| GAP-EXEC-09 | Live | MT5 bug workaround: `result.deal` returns 0 for live accounts; code uses `position=result.order`. |

### 19.6 Signal Engine

| ID | Category | Description |
|---|---|---|
| GAP-SIG-01 | MA Crossover | `trading_context` compared against string `"BACKTEST"` rather than `TypeContext.BACKTEST` enum. |
| GAP-SIG-02 | MA Crossover | `ma_type` compared against raw strings `"SIMPLE"` / `"EXPONENTIAL"` rather than `MAType` enum. |
| GAP-SIG-03 | MA Crossover | `forecast` hardcoded to `10` with no configurability. |
| GAP-SIG-04 | MA Crossover | No `HyperParameter` unwrapping — if `fast_period` or `slow_period` is a `HyperParameter`, arithmetic will fail. |
| GAP-SIG-05 | MA Crossover | Uses `pandas` for MA calculation despite framework preference for Polars/Numba. |
| GAP-SIG-06 | Signal Service | `generate_signal` does not catch exceptions from the engine — engine failure propagates to halt the event loop. |
| GAP-SIG-07 | Signal Service | `set_signal_engine` leaves `self.signal_engine` stale after replacement. |
| GAP-SIG-08 | Signal Service | Debug log message typo: `"SINAL"` instead of `"SIGNAL"`. |

### 19.7 Sizing Engine

| ID | Category | Description |
|---|---|---|
| GAP-SIZ-01 | RiskPct | `_convert_currency_amount_to_another_currency` instance method is dead code (never called). |
| GAP-SIZ-02 | RiskPct | Hard-coded FX pair list of 30 pairs — profit currencies outside this list cause `IndexError`. |
| GAP-SIZ-03 | RiskPct | Integer truncation on price distance may underestimate SL distance and oversize position. |
| GAP-SIZ-04 | RiskPct | No volume bounds check (no clamp to `[volume_min, volume_max]`). |
| GAP-SIZ-05 | RiskPct | Mixed `Decimal`/`float` arithmetic may cause precision issues. |
| GAP-SIZ-06 | Service | `events_queue` is stored on `SizingEngineService` but never used. |

### 19.8 Risk Engine

| ID | Category | Description |
|---|---|---|
| GAP-RISK-01 | Service | Method name typo: `set_custom_asses_order` should be `set_custom_assess_order`. |
| GAP-RISK-02 | Service | `RiskEngineService.assess_order` signature differs from `IRiskEngine.assess_order` (no `modules` param, returns `None` not `float`). |
| GAP-RISK-03 | Service | `_get_risk_management_method` silently falls back to `PassthroughRiskEngine` for unknown configs. |
| GAP-RISK-04 | Configs | No parameterized risk configs exist (no max drawdown, max exposure, etc.). |

### 19.9 Portfolio / Portfolio Handler

| ID | Category | Description |
|---|---|---|
| GAP-PORT-01 | Portfolio | `_export_historical_pnl_json` returns `{}` (dict) on serialization error instead of `""` (string) — inconsistent return type. |
| GAP-PORT-02 | Portfolio | Historical data recorded for first-seen symbol only — non-deterministic if symbol arrival order varies. |
| GAP-PORT-03 | Portfolio Handler | `TradeArchiver` instantiated directly in `__init__` — not injectable for testing. |
| GAP-PORT-04 | Portfolio Handler | `save_compressed_pickle`: `zip_file.setpassword(password.encode())` has no effect on write operations. |
| GAP-PORT-05 | Portfolio Handler | `save_compressed_pickle` would raise `AttributeError` if `password=None` (`.encode()` on `None`). |
| GAP-PORT-06 | Portfolio Handler | Default export path (`Desktop/PyEventBT/`) assumes desktop environment; fails in headless/server contexts. |

### 19.10 Scheduling

| ID | Category | Description |
|---|---|---|
| GAP-SCHED-01 | Schedule Service | `remove_schedule`: uses `dict.pop(schedule)` with `Schedule` key on dict keyed by `StrategyTimeframes` — never matches. |
| GAP-SCHED-02 | Schedule Service | `remove_inactive_schedules`: accesses `key.is_active` on `StrategyTimeframes` keys — raises `AttributeError`. |
| GAP-SCHED-03 | Schedule Service | `__last_callback_args` initialized but never used. |
| GAP-SCHED-04 | Schedule Service | `TimeframeWatchInfo.__eq__` executes `ValueError(...)` without `raise` — returns `None` instead of raising. |
| GAP-SCHED-05 | Schedule Service | First bar always skipped (callbacks never fire on bar 0). |
| GAP-SCHED-06 | Schedule Service | No error handling in callback execution — exception in callback can crash event loop. |
| GAP-SCHED-07 | Schedule Service | Schedule names use `repr(callback)` — non-deterministic for lambdas/closures. |

### 19.11 Hooks

| ID | Category | Description |
|---|---|---|
| GAP-HOOK-01 | Hook Service | No `remove_hook` method — registered callbacks cannot be unregistered. |
| GAP-HOOK-02 | Hook Service | Callbacks do not receive the triggering event (only `Modules`). |
| GAP-HOOK-03 | Hook Service | No error handling in `call_callbacks` — exception in callback propagates to event loop. |
| GAP-HOOK-04 | Hook Service | No ordering/priority mechanism for callbacks. |

### 19.12 Backtest Results

| ID | Category | Description |
|---|---|---|
| GAP-BT-01 | BacktestResults | No computed statistics (Sharpe, max drawdown, CAGR, Sortino, Calmar, win rate, profit factor, etc.). |
| GAP-BT-02 | BacktestResults | `plot_old()` not deprecated. |
| GAP-BT-03 | BacktestResults | No built-in serialization method. |
| GAP-BT-04 | WalkForward | Typo in field: `retrainting_timestamps` (should be `retraining_timestamps`). |
| GAP-BT-05 | WalkForward | Typo in validator: `transform_timstamps` (should be `transform_timestamps`). |
| GAP-BT-06 | WalkForward | Path construction uses string concatenation (`path + "/file"`) instead of `os.path.join()`. |

### 19.13 SharedData / Simulator

| ID | Category | Description |
|---|---|---|
| GAP-SHARED-01 | SharedData | Pseudo-singleton: calling `SharedData()` resets ALL state including accumulated positions. |
| GAP-SHARED-02 | SharedData | `yaml.add_constructor` modifies global `yaml.SafeLoader` — affects all YAML parsing in the process. |
| GAP-SHARED-03 | SharedData | `_load_yaml_file` returns `False` on error but callers pass result to `AccountInfo(**yaml_data)` — causes `TypeError`. |
| GAP-SHARED-04 | SharedData | `open_positions`, `pending_orders`, etc. are used by external modules but not declared in class body. |
| GAP-SHARED-05 | SharedData | Not thread-safe — no locking on class-level attribute mutations. |
| GAP-SHARED-06 | SharedData | YAML files loaded at import time — unexpected import-time failures possible. |

### 19.14 Configuration Objects

| ID | Category | Description |
|---|---|---|
| GAP-CFG-01 | Mt5PlatformConfig | `password` stored as plain `str` (no `SecretStr`). |
| GAP-CFG-02 | Mt5PlatformConfig | No `path` validation (accepts any string). |
| GAP-CFG-03 | CSVBacktestDataConfig | `backtest_end_timestamp` default evaluated at import time. |
| GAP-CFG-04 | MT5LiveDataConfig | No `platform_config` field in actual source (mentioned in documentation). |
| GAP-CFG-05 | MT5SimulatedExecutionConfig | No Pydantic validators for positive balance, positive leverage. |
| GAP-CFG-06 | SizingConfigs | No `sl_pips` field on `RiskPctSizingConfig` (mentioned in some docs). |

### 19.15 Indicators

| ID | Category | Description |
|---|---|---|
| GAP-IND-01 | Indicators | Unused pandas imports (`DataFrame`, `Series`). |
| GAP-IND-02 | Indicators | All indicators implement `IIndicator` but none match its declared `compute(self, data: pd.DataFrame) -> pd.Series` interface. |
| GAP-IND-03 | Indicators | `BollingerBands` uses population std dev (divides by `period`) — may be unintentional. |
| GAP-IND-04 | Indicators | Only 4 of 11 indicators exported from `__init__.py`. |

### 19.16 Live Broker

| ID | Category | Description |
|---|---|---|
| GAP-LIVE-01 | LiveMT5Broker | `OrderSendResult`, `datetime`, `dotenv` imports unused. |
| GAP-LIVE-02 | LiveMT5Broker | Legacy `initialize_platform()` (env-var based) not removed. |
| GAP-LIVE-03 | LiveMT5Broker | If `mt5 = None` (unavailable), `__init__` raises `AttributeError` — not guarded. |
| GAP-LIVE-04 | LiveMT5Broker | `_print_account_info` calls `mt5.account_info()._asdict()` — fails with simulator's Pydantic model. |
| GAP-LIVE-05 | LiveMT5Broker | Mixed language comments (Spanish). |

### 19.17 General / Cross-Cutting

| ID | Category | Description |
|---|---|---|
| GAP-GEN-01 | Testing | No automated tests in the repository. All validation relies on running example scripts manually. |
| GAP-GEN-02 | Package | No `py.typed` marker (PEP 561 type checking support). |
| GAP-GEN-03 | Package | Mixed import styles (wildcard vs explicit). |
| GAP-GEN-04 | Package | `scikit-learn` and `scipy` listed as dependencies with no clear usage in documented modules. |
| GAP-GEN-05 | Strategy | Commented-out imports: `optimization`, `hyperopt`, `uuid` suggest removed/incomplete features. |
| GAP-GEN-06 | StrategyTimeframes | Month/year `to_timedelta()` approximations (30d, 180d, 365d) — calendar drift for multi-year backtests. |
| GAP-GEN-07 | StrategyTimeframes | `to_timedelta()` lookup dict rebuilt on every call (minor performance concern). |
| GAP-GEN-08 | StrategyTimeframes | `__gt__`/`__lt__` type hint says `str` but implementation calls `.to_timedelta()` — `AttributeError` if actual `str` passed. |
| GAP-GEN-09 | TradeArchiver | Decimal precision inconsistency: JSON export uses 5 decimal places, Parquet export uses 6. |
| GAP-GEN-10 | HyperParameter | No method to generate value list from `HyperParameterRange`. |
| GAP-GEN-11 | Quantdle | No retry logic for API failures. |
| GAP-GEN-12 | Quantdle | No intra-range gap detection. |
| GAP-GEN-13 | Quantdle | Overwrites CSV in place without backup. |

---

## 20. Derived Requirements Summary

All "Requirements Derived" items from all documentation files, consolidated and categorized.

### 20.1 Project / Root

- R-ROOT-01: The project must be installable via Poetry with Python 3.12+.
- R-ROOT-02: A CLI entry point `pyeventbt` must be available after installation.
- R-ROOT-03: Example strategies must demonstrate both backtest and live trading paths.
- R-ROOT-04: The framework must support external data sources (CSV files, Quantdle API).

### 20.2 Package Architecture

- No `py.typed` marker — should be added for PEP 561 compliance.
- Mixed import styles should be standardized.

### 20.3 Strategy API

- REQ-STRAT-001: The system shall provide a facade class that lets users register custom signal, sizing, and risk engines via Python decorators.
- REQ-STRAT-002: The system shall support pre-defined engine configurations (`MACrossoverConfig`, `MinSizingConfig`, `PassthroughRiskConfig`) as alternatives to custom engines.
- REQ-STRAT-003: The system shall allow users to register periodic scheduled callbacks at configurable timeframe intervals.
- REQ-STRAT-004: The system shall allow users to register lifecycle hooks (`ON_START`, etc.).
- REQ-STRAT-005: The `backtest()` method shall create an isolated event pipeline (fresh Queue) to support nested/sequential backtests.
- REQ-STRAT-006: The `run_live()` method shall connect to MT5 via `Mt5PlatformConfig` and run indefinitely.
- REQ-STRAT-007: Strategy IDs must be numeric strings (cast to `int` for MT5 magic numbers).
- REQ-STRAT-008: The system shall support multiple timeframes per strategy, sorted ascending, with the smallest used as the base timeframe.
- REQ-STRAT-009: Backtest duration shall be logged at WARNING level upon completion.

### 20.4 Event System

- EVT-01: All events must inherit from `EventBase` and carry an `EventType` discriminator.
- EVT-02: Bar prices must be stored as integers with a `digits` field for decimal reconstruction.
- EVT-03: `SignalEvent` must carry direction, order type, and optional price levels (SL, TP, order price).
- EVT-04: `OrderEvent` must include a concrete `volume` (converted from forecast by sizing engine).
- EVT-05: `FillEvent` must include full cost breakdown (commission, swap, fee, gross_profit) and currency denomination.
- EVT-06: `ScheduledEvent` must reference the triggering timeframe and provide both current and former timestamps.
- EVT-07: Rollover support requires a tuple of `(needs_rollover, original_contract, new_contract)` on signal and order events.

### 20.5 Trading Director

- TD-01: The event loop must dispatch events based on `EventType` to registered handler methods.
- TD-02: When the event queue is empty, the data provider must be polled for new bars.
- TD-03: At the end of a backtest, all open positions must be closed before finalizing results.
- TD-04: Live trading must sleep for a configurable heartbeat between iterations.
- TD-05: Lifecycle hooks must fire at appropriate points.
- TD-06: Scheduled callbacks must be evaluated on every bar event.
- TD-07: Schedules can be disabled globally via `run_schedules=False`.

### 20.6 Data Provider — CSV

- REQ-DP-CSV-001: Remove or explicitly deprecate the five legacy methods.
- REQ-DP-CSV-002: Remove unused `_base_minutes` and `_base_idx_map_int` cache fields.
- REQ-DP-CSV-003: Make the auxiliary FX pair list configurable or dynamic.
- REQ-DP-CSV-004: Add a mechanism to set `continue_backtest = False` when all symbol generators are exhausted.
- REQ-DP-CSV-005: Support additional account currencies beyond USD/EUR/GBP.
- REQ-DP-CSV-006: Validate that higher-TF `get_latest_bars` slice logic is correct for edge cases.
- REQ-DP-CSV-007: Use a structured logging formatter rather than inline ANSI escape codes.

### 20.7 Data Provider — MT5 Live

- REQ-DP-MT5-001: Reuse the first `get_latest_bar` call result in `update_bars` instead of calling the API twice.
- REQ-DP-MT5-002: Fix `_map_timeframe` error handling so invalid timeframes produce a clear error message.
- REQ-DP-MT5-003: Remove unused `last_bar_datetime` and `futures_tuple` attributes, or implement futures contract logic.
- REQ-DP-MT5-004: Remove or deprecate `get_latest_bar_old` and `get_latest_bars_old_pandas`.
- REQ-DP-MT5-005: Consider adding polling interval/rate limiting.
- REQ-DP-MT5-006: Add `continue_backtest` and `close_positions_end_of_data` stub attributes or make service layer check context.

### 20.8 Data Provider Configurations

- REQ-DP-CFG-001: Fix `backtest_end_timestamp` default to use `default_factory=datetime.now`.
- REQ-DP-CFG-002: Lift `tradeable_symbol_list: list[str]` to `BaseDataConfig`.
- REQ-DP-CFG-003: Add explicit generic types to all `list` annotations.
- REQ-DP-CFG-004: Remove unused `pandas` import.
- REQ-DP-CFG-005: Add `platform_config: Mt5PlatformConfig` to `MT5LiveDataConfig` if needed.

### 20.9 Execution Engine — Simulator

- R-EXEC-SIM-01: The simulator shall maintain in-memory account state reflecting all executed trades.
- R-EXEC-SIM-02: Market orders shall be filled at current bar prices and immediately emit `FillEvent`s.
- R-EXEC-SIM-03: Pending orders shall be checked against each new bar and triggered when price conditions are met.
- R-EXEC-SIM-04: SL/TP levels shall be checked on each bar and positions closed automatically when hit.
- R-EXEC-SIM-05: P&L shall be calculated with proper currency conversion.
- R-EXEC-SIM-06: Swap and commission shall be applied to positions according to instrument specifications.

### 20.10 Execution Engine — Live

- R-EXEC-LIVE-01: Market orders shall be sent via `mt5.order_send` with `TRADE_ACTION_DEAL` and `ORDER_FILLING_FOK`.
- R-EXEC-LIVE-02: Deal confirmation shall be retried for up to 5 seconds before giving up.
- R-EXEC-LIVE-03: Pending orders shall be sent via `TRADE_ACTION_PENDING` and tracked client-side.
- R-EXEC-LIVE-04: Position and order queries shall filter by strategy `magic_number`.
- R-EXEC-LIVE-05: The connector shall handle missing MT5 package gracefully (conditional import).
- R-EXEC-LIVE-06: `FillEvent`s shall be generated from `TradeDeal` objects.

### 20.11 Execution Engine Configurations

- R-EXEC-CFG-01: `MT5SimulatedExecutionConfig` shall require `initial_balance`, `account_currency`, `account_leverage`, and `magic_number`.
- R-EXEC-CFG-02: `MT5LiveExecutionConfig` shall require `magic_number` only.
- R-EXEC-CFG-03: Future versions should add Pydantic validators for positive balance, positive leverage, and valid currency codes.

### 20.12 Portfolio

- RQ-PORT-001: Portfolio must track balance, equity, realised PnL, and unrealised PnL using `Decimal` precision.
- RQ-PORT-002: Portfolio must delegate to the execution engine for position/order state and account values on every base-timeframe bar.
- RQ-PORT-003: In backtest mode, portfolio must record historical balance and equity at each base-timeframe bar.
- RQ-PORT-004: Portfolio must support filtering positions and pending orders by symbol and ticket.
- RQ-PORT-005: Portfolio must support exporting historical PnL in Parquet (zstd), CSV, JSON, and DataFrame formats.
- RQ-PORT-006: At backtest end, portfolio must update final state and set equity equal to balance.

### 20.13 Portfolio Handler

- RQ-PH-001: PortfolioHandler must process bar events only for the base timeframe.
- RQ-PH-002: Signal events must pass through the sizing engine and then the risk engine before order placement.
- RQ-PH-003: Fill events must be archived for historical record-keeping.
- RQ-PH-004: At backtest end, the handler must finalize portfolio state and return a `BacktestResults` object.
- RQ-PH-005: Backtest results must be optionally exportable to CSV and/or Parquet formats.
- RQ-PH-006: Export directory must default to the OS-specific Desktop path when no custom directory is configured.

### 20.14 Signal Engine

- REQ-SIGSVC-01: The service must support both predefined signal engine configs and user-supplied custom engine callables.
- REQ-SIGSVC-02: Generated `SignalEvent`(s) must be placed onto the shared events queue.
- REQ-SIGSVC-03: The service must handle single `SignalEvent` and `list[SignalEvent]` returns uniformly.
- REQ-SIGSVC-04: `None` return values must be silently discarded.
- REQ-SIGMAC-01: The MA crossover engine must only process bar events matching the configured `signal_timeframe`.
- REQ-SIGMAC-02: BUY signals must only be emitted when no long position is currently open.
- REQ-SIGMAC-03: SELL signals must only be emitted when no short position is currently open.
- REQ-SIGMAC-04: Opposite-side positions must be closed before emitting a new directional signal.
- REQ-SIGMAC-05: The engine must support both SIMPLE and EXPONENTIAL moving average types.
- REQ-SIGMAC-06: Signal generation must degrade gracefully when insufficient historical data is available.
- REQ-SIGCFG-01: Every signal engine config must provide `strategy_id` and `signal_timeframe`.
- REQ-SIGCFG-02: `MACrossoverConfig` must support both fixed integer periods and `HyperParameter` objects.
- REQ-SIGCFG-03: `MAType` must enumerate at least SIMPLE and EXPONENTIAL variants.

### 20.15 Sizing Engine

- R-SIZING-SVC-01: The service must default to minimum-lot sizing when no explicit config is provided.
- R-SIZING-SVC-02: Custom sizing functions must accept `(SignalEvent, Modules)` and return `SuggestedOrder`.
- R-SIZING-SVC-03: The service must pass `modules` to the engine for engines requiring runtime data.
- R-SIZING-CFG-01: All sizing configurations must extend `BaseSizingConfig`.
- R-SIZING-CFG-02: `FixedSizingConfig.volume` must be a `Decimal`.
- R-SIZING-CFG-03: `RiskPctSizingConfig.risk_pct` must be greater than 0.
- R-RISKPCT-01: The engine must require a non-zero stop-loss on every signal event.
- R-RISKPCT-02: The engine must convert tick value to account currency when profit currency differs.
- R-RISKPCT-03: The calculated volume must be rounded to the symbol's `volume_step`.
- R-RISKPCT-04: The engine should validate that the final volume falls within `[volume_min, volume_max]`.
- R-RISKPCT-05: Currency conversion should support a broader set of FX pairs or use a dynamic lookup.

### 20.16 Risk Engine

- R-RISK-SVC-01: `RiskEngineService` shall delegate order evaluation to the configured concrete `IRiskEngine`.
- R-RISK-SVC-02: Orders with approved volume ≤ 0 shall be silently discarded.
- R-RISK-SVC-03: The service shall support runtime replacement of risk logic via a callable injection method.
- R-RISK-SVC-04: Created `OrderEvent`s shall faithfully copy all fields from the originating `SignalEvent`.
- R-RISK-CFG-01: The system shall provide a `BaseRiskConfig` base class.
- R-RISK-CFG-02: `PassthroughRiskConfig` shall require no parameters and map to the passthrough engine.
- R-RISK-CFG-03: Future risk configurations should support numeric thresholds validated by Pydantic.

### 20.17 Scheduling

- SS-01: Users must be able to register callbacks that fire at regular timeframe intervals.
- SS-02: Scheduling must be driven by bar timestamps, not wall-clock time.
- SS-03: Schedules must be individually or globally activate-able and deactivate-able.
- SS-04: Each callback invocation must receive a `ScheduledEvent` with current and former timestamps plus a `Modules` instance.
- SS-05: The first bar must initialize timestamp tracking without triggering callbacks.
- SS-06: Schedules can be disabled entirely when `run_schedules=False`.

### 20.18 Hooks

- HK-01: Users must be able to register callbacks for lifecycle events (start, signal, order, end).
- HK-02: Hooks must be globally enable/disable-able.
- HK-03: Callbacks must receive a `Modules` instance for access to system components.
- HK-04: Multiple callbacks per hook must be supported, executing in registration order.

### 20.19 Indicators

- R-IND-01: All indicators must accept numpy arrays and return numpy arrays or tuples of numpy arrays.
- R-IND-02: Indicators must raise `ValueError` for insufficient input data length.
- R-IND-03: NaN values must fill positions where the indicator cannot yet be computed.
- R-IND-04: Numba `@njit` must be applied to inner computation loops for performance.
- R-IND-05: Each indicator must implement `IIndicator`.

### 20.20 Backtest Results

- R-BT-01: `BacktestResults` must store PnL and trades DataFrames from a completed backtest.
- R-BT-02: Equity returns must be computed as percentage change of the EQUITY column.
- R-BT-03: A `plot()` method must render EQUITY and BALANCE curves via matplotlib.
- R-TA-01: Every FillEvent must be archived with a unique incrementing integer ID.
- R-TA-02: Export to pandas DataFrame must include all FillEvent fields with enum values resolved.
- R-TA-03: Parquet export must use zstd compression with an explicit schema.
- R-TA-04: Export methods must create parent directories if they do not exist.
- R-TA-05: JSON export must handle serialization errors gracefully.
- REQ-WF-001: The system shall support anchored and unanchored walk-forward optimization modes.
- REQ-WF-002: Walk-forward results shall be serializable to and from CSV files.
- REQ-WF-003: Walk-forward results shall track retraining timestamps and hyperparameter values.
- REQ-WF-004: `hyperparameters_track` shall accept both `pd.DataFrame` and `List[Dict]` inputs.

### 20.21 Live Trading Broker

- REQ-LIVE-001: The live broker must verify account type and display appropriate warnings (longer delay for REAL).
- REQ-LIVE-002: All symbols must be added to MarketWatch before trading begins.
- REQ-LIVE-003: Algorithmic trading must be verified as enabled before allowing the trading loop.
- REQ-LIVE-004: Connection health must be queryable via `is_connected()` and `is_closed()`.
- R-CFG-01: MT5 live trading must require explicit connection parameters.
- R-CFG-02: Configuration must be serializable to/from YAML.
- R-CFG-03: Sensitive credentials should be handled with care.

### 20.22 SharedData / Simulator

- REQ-SHARED-001: Default state must be loaded from YAML files.
- REQ-SHARED-002: All financial values in YAML files must be loaded as `Decimal`.
- REQ-SHARED-003: `SharedData` must be the single source of truth for simulator state.
- REQ-SHARED-004: Default account configuration must provide balance=10000, leverage=30, currency=USD.
- REQ-SIM-CONN-001: `initialize()` must validate credentials and propagate login/server to `AccountInfo`.
- REQ-SIM-CONN-002: `symbols_get()` must support wildcard pattern matching with inclusion and exclusion.
- REQ-SIM-CONN-003: All connector methods must set `SharedData.last_error_code`.
- REQ-SIM-CONN-004: Symbol selection must update both `select` and `visible` flags.

### 20.23 Session Configurations

- TSC-01: Backtest sessions require initial capital, start date, and backtest name.
- TSC-02: Live sessions require a symbol list, heartbeat interval, and MT5 platform configuration.
- TSC-03: Session configuration must be polymorphic via a common base type.

### 20.24 Timeframes

- REQ-TF-001: The system shall support 22 distinct bar timeframes from 1 minute to 1 year.
- REQ-TF-002: Timeframe values shall be compatible with pandas frequency codes.
- REQ-TF-003: Timeframes shall be orderable by duration.
- REQ-TF-004: Timeframes shall be usable as dictionary keys (hashable).

### 20.25 HyperParameter

- R-HP-01: Strategy parameters must support both continuous ranges and discrete value lists.
- R-HP-02: Each hyper-parameter must carry a name, current value, and its search space definition.
- R-HP-03: The framework must support numeric types (`float` and `int`) for parameter values.

### 20.26 Quantdle

- REQ-DP-QDL-001: Add intra-range gap detection to fill missing data within existing CSV files.
- REQ-DP-QDL-002: Add retry logic with exponential backoff for API download failures.
- REQ-DP-QDL-003: Validate timeframe in `_convert_to_quantdle_timeframe` and raise clear error for unsupported values.
- REQ-DP-QDL-004: Create backup of existing CSV before overwriting.
- REQ-DP-QDL-005: Add schema validation for Quantdle API responses.
- REQ-DP-QDL-006: Replace Unicode symbols in log messages with ASCII alternatives.

### 20.27 Logging

- REQ-LOG-001: The system shall allow users to configure logging verbosity at Strategy construction time.
- REQ-LOG-002: Supported logging levels shall match Python standard library logging levels.

### 20.28 Modules (Dependency Injection)

- REQ-MOD-001: The `Modules` container shall provide user callbacks with access to data provider, execution engine, portfolio, and trading context.
- REQ-MOD-002: The `Modules` container shall support arbitrary (non-serializable) types as field values.
- REQ-MOD-003: A single `Modules` instance shall be shared across all callbacks within a session.

### 20.29 Portfolio Entities

- RQ-OPOS-001: An open position must have defined entry time, entry price, direction, symbol, unique ticket, volume, and strategy ID.
- RQ-OPOS-002: Stop-loss, take-profit, swap, and comment are optional metadata fields.
- RQ-OPOS-003: The model must support multiple positions per symbol (MT5 hedging account model).
- RQ-CPOS-001: A closed position must record both entry and exit timestamps and prices.
- RQ-CPOS-002: A closed position must include the realised PnL and commission.
- RQ-CPOS-003: A closed position must be identifiable by ticket and strategy_id.
- RQ-PORD-001: A pending order must specify trigger price, order type, symbol, unique ticket, volume, and strategy ID.
- RQ-PORD-002: Supported pending order types must include BUY_LIMIT, SELL_LIMIT, BUY_STOP, SELL_STOP.
- RQ-PORD-003: Stop-loss, take-profit, and comment are optional metadata fields.
