# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## Commands

```bash
# Install dependencies
poetry install

# Run a strategy example
python example_ma_crossover.py
python example_bbands_breakout.py

# Check version
pyeventbt --version

# Build/publish (used by CI)
poetry build
poetry publish
```

There are no automated tests in this repository.

## Architecture

PyEventBT is an event-driven backtesting and live trading framework for MetaTrader 5 (MT5). The core pattern is a shared `Queue` through which all components communicate via typed events.

### Event Loop

`TradingDirector` (`pyeventbt/trading_director/trading_director.py`) owns the main loop. It dequeues events and dispatches to handlers:

- `BAR` → `PortfolioHandler.process_bar_event` → `ScheduleService` → `SignalEngineService.generate_signal`
- `SIGNAL` → `PortfolioHandler.process_signal_event` (runs sizing + risk engines, emits `ORDER`)
- `ORDER` → `ExecutionEngine._process_order_event` (executes trade, emits `FILL`)
- `FILL` → `PortfolioHandler.process_fill_event` (updates portfolio state)

The `DataProvider` feeds `BarEvent`s into the queue when the queue is empty (`update_bars()`).

### Key Components

| Module | Role |
|---|---|
| `Strategy` | User-facing facade. Decorators register signal/sizing/risk engines; `.backtest()` / `.run_live()` wire everything and start the loop |
| `TradingDirector` | Event loop; dispatches events to services |
| `DataProvider` | Feeds bar data (CSV for backtest, MT5 live connection for live) |
| `SignalEngineService` | Calls the user's `@strategy.custom_signal_engine` function, emits `SignalEvent`s |
| `SizingEngineService` | Converts `SignalEvent` → `SuggestedOrder` with position size |
| `RiskEngineService` | Validates/filters `SuggestedOrder`s before order placement |
| `ExecutionEngine` | Places orders via MT5 simulator or live MT5; emits `FillEvent`s |
| `Portfolio` | Tracks open/closed positions and account balance |
| `PortfolioHandler` | Orchestrates sizing → risk → execution pipeline |
| `ScheduleService` | Fires `@strategy.run_every(timeframe)` callbacks on each bar |

### User-Facing API (`pyeventbt/__init__.py`)

Users import from the `pyeventbt` package directly:
- `Strategy`, `Modules`, `StrategyTimeframes` — core strategy building blocks
- `BarEvent`, `SignalEvent`, `OrderEvent`, `FillEvent` — event types
- `PassthroughRiskConfig`, `MinSizingConfig`, `FixedSizingConfig`, `RiskPctSizingConfig` — predefined engine configs
- `Mt5PlatformConfig` — live trading MT5 connection config
- `HyperParameter`, `Variable` — for parameter management
- `QuantdleDataUpdater` — downloads historical data from Quantdle
- `BacktestResults` — returned by `.backtest()`; has a `.plot()` method
- `indicators` submodule — `SMA`, `KAMA`, `BollingerBands`, `ATR`, `RSI`, etc. (Numba-accelerated)

### Backtest vs Live

Both paths share the same engine pipeline. The difference is the connector layer:
- **Backtest**: `CSVBacktestDataConfig` + `MT5SimulatedExecutionConfig` (uses `mt5_simulator_connector.py`)
- **Live**: `MT5LiveDataConfig` + `MT5LiveExecutionConfig` (uses `live_mt5_broker.py`)

MT5 import errors are handled gracefully — the package loads without MT5 installed (for backtest-only use).

### Internal Conventions

- Each module follows a `core/interfaces/`, `core/entities/`, `core/configurations/`, `services/`, `connectors/` layout
- `strategy_id` maps to MT5 Magic Numbers; must be a string of digits
- Bar prices are stored as integers for compactness with a `digits` field to reconstruct decimals (`close / 10**digits`)
- `Modules` is a Pydantic model passed into every user callback — provides access to `DATA_PROVIDER`, `EXECUTION_ENGINE`, `PORTFOLIO`, and `TRADING_CONTEXT`
- `polars` DataFrames are returned by `DATA_PROVIDER.get_latest_bars()`; use `.select('close').to_numpy().flatten()` to pass to indicators
