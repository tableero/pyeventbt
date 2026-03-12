# Technical Review — Dependencies, Architecture & Runtime Issues

> Extracted from the main documentation. See also: [REQUIREMENTS.md](REQUIREMENTS.md) Section 21 for the same findings in requirements format.

This section documents all known issues found during a thorough code review, organized by severity. Each issue includes the affected file, line numbers, and impact assessment. This serves as both a transparency document and a prioritized backlog for future development.

### 30.1 Severity Definitions

| Severity | Meaning |
|---|---|
| **CRITICAL** | Can cause data loss, crashes in production, or security issues. Must be fixed before any production deployment. |
| **HIGH** | Causes incorrect behavior, memory issues, or breaks expected contracts. Should be fixed in the next release. |
| **MEDIUM** | Code quality, maintainability, or correctness issues that degrade the codebase over time. |
| **LOW** | Minor issues, cosmetic problems, or opportunities for improvement. |

---

### 30.2 CRITICAL Issues

#### 30.2.1 No Automated Tests

**Impact:** Every other issue in this list is unvalidated. No regression detection, no correctness verification for indicators, no stress testing for the event queue. Manual testing via example scripts is the only validation.

**Files affected:** Entire codebase.

**Risk:** Any code change can silently break backtest correctness, live trading safety, or indicator accuracy with no automated detection.

#### 30.2.2 SharedData Singleton — Global Mutable State Without Thread Safety

**File:** `pyeventbt/broker/mt5_broker/shared/shared_data.py:19-28`

`SharedData` uses **class-level attributes** as a pseudo-singleton. Multiple components read and write these attributes with zero synchronization:

```python
class SharedData():
    last_error_code: tuple
    credentials: InitCredentials = None
    terminal_info: TerminalInfo = None
    account_info: AccountInfo = None
    symbol_info: dict[str, SymbolInfo] = None
```

**Sub-issues:**

| Issue | File:Line | Description |
|---|---|---|
| Direct `__dict__` mutation | `mt5_simulator_execution_engine_connector.py:145-152` | Bypasses Pydantic validation: `SharedData.account_info.__dict__["balance"] = self.balance` |
| YAML constructor leak | `shared_data.py:51` | `yaml.add_constructor()` registered on every `_load_yaml_file()` call — leaks in long-running sessions |
| Import-time side effects | `mt5_simulator_wrapper.py:28` | `SharedData()` called at class definition time — mutates global state on import |
| No reset mechanism | `shared_data.py` | Cannot isolate state between backtest runs in same process |
| `None` without `Optional` | `shared_data.py:25-28` | Type hints say `dict[str, SymbolInfo]` but default is `None` — accessing without None check crashes |

#### 30.2.3 No Event Validation in the Dispatch Loop

**File:** `trading_director/trading_director.py:154,190`

```python
self.event_handlers_dict[event.type](event)
```

**Sub-issues:**

| Issue | Impact |
|---|---|
| `ScheduledEvent` (type `SCHEDULED_EVENT`) has no handler entry → `KeyError` crashes the loop | Backtest terminates unexpectedly |
| No exception handling around handlers — a single malformed event kills the entire run | No recovery, partial results lost |
| No event field validation before dispatch | Missing `symbol` or `time_generated` → unhandled `AttributeError` deep in handler |

#### 30.2.4 Live Trading Has No Reconnection or Recovery Logic

**File:** `broker/mt5_broker/connectors/live_mt5_broker.py:210-232`

- `is_connected()` is a query method, not a monitor — no automatic reconnection on disconnect
- No retry logic on `mt5.initialize()` failure
- No circuit breaker — if 10 consecutive orders fail, system keeps trying without backoff
- No graceful shutdown for the live loop (`while True` with no break condition or signal handler)
- Deal confirmation retry blocks the event loop for up to 5 seconds of `time.sleep()` (`mt5_live_execution_engine_connector.py`)

#### 30.2.5 Missing Dependency in `pyproject.toml`

**File:** `pyproject.toml`

`quantdle` is used by `QuantdleDataUpdater` (exported in `pyeventbt/__init__.py`) but is only listed in `requirements.txt`, **not in `pyproject.toml`**. Users installing via `pip install pyeventbt` will get `ImportError` when using `QuantdleDataUpdater`.

---

### 30.3 HIGH Issues

#### 30.3.1 Strategy Decorators Return None

**File:** `strategy/strategy.py:162-165, 195-198, 230-233`

