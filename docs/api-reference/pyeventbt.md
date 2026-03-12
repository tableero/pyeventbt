# Package: pyeventbt

**Package**: `pyeventbt`
**Purpose**: Top-level Python package for the PyEventBT event-driven backtesting and live trading framework. Serves as the public API surface and aggregates all user-facing components.
**Tags**: `#package` `#top-level` `#public-api` `#facade`

---

## Modules

| Module | Description |
|---|---|
| `__init__.py` | Public API re-exports: Strategy, events, configs, indicators, entities, results |
| `app.py` | CLI entry point with `--version` and `info` command |
| `config/` | MT5 platform configuration and base config model |
| `core/` | Core entities: HyperParameter, Variable |
| `utils/` | Utility functions: date formatting, currency conversion, order type mapping, logging |
| `strategy/` | Strategy class, Modules facade, timeframe definitions (not in scope of this doc set) |
| `events/` | Event types: BarEvent, SignalEvent, OrderEvent, FillEvent (not in scope) |
| `indicators/` | Technical indicators: SMA, KAMA, BollingerBands, ATR, RSI, etc. (not in scope) |
| `risk_engine/` | Risk engine configurations and services (not in scope) |
| `sizing_engine/` | Sizing engine configurations and services (not in scope) |
| `data_provider/` | Data provider, bar entities, QuantdleDataUpdater (not in scope) |
| `portfolio/` | Portfolio tracking for open/closed positions (not in scope) |
| `backtest/` | BacktestResults and plotting (not in scope) |
| `trading_director/` | Main event loop dispatcher (not in scope) |
| `execution_engine/` | Order execution: MT5 simulator and live broker (not in scope) |

---

## Internal Architecture

The `pyeventbt` package follows a modular architecture where each subdirectory represents a bounded domain:

```
pyeventbt/
  __init__.py          <-- Public API surface (re-exports)
  app.py               <-- CLI
  config/              <-- Configuration models
  core/                <-- Shared entities (HyperParameter, Variable)
  utils/               <-- Cross-cutting utilities
  strategy/            <-- Strategy facade and decorators
  events/              <-- Event type definitions
  indicators/          <-- Technical indicator implementations
  risk_engine/         <-- Risk validation pipeline
  sizing_engine/       <-- Position sizing pipeline
  data_provider/       <-- Market data ingestion
  portfolio/           <-- Position and account tracking
  backtest/            <-- Backtest result analysis
  trading_director/    <-- Event loop orchestration
  execution_engine/    <-- Trade execution connectors
```

Each subpackage typically follows an internal convention:
- `core/interfaces/` -- abstract base classes
- `core/entities/` -- data models
- `core/configurations/` -- config models
- `services/` -- business logic
- `connectors/` -- external system adapters

---

## Cross-Package Dependencies

The `pyeventbt` package depends on:
- `pydantic` for all model definitions
- `polars` and `pandas` for data handling
- `numpy` and `numba` for indicator computation
- `matplotlib` for result visualization
- `PyYAML` for config serialization
- `importlib.metadata` for version resolution

---

## Gaps & Issues

1. **Incomplete public API documentation**: The `__init__.py` docstring references `https://pyeventbt.com` which may not be live.
2. **Mixed import styles**: Some subpackages use wildcard imports (`from .configs import *`) while others use explicit imports.
3. **No `py.typed` marker**: The package does not include a `py.typed` file for PEP 561 type checking support.
4. **Large dependency footprint**: `scikit-learn` and `scipy` are listed as dependencies but their usage is unclear from the documented modules.
