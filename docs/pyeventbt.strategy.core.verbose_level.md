# pyeventbt.strategy.core.verbose_level

**File**: `pyeventbt/strategy/core/verbose_level.py`

**Module**: `pyeventbt.strategy.core.verbose_level`

**Purpose**: Defines the `VerboseLevel` class, which provides named constants for Python standard logging levels. Used as the `logging_level` parameter type for `Strategy.__init__()`.

**Tags**: `#logging` `#configuration` `#constants`

---

## Dependencies

None (no imports).

---

## Classes

### `VerboseLevel(int)`

A plain subclass of `int` with class-level attributes mapping human-readable names to standard Python logging level integers. This is **not** an `Enum`.

#### Class Attributes

| Attribute | Value | Equivalent `logging` constant |
|---|---|---|
| `CRITICAL` | `50` | `logging.CRITICAL` |
| `FATAL` | `50` | `logging.FATAL` |
| `ERROR` | `40` | `logging.ERROR` |
| `WARNING` | `30` | `logging.WARNING` |
| `WARN` | `30` | `logging.WARN` |
| `INFO` | `20` | `logging.INFO` |
| `DEBUG` | `10` | `logging.DEBUG` |
| `NOTSET` | `0` | `logging.NOTSET` |

#### Usage

```python
strategy = Strategy(logging_level=VerboseLevel.DEBUG)
```

Since `VerboseLevel` extends `int`, the class attributes are plain `int` values and can be passed directly to `logger.setLevel()`.

---

## Data Flow

- **Input**: User selects a verbosity level when constructing `Strategy`.
- **Output**: Passed to `logging.Logger.setLevel()` inside `Strategy.__init__()`.

---

## Gaps & Issues

1. **Not an Enum**: Unlike most configuration types in this codebase, `VerboseLevel` is a plain `int` subclass. It cannot be iterated, does not support `.name`/`.value` properties, and does not prevent creation of arbitrary `VerboseLevel(42)` instances.
2. **Duplicates standard library**: The `logging` module already provides `logging.DEBUG`, `logging.INFO`, etc. with identical values. This class adds no functionality beyond namespacing.
3. **`WARN` and `FATAL` aliases**: These mirror deprecated `logging.WARN` and `logging.FATAL` aliases. Python documentation recommends using `WARNING` and `CRITICAL` instead.

---

## Requirements Derived

1. **REQ-LOG-001**: The system shall allow users to configure logging verbosity at Strategy construction time.
2. **REQ-LOG-002**: Supported logging levels shall match Python standard library logging levels (CRITICAL, ERROR, WARNING, INFO, DEBUG, NOTSET).