All custom engine decorators (`@strategy.custom_signal_engine`, `@strategy.custom_sizing_engine`, `@strategy.custom_risk_engine`) register the function but do **not return it**. The decorated function becomes `None`, making it uncallable outside the decorator context.

```python
def decorator(fn):
    self.__signal_engines.setdefault(strategy_id, fn)
    # Missing: return fn
return decorator
```

#### 30.3.2 Unbounded Memory Growth

| Structure | File:Line | Growth rate | Impact |
|---|---|---|---|
| `executed_deals: dict` | `mt5_simulator_execution_engine_connector.py:55` | Grows per trade, never pruned | 10-year backtest with 1000 trades/year → 10K objects in memory |
| `historical_balance: dict` | `portfolio.py:45` | Grows per base-timeframe bar | 10-year 1-min backtest → ~5.25M entries |
| `historical_equity: dict` | `portfolio.py:46` | Same as above | ~5.25M entries |
| `queue.Queue` | No `maxsize` set | Unbounded if handlers are slow | Can consume all available RAM |

#### 30.3.3 Bare `except:` Blocks

**File:** `data_provider/connectors/mt5_live_data_connector.py:133, 239`

Bare `except:` catches `SystemExit` and `KeyboardInterrupt` — the user cannot Ctrl+C to stop live trading. Should be `except Exception:`.

#### 30.3.4 Float Precision Loss in Price Encoding

**File:** `data_provider/connectors/csv_data_connector.py:379-387`

```python
pl.col(c).mul(pl.lit(scale, pl.Float64)).floor().cast(pl.Int64)
```

For EURUSD with 5 digits, `scale = 100000.0`. Due to IEEE 754 rounding: `1.12345 * 100000.0` → `112344.999999` → `.floor()` → `112344` (loses last digit). This affects every bar in every backtest.

**Fix:** Use `round()` instead of `floor()`, or convert to Decimal before scaling.

#### 30.3.5 End-of-Backtest Position Closing Has No Timeout

**File:** `trading_director/trading_director.py:158-167`

If fill generation has a bug, the close-all loop runs indefinitely with no max-iteration guard. The backtest hangs forever.

#### 30.3.6 Mutable Default Arguments in Strategy API

**File:** `strategy/strategy.py`

| Issue | Description |
|---|---|
| `end_date` default is `datetime.now()` | Evaluated at module load time, not at call time — all backtests share the same end date |
| `strategy_timeframes=[]`, `symbols_to_trade=[]` | Mutable list defaults shared across calls |
| `HookService()` default in TradingDirector | Shared instance across calls that omit this parameter |

---

### 30.4 MEDIUM Issues

#### 30.4.1 Version and Documentation Inconsistencies

| Location | Current value | Should be |
|---|---|---|
| `pyproject.toml` | `0.0.5` | (source of truth) |
| `docs/REQUIREMENTS.md:15` | `0.0.4` | `0.0.5` |
| `DOCUMENTATION.md:3` | `0.0.4` | `0.0.5` |
| PyPI classifier | `Development Status :: 1 - Planning` | `3 - Alpha` or `4 - Beta` |
| Poetry version in CI | `version: latest` (unpinned) | Should pin to specific version |

#### 30.4.2 Decimal/Float Mixing Throughout the Codebase

| Location | Issue |
|---|---|
| `events.py:121` | `SignalEvent.forecast` is `float`, all other financial fields are `Decimal` |
| `indicators.py:72-97` | `Bar.close_f` returns float; downstream wraps in `Decimal(str(...))` |
| `mt5_simulator_execution_engine_connector.py:366-375` | Compares `Decimal` SL/TP with `float` bar prices |
| General | `Decimal(str(float_value))` pattern used throughout — captures float representation errors |

#### 30.4.3 Hardcoded Values That Should Be Configuration

| Value | File:Line | Purpose |
|---|---|---|
| `200000000` | `mt5_simulator_execution_engine_connector.py:65` | Ticket counter start |
| `300000000` | `mt5_simulator_execution_engine_connector.py:66` | Deal counter start |
| `999999999999` | `mt5_simulator_execution_engine_connector.py:684,706` | Sentinel for "not found" |
| `volume * Decimal('2.5')` | `mt5_simulator_execution_engine_connector.py:242` | Darwinex commission rate |
| FX/commodity/index lists | `mt5_simulator_execution_engine_connector.py:73-75` | Instrument classification |
| `"TODO: remove the = part"` | `mt5_simulator_execution_engine_connector.py:281` | Test logic left in production |
| 33 auxiliary FX pairs | `csv_data_connector.py` | Hardcoded cross-rate pair list |
| `USD`, `EUR`, `GBP` only | `csv_data_connector.py` | Supported account currencies |
| `Desktop/PyEventBT/` | `portfolio_handler.py` | Default export path — fails on headless servers |

