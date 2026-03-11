# Project Root

**Package**: `pyeventbt`
**Purpose**: Event-driven backtesting and live trading framework for Python and MetaTrader 5.
**Tags**: `#project-root` `#pyproject` `#examples` `#configuration`

---

## Project Metadata

| Field | Value |
|---|---|
| Name | pyeventbt |
| Version | 0.0.4 |
| License | Apache-2.0 |
| Authors | Marti Castany, Alain Porto |
| Python | ^3.12 |
| Homepage | https://github.com/marticastany/pyeventbt |
| Development Status | 1 - Planning |

---

## Dependencies (pyproject.toml)

| Package | Version Constraint | Role |
|---|---|---|
| python-dotenv | ^1.1.1 | Environment variable loading |
| pydantic | ^2.12.3 | Data validation and configuration models |
| numpy | ^2.0.0 | Numerical computation, indicator calculations |
| polars | ^1.35.0 | DataFrame operations for bar data |
| pandas | ^2.2.3 | Timestamp handling, time series utilities |
| numba | ^0.62.1 | JIT compilation for indicator performance |
| matplotlib | ^3.7.0 | Backtest result plotting |
| scipy | ^1.10.0 | Scientific computing utilities |
| scikit-learn | ^1.3.0 | Machine learning utilities |
| PyYAML | ^6.0 | YAML config file serialization |

---

## CLI Entry Point

Defined in `[tool.poetry.scripts]`:

```
pyeventbt = pyeventbt.app:main
```

Running `pyeventbt` invokes `main()` in `pyeventbt/app.py`. Supports `--version` flag and `info` command.

---

## Root-Level Files

| File | Description |
|---|---|
| `pyproject.toml` | Poetry project configuration, dependencies, build system |
| `README.md` | Project readme (referenced by pyproject.toml) |
| `example_ma_crossover.py` | Moving Average crossover strategy example using SMA fast/slow crossover on daily bars |
| `example_bbands_breakout.py` | Bollinger Bands breakout strategy example with intraday entry/exit timing |
| `example_quantdle_ma_crossover.py` | MA crossover example with QuantdleDataUpdater for automatic data downloading |

---

## Example Files

### example_ma_crossover.py

Demonstrates the core user-facing API:
1. Creates a `Strategy` instance with logging level
2. Registers a `@strategy.custom_signal_engine` decorated function that receives `BarEvent` and `Modules`
3. Computes SMA fast (10) and slow (30) on daily close prices via `modules.DATA_PROVIDER.get_latest_bars()`
4. Emits `SignalEvent` for BUY/SELL based on MA crossover
5. Configures `MinSizingConfig` and `PassthroughRiskConfig`
6. Calls `strategy.backtest()` with CSV data directory, date range, and initial capital
7. Calls `backtest.plot()` to visualize results

Includes commented-out `strategy.run_live()` example with `Mt5PlatformConfig`.

### example_bbands_breakout.py

Demonstrates:
- Multi-timeframe strategy (`ONE_HOUR` + `ONE_DAY`)
- `BollingerBands.compute()` indicator usage
- STOP order types (BUY_STOP, SELL_STOP) via `order_type="STOP"` in `SignalEvent`
- Intraday scheduling: order placement at 08:00, position close at 21:00
- Daily state tracking with module-level dictionaries
- Position/order management via `modules.EXECUTION_ENGINE` and `modules.PORTFOLIO`

### example_quantdle_ma_crossover.py

Demonstrates:
- `QuantdleDataUpdater` initialization with API key and key ID
- `updater.update_data()` to download and cache CSV data for symbols
- Same MA crossover logic as `example_ma_crossover.py` but with Quantdle-sourced data

---

## Build System

```toml
[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

Build and publish via:
```bash
poetry build
poetry publish
```

---

## Gaps & Issues

1. **No automated tests**: The repository contains no test suite. All validation relies on running example scripts manually.
2. **Development status**: Classified as "1 - Planning" despite having functional code and published examples.
3. **`csv_dir = None` in examples**: The MA crossover example sets `csv_dir = None` with a comment suggesting the user should provide their own path. Behavior when `csv_dir` is `None` is undocumented at this level.
4. **Hardcoded paths in examples**: Commented-out paths reference `/Users/marticastany/Desktop/`, which are developer-specific.
5. **Quantdle example has placeholder API keys**: Uses `"your_api_key_here"` strings that will fail at runtime if not replaced.

---

## Requirements Derived

- R-ROOT-01: The project must be installable via Poetry with Python 3.12+.
- R-ROOT-02: A CLI entry point `pyeventbt` must be available after installation.
- R-ROOT-03: Example strategies must demonstrate both backtest and live trading paths.
- R-ROOT-04: The framework must support external data sources (CSV files, Quantdle API).
