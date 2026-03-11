# pyeventbt.strategy.core

**Package**: `pyeventbt.strategy.core`

**Purpose**: Contains core data models, enumerations, and type definitions used by the `Strategy` facade and the broader framework -- including the `Modules` dependency-injection container, timeframe definitions, logging levels, walk-forward result types, account currency options, and error definitions.

**Tags**: `#core` `#data-models` `#enums` `#configuration` `#types`

---

## Modules

| Module | File | Description |
|---|---|---|
| `pyeventbt.strategy.core.modules` | `modules.py` | Pydantic `Modules` model that bundles framework components for injection into user callbacks. |
| `pyeventbt.strategy.core.strategy_timeframes` | `strategy_timeframes.py` | `StrategyTimeframes` enum defining all supported bar timeframes with `timedelta` conversion. |
| `pyeventbt.strategy.core.verbose_level` | `verbose_level.py` | `VerboseLevel` class mapping standard Python logging level integers to named constants. |
| `pyeventbt.strategy.core.walk_forward` | `walk_forward.py` | `WalkforwardType` enum and `WalkForwardResults` Pydantic model for walk-forward optimization results. |
| `pyeventbt.strategy.core.errors` | `errors.py` | Placeholder for strategy-specific error/exception classes. Currently empty. |
| `pyeventbt.strategy.core.account_currencies` | `account_currencies.py` | `AccountCurrencies` enum for supported account denomination currencies. |
| `pyeventbt.strategy.core.__init__` | `__init__.py` | Package initializer. Empty beyond the license header. |

---

## Internal Architecture

This sub-package is purely declarative -- it defines types, enums, and data containers. It contains no business logic or service classes. All modules are imported by `pyeventbt.strategy.strategy` and some are re-exported from the top-level `pyeventbt` package.

```
strategy.core
  |
  +-- modules.py          (Modules: Pydantic DI container)
  +-- strategy_timeframes.py (StrategyTimeframes enum)
  +-- verbose_level.py     (VerboseLevel: logging constants)
  +-- walk_forward.py      (WalkForwardResults: optimization output)
  +-- account_currencies.py (AccountCurrencies enum)
  +-- errors.py            (empty placeholder)
```

---

## Cross-Package Dependencies

| Module | External Dependencies |
|---|---|
| `modules.py` | `pydantic.BaseModel`, `pyeventbt.data_provider.core.interfaces.IDataProvider`, `pyeventbt.execution_engine.core.interfaces.IExecutionEngine`, `pyeventbt.portfolio.core.interfaces.IPortfolio`, `pyeventbt.trading_context.TypeContext` |
| `strategy_timeframes.py` | `enum.Enum`, `datetime.timedelta` (stdlib only) |
| `verbose_level.py` | None (stdlib only) |
| `walk_forward.py` | `pydantic.BaseModel`, `pydantic.ConfigDict`, `pydantic.field_validator`, `pandas`, `pyeventbt.backtest.core.backtest_results.BacktestResults` |
| `account_currencies.py` | `enum.Enum` (stdlib only) |
| `errors.py` | None |

---

## Gaps & Issues

1. **`errors.py` is empty**: No custom exceptions are defined, despite the file's existence suggesting they were planned.
2. **`__init__.py` exports nothing**: Users must use full dotted paths to import individual types.
3. **`VerboseLevel` is not an Enum**: It is a plain `int` subclass with class attributes. This means it lacks standard enum features like iteration, membership testing, and `.name`/`.value` properties.
4. **`WalkForwardResults` has a typo**: The field is named `retrainting_timestamps` (should be `retraining_timestamps`). This typo propagates to the CSV export/import methods.
5. **`StrategyTimeframes` month approximations**: `ONE_MONTH` = 30 days, `SIX_MONTH` = 180 days, `ONE_YEAR` = 365 days. These are approximations that may cause drift in long-running backtests.