#### 30.4.4 BollingerBands Uses Population Variance

**File:** `indicators/indicators.py:555`

```python
std = np.sqrt(variance / period)  # Population variance (N), not sample variance (N-1)
```

For period=5, this underestimates standard deviation by ~11%. Most trading platforms use sample variance (N-1).

#### 30.4.5 Logging Inconsistencies

- 28+ `print()` statements mixed with `logger.info/error/warning`
- ANSI color codes embedded directly in log messages — renders as garbage in non-terminal outputs
- No correlation IDs to trace signal → order → fill chains
- No handler execution time logging
- No queue depth monitoring
- Error messages silently swallowed in `except Exception` blocks with no context logging

#### 30.4.6 CSV Data Provider Memory Usage

**File:** `data_provider/connectors/csv_data_connector.py:252-325`

- Eagerly loads full M1 datasets for all symbols into Polars DataFrames
- Left join on `full_idx` (union of all symbols' timestamps) causes cartesian expansion
- `streaming=True` on line 252 is immediately defeated by `.collect()`
- Multiple `.rechunk()` calls force data consolidation
- A 5-year daily backtest with 50 symbols can easily consume 3-5 GB of RAM; minute-level multiplies by ~1440x

#### 30.4.7 Interface Compliance Gaps

| Issue | Description |
|---|---|
| `IIndicator` declares `compute(self, data: pd.DataFrame) -> pd.Series` | No indicator implements this — all use static `@njit` functions with numpy arrays |
| `IRiskEngine.assess_order` signature | Interface takes `(suggested_order, modules)`, service takes different params |
| `IExecutionEngine` private methods | Interface declares `_get_*` methods — private by convention, but called by external `Portfolio` class |
| Live ExecutionEngine `_update_values_and_check_executions_and_fills` | Implemented as `pass` (no-op) — SL/TP not checked in live mode |

#### 30.4.8 Type Safety Gaps

| Location | Issue |
|---|---|
| `csv_data_connector.py:67` | `self.symbol_data_generator: dict[str, any] = {}` — lowercase `any` is not a type hint |
| `parameter_store.py:17` | `value: Any` with no validation |
| Strategy decorator methods | Missing return type annotations on all decorator methods |
| `portfolio_handler.py:125` | `zip_file.setpassword(password.encode())` — crashes if `password=None` |

#### 30.4.9 Dead Code and Deprecated Methods

| File | Description |
|---|---|
| `csv_data_connector.py` | 5 deprecated methods without deprecation markers (`get_latest_tick_old`, `get_latest_bar_old_lookahead_bias`, etc.) |
| `mt5_live_data_connector.py` | 2 deprecated pandas-based methods without deprecation markers |
| `csv_data_connector.py` | `_base_minutes` and `_base_idx_map_int` caches initialized but never used |
| `mt5_live_data_connector.py` | `last_bar_datetime` and `futures_tuple` initialized but never used |
| `sizing_engine_service.py` | `events_queue` stored but never used |
| `sizing_engine_service.py:22,25` | `SuggestedOrder` imported twice |
| `strategy.py` | Commented-out imports: `optimization`, `hyperopt`, `uuid` |
| `strategy.py:394,500` | FIXME comments acknowledging architecture debt |

#### 30.4.10 CI/CD Risks

**File:** `.github/workflows/release.yml`

| Issue | Impact |
|---|---|
| Poetry version not pinned (`version: latest`) | Breaking Poetry update can fail all builds |
| No `poetry.lock` committed | Builds are non-deterministic — different runs can install different versions |
| No test step in pipeline | Code is published to PyPI without any automated verification |
| `quantdle` missing from `pyproject.toml` | Published package has broken import for `QuantdleDataUpdater` |

---

### 30.5 LOW Issues

#### 30.5.1 Minor Code Quality

| Issue | File:Line |
|---|---|
| Duplicate imports | `sizing_engine_service.py:22,25`, `indicator_interface.py` |
| No hook deregistration mechanism | `hook_service.py` — callbacks accumulate across Strategy instances |
| `BacktestResults` assumes DataFrame schema | `backtest_results.py:35` — `backtest_pnl.EQUITY.pct_change()` with no column validation |
| Walk-forward typos | `WalkForward.retrainting_timestamps`, `transform_timstamps` |
| Signal engine typo | `signal_engine_service.py` — debug log says `"SINAL"` instead of `"SIGNAL"` |
| Risk engine typo | `risk_engine_service.py` — `set_custom_asses_order` (should be `assess`) |
| Mixed language comments | `live_mt5_broker.py` — Spanish comments mixed with English |
| Path string concatenation | Walk-forward uses `path + "/file"` instead of `os.path.join()` |

---

### 30.6 Dependencies Analysis

#### 30.6.1 Declared Dependencies

| Package | Constraint (`pyproject.toml`) | Role | Notes |
|---|---|---|---|
| python-dotenv | ^1.1.1 | Environment variables | OK |
| pydantic | ^2.12.3 | Data validation | OK — core to all configs and events |
| numpy | ^2.0.0 | Numerical computation | OK — required by indicators |
| polars | ^1.35.0 | DataFrame operations | OK — core to data provider |
| pandas | ^2.2.3 | Timestamps, legacy exports | Could be optional — only needed for ScheduleService timestamps and BacktestResults |
| numba | ^0.62.1 | JIT compilation | OK — required for indicator performance |
| matplotlib | ^3.7.0 | Plotting | Could be optional — only needed for `BacktestResults.plot()` |
| scipy | ^1.10.0 | Scientific computing | **Usage unclear** — no direct usage found in documented modules |
| scikit-learn | ^1.3.0 | Machine learning | **Usage unclear** — no direct usage found in documented modules |
| PyYAML | ^6.0 | YAML config loading | OK — required by SharedData |

#### 30.6.2 Missing Dependencies

| Package | Used by | Listed in `requirements.txt`? | Listed in `pyproject.toml`? |
|---|---|---|---|
| `quantdle` | `QuantdleDataUpdater` | Yes (`>=1.0.0,<2.0.0`) | **No** |
| `MetaTrader5` | Live trading | Commented out | No (optional, Windows-only) |

#### 30.6.3 Dependency Recommendations

| Recommendation | Rationale |
|---|---|
| Add `quantdle` to `pyproject.toml` | Prevents `ImportError` for PyPI users |
| Declare `MetaTrader5` as optional extra: `pip install pyeventbt[mt5]` | Clean optional dependency management |
| Move `matplotlib` to optional extra: `pip install pyeventbt[plot]` | Not needed for headless/server backtest |
| Move `scipy` and `scikit-learn` to optional extra or remove | No clear usage found in core modules |
| Commit `poetry.lock` to repository | Ensures deterministic builds |
| Pin Poetry version in CI to specific release | Prevents non-deterministic build failures |

---

### 30.7 Recommended Fix Priority

| Priority | Issue | Effort | Impact |
|---|---|---|---|
| **P0** | Add event validation + exception handling in dispatch loop | Small | Prevents crashes |
| **P0** | Add `quantdle` to `pyproject.toml` | Trivial | Prevents install failures |
| **P0** | Fix bare `except:` → `except Exception:` | Trivial | Allows Ctrl+C in live trading |
| **P1** | Fix decorator return values (`return fn`) | Trivial | Fixes Python convention |
| **P1** | Bound memory growth (queue maxsize, prune `executed_deals`, cap historical dicts) | Medium | Prevents OOM on long backtests |
| **P1** | Fix float precision in price encoding (use `round()` not `floor()`) | Small | Fixes silent price errors |
| **P1** | Add end-of-backtest timeout guard | Small | Prevents infinite hang |
| **P2** | Add reconnection logic for live trading | Medium | Production safety |
| **P2** | Fix version inconsistencies across docs | Trivial | User trust |
| **P2** | Replace `print()` with logger, remove ANSI codes | Medium | Production observability |
| **P2** | Fix BollingerBands variance (N → N-1) | Trivial | Indicator correctness |
| **P2** | Commit `poetry.lock`, pin CI Poetry version | Small | Build reproducibility |
| **P3** | Fix mutable default arguments | Small | Correctness |
| **P3** | Refactor SharedData into injectable dependency | Large | Testability, thread safety |
| **P3** | Move optional deps to extras | Small | Cleaner install |
| **P3** | Add automated tests | Large | Foundation for all future changes |
| **P3** | Clean up dead code and deprecated methods | Medium | Maintainability |

---
