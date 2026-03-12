# Package: pyeventbt.utils

**Package**: `pyeventbt.utils`
**Purpose**: Cross-cutting utility functions and helpers used throughout the PyEventBT framework, including order type mapping, currency conversion, date formatting, timeframe detection, and colored logging.
**Tags**: `#utils` `#helpers` `#logging` `#currency` `#mt5`

---

## Modules

| Module | Description |
|---|---|
| `__init__.py` | Empty package marker |
| `utils.py` | `Utils` class with static methods, `TerminalColors`, `LoggerColorFormatter`, `check_platform_compatibility()`, `print_percentage_bar()` |

---

## Internal Architecture

```
utils/
  __init__.py
  utils.py       <-- All utility functions in a single file
```

The package contains a single implementation file with a mix of:
- A `Utils` class with static utility methods (order type conversion, currency conversion, timeframe bar detection, date formatting, futures contract naming)
- Standalone functions (`check_platform_compatibility`, `colorize`, `print_percentage_bar`)
- A `TerminalColors` class for ANSI color codes
- A `LoggerColorFormatter` for color-coded log output

---

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pandas` | Timestamp arithmetic, Timedelta |
| `datetime`, `zoneinfo` | Date formatting, timezone handling |
| `decimal.Decimal` | Currency conversion precision |
| `platform` | OS detection for MT5 compatibility check |
| `logging` | Logger and custom formatter |
| `functools.lru_cache` | Caching for order type lookups |
| `pyeventbt.data_provider.core.interfaces.data_provider_interface.IDataProvider` | Type hint for currency conversion methods |

Consumed by:
- Execution engine modules (order type conversion)
- Portfolio/sizing modules (currency conversion)
- Trading director and other modules (logging, date formatting)
- Live trading setup (platform compatibility check)

---

## Gaps & Issues

1. **Single-file design**: All utilities are in one file rather than being organized by concern (e.g., separate modules for currency, order types, logging).
2. **Spanish comments**: Several code comments are in Spanish (e.g., "Buscamos el simbolo que relaciona nuestra divisa origen"), which reduces readability for non-Spanish-speaking contributors.
3. **Hardcoded FX symbol list**: Currency conversion methods contain a hardcoded tuple of FX pairs. Adding new currency pairs requires modifying the source code.
4. **`TerminalColors` is not an Enum**: Despite the CLAUDE.md referring to it as an enum, it is a plain class with string class attributes. This means it cannot be iterated or validated as an enum.
5. **Name shadowing in `LoggerColorFormatter`**: The class defines both a `format` class attribute (string template) and a `format` method. The method shadows the attribute, which works at runtime because the method accesses `self.FORMATS` (which references the class-level string via closure), but it is confusing.
