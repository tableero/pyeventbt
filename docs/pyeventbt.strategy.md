# pyeventbt.strategy

**Package**: `pyeventbt.strategy`

**Purpose**: Top-level package providing the user-facing `Strategy` facade that wires together all framework components (data providers, execution engines, signal/sizing/risk engines, portfolio, and the trading director event loop) for both backtesting and live trading workflows.

**Tags**: `#facade` `#user-api` `#orchestration` `#backtest` `#live-trading`

---

## Modules

| Module | File | Description |
|---|---|---|
| `pyeventbt.strategy.strategy` | `strategy.py` | Main `Strategy` class -- the primary user-facing entry point for configuring and running backtests or live trading sessions. |
| `pyeventbt.strategy.core` | `core/` | Sub-package containing data models, enums, and configuration types used by the Strategy class. |
| `pyeventbt.strategy.services` | `services/` | Sub-package containing service classes that support strategy execution (e.g., parameter storage). |
| `pyeventbt.strategy.__init__` | `__init__.py` | Package initializer. Empty beyond the license header; no public re-exports. |

---

## Internal Architecture

The `Strategy` class acts as a **builder and orchestrator**. Users interact exclusively with `Strategy` to:

1. **Register engines** via decorators (`@strategy.custom_signal_engine`, `@strategy.custom_sizing_engine`, `@strategy.custom_risk_engine`) or pre-defined configuration methods (`configure_predefined_signal_engine`, etc.).
2. **Register scheduled callbacks** via `@strategy.run_every(interval)`.
3. **Register hooks** via `@strategy.hook(Hooks.ON_START)`.
4. **Launch execution** via `strategy.backtest(...)` or `strategy.run_live(...)`.

When `backtest()` or `run_live()` is called, Strategy instantiates:
- A `Queue` (the shared event bus)
- `DataProvider` (CSV-backed for backtest, MT5-backed for live)
- `ExecutionEngine` (simulated or live MT5)
- `Portfolio`
- `Modules` (Pydantic model bundling the above for injection into user callbacks)
- `SignalEngineService`, `SizingEngineService`, `RiskEngineService`
- `PortfolioHandler`
- `TradingDirector` (owns the event loop)

All scheduled events are forwarded to `TradingDirector.add_schedule()`, and then `TradingDirector.run()` is called to start the event loop.

```
User code
  |
  v
Strategy (facade)
  |
  +---> Queue (event bus)
  +---> DataProvider ---> BarEvent ---> Queue
  +---> TradingDirector (event loop)
  |       |
  |       +---> SignalEngineService ---> SignalEvent ---> Queue
  |       +---> PortfolioHandler
  |               +---> SizingEngineService ---> SuggestedOrder
  |               +---> RiskEngineService ---> validated order
  |               +---> ExecutionEngine ---> FillEvent ---> Queue
  |               +---> Portfolio (state tracking)
  +---> HookService (lifecycle hooks)
```

---

## Cross-Package Dependencies

| Dependency Package | What Is Used |
|---|---|
| `pyeventbt.backtest` | `BacktestResults` |
| `pyeventbt.core` | `HyperParameter` |
| `pyeventbt.hooks` | `HookService`, `Hooks` |
| `pyeventbt.trading_context` | `TypeContext` |
| `pyeventbt.trading_director` | `TradingDirector`, `MT5BacktestSessionConfig`, `MT5LiveSessionConfig` |
| `pyeventbt.signal_engine` | `SignalEngineService`, `MACrossoverConfig`, `ISignalEngine` |
| `pyeventbt.sizing_engine` | `SizingEngineService`, `MinSizingConfig`, `RiskPctSizingConfig`, `FixedSizingConfig` |
| `pyeventbt.risk_engine` | `RiskEngineService`, `PassthroughRiskConfig` |
| `pyeventbt.data_provider` | `DataProvider`, `CSVBacktestDataConfig`, `MT5LiveDataConfig` |
| `pyeventbt.execution_engine` | `ExecutionEngine`, `MT5SimulatedExecutionConfig`, `MT5LiveExecutionConfig` |
| `pyeventbt.portfolio` | `Portfolio`, `IPortfolio` |
| `pyeventbt.portfolio_handler` | `PortfolioHandler`, `SuggestedOrder` |
| `pyeventbt.events` | `BarEvent`, `ScheduledEvent`, `SignalEvent` |
| `pyeventbt.config` | `Mt5PlatformConfig` |
| `pyeventbt.utils` | `LoggerColorFormatter`, `TerminalColors`, `colorize` |
| `pydantic` | `BaseModel`, `ConfigDict`, `field_validator` |
| `pandas` | `pd.Timestamp`, `pd.DataFrame` |

---

## Gaps & Issues

1. **Commented-out imports** in `strategy.py`: optimization/hyperopt/uuid imports are commented out, suggesting incomplete or removed optimization features.
2. **FIXME comments**: Two identical FIXMEs in `strategy.py` (lines 394, 500) indicate planned refactoring to move TradingDirector logic into Strategy.
3. **`__init__.py` files are empty**: No public API is re-exported from any `__init__.py` in this package tree. Users must import from the full dotted path or rely on the top-level `pyeventbt/__init__.py`.
4. **`setdefault` for engine registration**: Using `dict.setdefault()` means only the first registration for a given `strategy_id` is kept; subsequent registrations are silently ignored. This could confuse users who expect later registrations to override earlier ones.
5. **Magic number creation unused in backtest**: `__create_mg_for_strategy_id` exists but is commented out in `backtest()` -- the `strategy_id` string is cast directly to `int` instead.
6. **Mutable default arguments**: `strategy_timeframes` and `symbols_to_trade` parameters use mutable list defaults (`[StrategyTimeframes.ONE_MIN]`, `['EURUSD']`), which is a known Python anti-pattern.
